---
id: "01M6K81NRBF6X8ZWVXVT17VNRE"
title_en: "The Memory Model: RVWMO, What FENCE Means, and Ztso"
summary_en: "Part 9 of the RISC-V series. Covers preserved program order in the RVWMO memory model, fence pred/succ, aq/rl, and Ztso. I enumerated 7 litmus tests under SC, TSO, and RVWMO to count whether MP, SB, and LB are allowed, and compared GCC's C11 atomic mappings under RVWMO and Ztso."
---
:::note
**Key points in three lines**
- RVWMO is a weak memory model: "accesses look in order within your own hart, but other harts may see some of them reordered." The orders that must be kept (preserved program order) are defined by 13 rules, and software requests the rest with `fence` or `aq`/`rl`.
- Enumerating all 7 litmus tests under SC, TSO, and RVWMO, the weak outcomes of MP, SB, and LB were all allowed under RVWMO, while TSO allowed only SB. `fence rw,rw` blocks SB but `fence.tso` does not. The enumerator reproduced the example result from the spec's appendix (a0 ∈ {2, 4, 5}) exactly.
- Running each test 1 million times on QEMU with 2 harts never produced a weak outcome. GCC emitted a `memory_order_seq_cst` store as `fence rw,w; sw; fence rw,rw`, and turning on Ztso removed all the acquire/release fences.
:::

While looking at atomic instructions in Part 8, I said several times that "a hart's accesses may appear reordered to other harts." Part 9 covers exactly what that means. A memory model (memory consistency model) is the set of rules that decides "which values a load may return." The question looks obvious with one hart, but becomes anything but obvious when there are several harts, each with its own store buffer and cache. This part summarizes the ISA-side rules (RVWMO, `fence`, Ztso) and leaves what a core implementation must honor as a checklist.

The setup is a Mac M1 Max, QEMU 11.1.3, `riscv64-elf-gcc` 16.2, and Python 3 (uv).

## 1. Why We Need a Memory Model

With only one hart, the rule is simple. A load returns the value most recently written to the same address in program order. No matter how much the core reorders instructions internally, it only has to look sequential to that hart itself.

With two harts, problems appear. The most common cause is the **store buffer**. A store takes time to reach the cache, so most cores put stores in a buffer and move on to the next instruction. A following load goes to memory without waiting for the buffer. The hart itself gets the value straight from the buffer (forwarding), so it sees no problem. But to other harts, it looks as if "the load happened before the store."

Let's trace it by hand. x and y start at 0.

```text
hart 0          hart 1
sw x ← 1        sw y ← 1
lw r1 ← y       lw r2 ← x
```

There are 4!/(2!·2!) = 6 ways to interleave these two harts' instructions. Following all 6, (r1, r2) comes out as (0, 1) once, (1, 0) once, and (1, 1) four times. However they are interleaved, at least one store comes before both loads, so (0, 0) is impossible. A model that says "all harts' accesses are equivalent to some single interleaving" is sequential consistency (SC).

![While the stores of hart 0 and hart 1 sit in their store buffers, both loads read 0 from memory first, producing r1 = r2 = 0, which is impossible under sequential consistency|full](/img/01M6K81NRBF6X8ZWVXVT17VNRE/3476c5b4a5bb.webp)

With store buffers, (0, 0) can happen. Both loads just need to read 0 from memory while both stores sit in their buffers. This test is called SB (store buffering). A single hardware optimization broke SC. A memory model decides at the ISA level which of these outcomes are allowed and which are forbidden. The wider the allowed range (a weaker model), the freer the hardware, and the more software has to request ordering explicitly where it needs it.

## 2. Litmus Tests: MP, SB, LB

Memory models are easiest to discuss with litmus tests. Take a tiny program with two harts and two or three instructions each, and ask "can this outcome happen?" Here are the three classic ones.

| Name | Hart 0 | Hart 1 | Weak outcome |
|---|---|---|---|
| MP | `x=1; y=1` | `r1=y; r2=x` | r1=1, r2=0 |
| SB | `x=1; r1=y` | `y=1; r2=x` | r1=0, r2=0 |
| LB | `r1=x; y=1` | `r2=y; x=1` | r1=1, r2=1 |

- **MP (message passing):** hart 0 writes data x and raises flag y. If hart 1 sees the flag (r1=1) and reads the data but gets the old value (r2=0), message passing is broken. This happens if the two stores are reordered or the two loads are reordered.
- **SB:** the example from section 1. It happens when a load after a store becomes visible first.
- **LB (load buffering):** it happens when a store after a load becomes visible first. It looks as if each hart read the other's "future" store.

How the three models treat these outcomes is the heart of this part. SC forbids all three, TSO (total store order), used by x86 and SPARC, allows only SB, and RVWMO allows all three. The hands-on in section 7 enumerates them to confirm.

## 3. RVWMO's Rules, Intuitively

The definition of RVWMO (RISC-V Weak Memory Ordering) looks like this. There must exist a **global memory order** that lines up all harts' memory accesses in a single sequence, and that order must respect part of each hart's program order, namely the **preserved program order (PPO)**. It must also satisfy three axioms.

The 13 PPO rules fall into four groups. When a precedes b in program order and both are regular memory accesses, a must also precede b in the global order if any of the following holds.

| Group | Rule | Intuition |
|---|---|---|
| Same address (1–3) | b is a store to an address overlapping a, etc. | Don't mix up accesses to the same location |
| Explicit synchronization (4–8) | A `fence` in between, aq on a, rl on b, an LR/SC pair | Order requested by software |
| Syntactic dependencies (9–11) | b's address or data depends on a's result, a store control-dependent on a | If values are linked, so is the order |
| Pipeline dependencies (12–13) | Indirect dependencies through intermediate instructions | What a typical pipeline honors naturally |

Orders not listed here, such as **store→load, load→load, store→store, and load→store to different addresses**, may all appear reordered. That is why MP, SB, and LB are all allowed.

There are three axioms.

- **Load Value Axiom:** a load returns the value of whichever store is latest in the global order among "stores that precede it in the global order" and "stores that precede it in program order." The second term is what allows store buffer forwarding. A hart can see its own store before anyone else does.
- **Atomicity Axiom:** no store from another hart can come between a matching LR/SC pair. This is the precise definition of Part 8's rule that "`sc` fails if another hart stored."
- **Progress Axiom:** no access can be postponed forever. Stores eventually become visible to others.

RVWMO is weaker than TSO, but unlike POWER, it does not allow a store to become visible to only some harts first (with the exception of forwarding within the storing hart). The spec's appendix calls this multi-copy atomicity (more precisely, other-multi-copy atomicity) and states that both RVWMO and RVTSO have this property. That is what makes the picture of "a global order lined up in a single sequence" hold.

## 4. FENCE: pred and succ

`fence pred, succ` guarantees that "pred-type accesses before the fence become visible before succ-type accesses after the fence." The types are chosen with four bits: R (memory read), W (memory write), I (device input), and O (device output). I checked the encodings with GNU as.

```text
fence rw,rw  = 0x0330000f  pred 0011 succ 0011
fence r,rw   = 0x0230000f  pred 0010 succ 0011
fence rw,w   = 0x0310000f  pred 0011 succ 0001
fence.tso    = 0x8330000f  fm 1000, rw,rw
fence        = 0x0ff0000f  iorw,iorw
# [31:28] fm, [27:24] PI PO PR PW,
# [23:20] SI SO SR SW, opcode MISC-MEM
```

Writing just `fence` makes the assembler fill in `iorw,iorw`. The `fm` (fence mode) field currently defines only 0000 (normal) and 1000 (TSO). Here is when to use what.

| Fence | Order it enforces | Where it is used |
|---|---|---|
| `fence rw,rw` | Everything before and after | SB, C11 `seq_cst` fence |
| `fence r,rw` | Earlier reads → everything after | After an acquire load |
| `fence rw,w` | Everything before → later writes | Before a release store |
| `fence w,w` | Write → write | MP writer side (Linux `smp_wmb`) |
| `fence r,r` | Read → read | MP reader side (Linux `smp_rmb`) |
| `fence.tso` | Read → everything, write → write | C11 `acq_rel` fence |
| `fence iorw,iorw` | Everything, including devices | Drivers (Linux `mb()`) |

`fence.tso` gives in one instruction the ordering x86 keeps natively. Only write→read ordering is missing. That is why `fence.tso` cannot block SB. The spec states that implementing `fence.tso` as `fence rw,rw` is also correct (since it is stronger). By the same principle, a simple implementation may ignore pred and succ and always execute a fence that blocks all accesses.

The spec reserves two things for the future. The `rs1` and `rd` fields are reserved: hardware ignores them and software sets them to 0. Undefined `fm` and pred/succ combinations are treated by hardware as a normal fence with `fm`=0000. A `fence` with empty pred or succ is in the HINT space, and `pause` (Zihintpause) uses this slot.

## 5. How aq/rl Relates to FENCE

The `aq` and `rl` bits from Part 8 are PPO rules 5–7. An access with aq becomes visible before all later accesses, and an access with rl becomes visible after all earlier accesses. Accesses that have both bits keep their order with respect to each other (RCsc). Compared with fences, an acquire can use `lw` followed by `fence r,rw` instead of `amoswap.w.aq`, and a release can use `fence rw,w` followed by `sw` instead of `amoswap.w.rl`.

The difference is scope. `fence r,rw` puts **every** load before the fence ahead of all later accesses. `.aq` puts **only that one instruction** ahead. It does not force unneeded ordering, so the hardware can overlap more accesses.

There is a limitation too. RISC-V currently has no regular loads or stores with aq/rl. aq and rl exist only on AMOs and LR/SC. So when a regular load needs ordering, as with C11's `atomic_load(acquire)`, a fence is used. The GCC output in the next section shows this.

## 6. Ztso: Strong Ordering Like x86

Ztso adds no instructions at all. Instead, it changes the memory model to RVTSO. The spec defines it as RVWMO plus three lines.

- Every load behaves like an acquire (acquire-RCpc).
- Every store behaves like a release (release-RCpc).
- Every AMO behaves as if it has both acquire and release (RCsc).

As a result, load→everything and store→store orders are kept automatically, and only store→load can be reordered. This is the same as x86, which has only a store buffer and is otherwise in order. The spec explains that Ztso is for porting x86 and SPARC code, and for letting implementations that already behave like TSO tell software so.

Direction matters. Code for RVWMO runs as-is on a Ztso core (the fences remain but do no harm). Conversely, a binary that assumes Ztso and drops fences runs incorrectly on an RVWMO core. That is why the spec asks Ztso-only binaries to carry a marker so platforms without Ztso can refuse to run them.

## 7. Hands-On: Enumerator, QEMU, GCC

**(1) Enumerating allowed outcomes.** I built a small enumerator in Python. For each hart, it generates every reordering of memory accesses that keeps only the orders PPO requires, then runs every interleaving between the harts on a single memory. If a load has an earlier store from its own hart to the same address that has not executed yet, it takes that value (forwarding from the Load Value Axiom). The difference between models lives in a single "order that must be kept" function.

```python
def preserved(model, prog, i, j):   # i < j
    a, b = prog[i], prog[j]
    if model == 'SC': return True
    if a[1] == b[1] and (b[0] == 'st'
            or a[0] == b[0] == 'ld'):
        return True                 # same address
    if any(f[0] in ('fence', 'tso')
           and fence_orders(f, a, b)
           for f in prog[i + 1:j]):
        return True                 # fence
    if b[0] == 'st' and len(b) > 3 and b[3] == a[2]:
        return True                 # data dependency
    if model == 'TSO':
        return not (a[0] == 'st' and b[0] == 'ld')
    return False                    # RVWMO
```

This is not a full implementation of RVWMO. It is simplified to give the same answers as the spec appendix's conclusions for these tests, which have two harts and no mixed same-address accesses. As a correctness check, I fed in the example test from spec appendix A.2 (hart 0: writes 1 and 2 to x, reads it, then writes 3; hart 1: writes 4 and 5 to x). The spec says the value hart 0 reads is one of 2, 4, or 5, and 1 and 3 are impossible, and the enumerator's answer was exactly {2, 4, 5}.

![Enumerating all 7 litmus tests, RVWMO allows the weak outcome in MP, SB, SB+fence.tso, and LB, TSO allows it only in SB and SB+fence.tso, and adding a fence or data dependency makes all three models forbid it|xl](/img/01M6K81NRBF6X8ZWVXVT17VNRE/35310e469700.webp)

| Test | SC | TSO | RVWMO |
|---|---|---|---|
| MP | Forbidden | Forbidden | Allowed |
| MP + `fence w,w` / `fence r,r` | Forbidden | Forbidden | Forbidden |
| SB | Forbidden | Allowed | Allowed |
| SB + `fence rw,rw` | Forbidden | Forbidden | Forbidden |
| SB + `fence.tso` | Forbidden | Allowed | Allowed |
| LB | Forbidden | Forbidden | Allowed |
| LB + data dependency | Forbidden | Forbidden | Forbidden |

Enumerating MP under RVWMO gives 2 orders per hart for 4 combinations, with 6 interleavings each, for 24 executions, and the four outcomes came out evenly, 6 each. Adding fences reduced it to the same 6 executions as SC, with 3 outcomes. The table confirms section 4's explanation that `fence.tso` cannot block SB. Adding a data dependency to LB (`r1=x; y=r1`) makes it forbidden, because PPO rule 10 requires "if values are linked, so is the order." Without the dependency, the hardware could send the store out first without waiting for the load result.

**(2) Running it on QEMU.** I copied the multi-hart startup code from Part 8 and ran MP, SB, and LB 1 million times each with 2 harts. Each round reset x=y=0, synchronized the two harts with an `amoadd.w` barrier, and had each hart spin 0–511 times using a fixed-seed random number to scatter the start times (51 seconds).

| Test | Weak outcomes in 1 million runs | Evidence the two harts overlapped |
|---|---|---|
| MP | 0 | |
| SB | 0 | (1,1) 23 times |
| SB + `fence rw,rw` | 0 | (1,1) 15 times |
| SB + `fence.tso` | 0 | (1,1) 30 times |
| LB | 0 | |

A weak outcome never appeared. That does not mean QEMU is SC. In SB, (1,1) requires both stores to become visible before both loads, so it is evidence that the two harts really ran the test code at the same moment. That happened only 23 times in 1 million. Most of the time, one hart finished entirely before the other ran, producing (0,1) or (1,0). With overlap this rare, the absence of weak outcomes supports no conclusion. Seeing weak outcomes on a real chip requires the careful synchronization and huge numbers of repetitions that tools like litmus7 use. This is why design verification checks the memory model with enumerators or formal models (the herd and Alloy models in Appendix B) rather than random execution.

**(3) The instructions C11 atomics become.** I compiled functions for each `memory_order` with `-march=rv64gc` and `rv64gc_ztso` (address calculation and sign extension omitted).

| C11 operation | RVWMO | Ztso |
|---|---|---|
| load relaxed | `lw` | `lw` |
| load acquire | `lw; fence r,rw` | `lw` |
| load seq_cst | `fence rw,rw; lw; fence r,rw` | `fence rw,rw; lw` |
| store release | `fence rw,w; sw` | `sw` |
| store seq_cst | `fence rw,w; sw; fence rw,rw` | `sw; fence rw,rw` |
| fetch_add acquire | `amoadd.w.aq` | `amoadd.w` |
| fetch_add seq_cst | `amoadd.w.aqrl` | `amoadd.w` |
| CAS seq_cst | `lr.w.aqrl` … `sc.w.rl` | Same |
| fence acq_rel | `fence.tso` | (none) |
| fence seq_cst | `fence rw,rw` | `fence rw,rw` |

Most of this matches the C/C++ mapping table in spec appendix A.5. There is one difference. The mapping table lists a `seq_cst` store as `fence rw,w; sw`, but GCC 16.2 appended one more `fence rw,rw`. In the same appendix, the spec notes that "if a seq_cst store from this table is followed by a seq_cst load that uses a possible future aq load, the order can be reversed, so either add one more fence on the store side or use `amoswap.rl`." It appears GCC chose that strengthened mapping (a guess). Under Ztso, all acquire/release fences disappear, and the aq/rl bits on AMOs are dropped too. Only the `fence rw,rw` that blocks SB remains, because store→load can still be reordered under TSO.

## 8. Design Checklist: The Memory Model

| Item | Rule |
|---|---|
| Base model | RVWMO. Store→load, load→load, store→store, and load→store to different addresses may be reordered |
| Store buffer forwarding | Allowed (Load Value Axiom). But values written by AMOs and SC must not be forwarded early to later loads (rule 3) |
| Same address | A store comes after earlier accesses to the same address. Two same-address loads that read different stores' values keep their order (rule 2) |
| Dependencies | Address and data dependencies, and control-dependent stores, keep their order. A store must not be sent out early through speculative execution |
| `fence` implementation | Simplest: stall later accesses until all earlier accesses finish and the store buffer drains. pred/succ may be ignored |
| W in pred, R in succ | In an in-order core with a FIFO store buffer, only this case needs to drain the buffer |
| `fence.tso` | Handling it as `fence rw,rw` is correct |
| Reserved combinations | Undefined `fm`/pred/succ become a normal fence. Ignore `rs1` and `rd` |
| pred or succ = 0 | HINT (`pause`, etc.). No fence behavior needed |
| I/O | The I and O bits order device accesses separately. PMAs determine which addresses are I/O |
| aq/rl | Only on AMOs and LR/SC. aq: delay the start of later accesses; rl: wait for earlier accesses to complete |
| Ztso implementation | Must keep load order and store order. A reordering core needs violation detection and replay |

A simple core that executes in order with a FIFO store buffer effectively behaves like TSO. Such a core handles most `fence`s almost for free and only needs to drain the store buffer for `fence w,r`-style fences. Out-of-order cores are a different story. They need machinery that executes loads speculatively early, then checks later whether a same-address rule or fence was violated and replays. This machinery is not unique to RISC-V, so this series looks at it only as far as needed when covering aq/rl and `fence` handling in the A extension implementation (Part 20).

## Summary

- A memory model decides which values a load may return. A store buffer alone produces results impossible under sequential consistency, such as SB's r1 = r2 = 0.
- RVWMO is defined by 13 PPO rules (same address, explicit synchronization, syntactic dependencies, pipeline dependencies) and 3 axioms. Enumeration showed that the weak outcomes of MP, SB, and LB are all allowed, and adding a fence or data dependency forbids them.
- `fence pred,succ` requests ordering with combinations of R, W, I, and O. `fence.tso` blocks everything except write→read, so it cannot block SB. aq/rl is a narrower ordering that applies to a single instruction.
- Ztso changes only the model to TSO, with no new instructions. GCC dropped all acquire/release fences under Ztso, and added one more fence than the spec table to RVWMO `seq_cst` stores. A weak outcome never appeared in 1 million QEMU runs.

**Next post:** Part 10 moves on to the floating-point extensions F and D. Using QEMU, I check the 32 FP registers and `fcsr`, the five rounding modes and exception flags, NaN boxing used when putting a 32-bit value into a 64-bit register, and the rules for out-of-range integer conversions.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. 2.7 memory ordering instructions (FENCE, fm, FENCE.TSO), 2.9 HINTs, Chapter 17 RVWMO memory consistency model v2.0 (17.1.3 preserved program order, 17.1.4 axioms), Chapter 18 "Ztso" v1.0, Appendix A RVWMO explanatory material (A.2 litmus tests, A.5 code porting and mapping tables), Appendix B formal memory model. https://github.com/riscv/riscv-isa-manual
- RISC-V litmus test collection. https://github.com/litmus-tests/litmus-tests-riscv
- D. Sorin, M. Hill, D. Wood, *A Primer on Memory Consistency and Cache Coherence*, 2nd ed., 2020. Chapter 3 SC, Chapter 4 TSO, Chapter 5 relaxed models
- C. Trippel et al., "TriCheck: Memory Model Verification at the Trisection of Software, Hardware, and ISA," ASPLOS 2017 (the catalyst for revising the RISC-V memory model)
- GCC RISC-V atomic mappings (`-march=…_ztso`). https://gcc.gnu.org/onlinedocs/gcc/RISC-V-Options.html
