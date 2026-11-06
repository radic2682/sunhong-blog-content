---
id: "01M6K7Y4WY8WJZ2JNKBGVS02AM"
title_en: "Implementing the A Extension: Reservations, Where AMOs Run, aq/rl, FENCE"
summary_en: "Part 20 of the RISC-V series. I compared LR/SC reservation sets, AMO atomicity, and aq/rl and FENCE implementations with a two-hart model. If reservations aren't cleared on another hart's store, the counter drops to 69.6%, and QEMU let SC succeed in all 20 runs of an ABA test."
---
:::note
**The 3 key points of this post**
- LR/SC is built with one reservation register (valid bit + address) per hart. The key is a single rule: "clear it when another hart's store touches the reservation set." Without this rule, a counter that two harts each incremented 2,000 times ended up at an average of 2,782 instead of 4,000.
- An AMO must do its read-modify-write as one indivisible unit that no other hart can break into. Splitting it into two steps left only 76.8% on average. In an in-order core with no store buffer, aq, rl, and FENCE are kept almost automatically; with a store buffer, the buffer must be drained on rl and FENCE (without draining, a counter incremented 1,000 times under a spinlock averaged 779.5).
- On QEMU `-smp 2`, 400,000 counter increments were exact with both LR/SC and AMOs, but in an ABA test where another hart changes the value and changes it back between LR and SC, SC succeeded all 20 times. The specification requires failure. This is because QEMU TCG emulates SC with a value comparison instead of a reservation.
:::

In part 19 we added the M extension to the RV32I 5-stage pipelined core. This time it's the A extension (atomic instructions). The meaning of the instructions was covered in part 8, so here I answer implementation questions: how large to make the reservation set and when to clear it, when SC must fail, where to do an AMO's read-modify-write, and what it takes to honor aq/rl and FENCE. The verification tools are a small Python model of two harts sharing memory and a two-hart QEMU run.

## 1. What the A extension asks of hardware

The A extension is the sum of two sub-extensions (Unprivileged spec chapter 14). Zalrsc is `lr.w` and `sc.w`; Zaamo is the nine AMOs (`amoswap` `amoadd` `amoand` `amoor` `amoxor` `amomax` `amomaxu` `amomin` `amominu`). All use opcode AMO (`0101111`) and funct3 `010` (word), with funct5 giving the kind, inst[26] aq, and inst[25] rl.

| Instruction | What it does | Value written to rd |
|---|---|---|
| `lr.w rd, (rs1)` | Read and register a reservation | The value read |
| `sc.w rd, rs2, (rs1)` | Write if the reservation is valid | 0 on success, nonzero on failure |
| `amoXX.w rd, rs2, (rs1)` | Read, compute, and write atomically | The original memory value |

Only the SC failure code 1 is currently defined. The specification gives as the reason for this choice that the 0/1 selector already used by `slt`/`sltu` can be reused. Addresses must be 4-byte aligned; otherwise an address-misaligned exception or access fault is raised (emulating misaligned LR/SC is practically impossible).

From the core's point of view, three things are new: the reservation register, a path that does a read and a write back to back in the memory stage (AMO), and a mechanism that honors the ordering bits (aq/rl).

## 2. Reservation sets: what to remember and when to clear

LR registers a set of bytes containing the address it read, the reservation set. The specification leaves the size to the implementation: it can be arbitrarily large as long as it contains the word read. Common choices are a word (4 bytes) or a cache line (64 bytes, etc.) (my assumption). A hart holds only one reservation. A new LR overwrites the previous reservation, and SC clears its own reservation whether it succeeds or fails.

Whether SC may succeed is defined in section 14.2 of the specification, split into "must fail" and "may fail."

| Condition | SC result |
|---|---|
| Address is outside the reservation set of the most recent LR | Must fail |
| Another hart stored to the reservation set after LR | Must fail |
| Another device (DMA, etc.) wrote to LR's bytes after LR | Must fail |
| There is another SC between LR and SC | Must fail |
| Another device wrote to part of the reservation set other than LR's bytes | May fail or succeed |
| Any other reason (implementation constraints) | May occasionally fail, as long as the forward-progress guarantee in section 3 holds |

The second row is the key. The condition is not "did the value change" but "did another hart store." That is how LR/SC avoids the ABA problem that CAS (compare-and-swap) suffers from (if the value goes A→B→A, you can't tell it changed). This is also the first reason the specification gives for choosing LR/SC over CAS.

![If hart 1 changes x to 99 and back to 7 between LR and SC, SC fails in a reservation implementation and succeeds in a value-comparison implementation (QEMU TCG)|full](/img/01M6K7Y4WY8WJZ2JNKBGVS02AM/ac6f6437d866.webp)

Let's walk through it by hand. With x = 7, hart 0 reads 7 with `lr.w` and sets a reservation. Hart 1 writes 99 to x and then 7. The first store clears hart 0's reservation. Hart 0's `sc.w` fails because it has no reservation, and x stays 7. Hart 0 starts over from LR. The model's reservation implementation behaved exactly like this (rd = 1, x = 7). In contrast, an implementation that only checks "is the memory value the same as at LR time" on SC succeeds and writes x = 8.

### The simplest implementation

For a single-hart in-order core, the reservation takes just two registers.

```text
LR:  resv_valid <= 1; resv_addr <= addr[31:2]
SC:  ok = resv_valid && resv_addr == addr[31:2]
     if ok: write to memory
     rd <= ok ? 0 : 1;  resv_valid <= 0
```

With no other harts, "a store from another hart" never happens. What remains are bus masters such as DMA. If they write to the bytes LR read, the reservation must be cleared, so a comparator snoops the bus write address. In a multi-hart system, cache coherence messages play this role: the reservation is cleared when another core takes that cache line in an exclusive (write) state. This naturally makes the reservation size a cache line.

A large reservation causes false sharing. I had two harts each increment a different counter (addresses `0x1000` and `0x1004`) 2,000 times with LR/SC. The result was always correct, but the SC failure rate differed.

| Reservation size | Correct runs (out of 20) | SC failures per success |
|---|---|---|
| 4 bytes | 20 | 0.0 |
| 64 bytes | 20 | 0.444 |

Reservation size is a performance issue, not a correctness issue. The specification only says that "the platform should provide a means to determine the size and shape of the reservation set."

There are also things the specification does not require of hardware. If the previous thread's reservation survives a context switch, the new thread's SC may wrongly succeed. The specification says software should then execute one SC to any address to clear the reservation. Hardware may also clear the reservation on a trap or `mret` (a common choice, but not mandatory).

## 3. SC failure conditions and forward progress

If SC "may occasionally fail," two harts could endlessly clear each other's reservations so that neither ever succeeds, a livelock. Section 14.3 of the specification defines a **constrained LR/SC loop** and requires that for a hart in such a loop, something must eventually happen (someone's SC succeeds, another hart stores, the loop exits, or a trap occurs).

| Constrained loop condition | Reason |
|---|---|
| At most 16 instructions from LR to the retry branch, laid out contiguously | Within 64 bytes: eases instruction cache and TLB associativity constraints |
| Only base I instructions between LR and SC | Multiply, divide, and FP are forbidden to avoid OS emulation |
| No loads/stores, backward branches/jumps, JALR, FENCE, or SYSTEM | So other accesses don't evict the reservation line, and to bound the time |
| SC uses the same address and size as the last LR | |

LR/SC sequences outside these conditions are "unconstrained," and an implementation may fail them unconditionally. On the hardware side there is one thing to honor: **loads and LRs do not clear another hart's reservation.** The specification states that loads or LRs from other harts, or evictions from a shared cache, must not block LR/SC forward progress indefinitely.

I checked with the model what happens when this is violated. In a wrong implementation where "another hart's LR clears my reservation," two harts alternately executed one instruction at a time of a constrained loop (9 instructions) with 4 `addi`s between LR and SC. I inserted 0 to 9 nops before hart 1 to vary the starting phase in 10 ways.

| Implementation | SC successes over 20,000 steps (10 phases) |
|---|---|
| Only stores clear reservations (as specified) | 1,250 per phase |
| Another hart's LR also clears reservations | 0–2 per phase, 0 in 3 of the phases |

When the phase lines up so that each LR clears the other's reservation, the two harts endlessly repeat LR → (the other's LR clears my reservation) → SC fails → retry. In a cache-based implementation, if "LR brings the line in exclusive state and invalidates the other core's line," you get exactly this pattern (my assumption).

## 4. Where to execute AMOs

An AMO reads a memory value, operates on it with rs2, and writes the result to the same address. No store from another hart may slip in between these three. There are two places to do it.

**Inside the core (near the ALU).** In a 5-stage pipeline, an AMO uses the MEM stage twice. It reads in the first cycle, computes with a small AMO ALU (add, AND/OR/XOR, signed/unsigned max/min, swap), and writes in the next cycle. The original value read goes to rd. Atomicity comes from preventing other masters from writing to that address in between. With a single bus, it is enough to hold the bus and not release it (lock); with a cache, the read and write are completed while holding the line in exclusive state. The specification says a simple implementation may build AMOs internally using LR/SC, as long as completion is eventually guaranteed.

**On the memory or bus side.** The core only sends a request like "add this value at this address," and the memory controller or shared cache does the read-modify-write. AXI5 atomic transactions (AtomicStore, AtomicLoad, AtomicSwap, AtomicCompare) and TileLink Arithmetic/Logical messages serve this purpose. When many cores hammer the same counter, this has the advantage that the line doesn't bounce between cores, and if rd is x0 the original value doesn't even need to be returned (the specification mentions this optimization). Bus and cache design is outside the scope of this series, so I'll stop here.

Either way, the key is "one indivisible unit." When I split the AMO in the model into a read step and a write step so another hart could slip in between, the counter that should have been 4,000 averaged 3,072 (76.8%).

![Results of two harts incrementing a counter: correct LR/SC, AMO, and rl implementations kept 100%, while not clearing reservations, splitting AMOs, or ignoring rl kept only 65.6–78.0%|xl](/img/01M6K7Y4WY8WJZ2JNKBGVS02AM/edfb1bb36809.webp)

## 5. Honoring aq/rl and FENCE in an in-order core

aq (acquire) means "memory accesses after this atomic instruction must not be seen before it," and rl (release) means "this must not be seen before the preceding memory accesses" (section 14.1). With both set, it is sequentially consistent. FENCE uses its pred and succ bits to specify "which earlier accesses (R/W/I/O) must be seen before which later accesses."

The question is "can my core reorder accesses?"

- **An in-order core with no store buffer and no non-blocking loads:** every memory access completes one at a time in program order. A later access only starts after the earlier one completes, so the ordering that aq, rl, and FENCE require holds from the start. FENCE can be treated as a NOP.
- **A core with a store buffer:** if a store goes into the buffer and execution moves straight on to the next instruction, a later load reaches memory before the earlier store (store→load reordering). In this case, FENCE (W in pred, R in succ) must wait until the buffer drains.

I checked with the SB (store buffering) litmus test, in which two harts each write 1 to their own variable and read the other's. If both read 0, the order was reversed. 20,000 random schedules each.

| Configuration | Times both read 0 |
|---|---|
| No store buffer, no FENCE | 0 |
| Store buffer, no FENCE | 3,705 |
| Store buffer, FENCE as NOP | 1,768 |
| Store buffer, FENCE drains the buffer | 0 |

So "FENCE may be treated as a NOP" only when the core has none of the structures that reorder memory accesses (store buffers, non-blocking caches, buses that don't wait for write responses, etc.). The same goes for I/O. If the bus treats device writes as done before the response (posted writes), a FENCE that orders I/O must wait for the response.

aq/rl follow the same logic. In an in-order core, an AMO blocks later instructions until it completes, so aq holds automatically. rl requires that the AMO be seen only after all earlier stores are visible, so with a store buffer, the buffer must be drained before the AMO. The model's spinlock (lock with `amoswap.w.aq`, increment the counter with `lw`/`addi`/`sw`, unlock with `amoswap.w.rl`) demonstrates this.

![With a store buffer, if amoswap.w.rl bypasses the buffer, the lock release becomes visible before the counter store and the other hart reads the stale value|full](/img/01M6K7Y4WY8WJZ2JNKBGVS02AM/b95de72ad488.webp)

| Spinlock configuration (expected 1,000) | Average | Correct runs (out of 20) |
|---|---|---|
| No store buffer | 1,000 | 20 |
| Store buffer + drain on rl | 1,000 | 20 |
| Store buffer + rl ignored | 779.5 | 0 |

One more recommendation from the specification is worth remembering: setting only rl on LR or only aq on SC guarantees no stronger ordering than setting neither. An implementation can simply read the aq/rl bits as they are and handle them as "if rl, drain what's before; if aq, block what's after until completion."

## 6. Hands-on: the two-hart model and QEMU `-smp 2`

The model (`atomics.py`) executes a small RV-like instruction list per hart, one instruction at a time, and a scheduler picks which hart advances with a fixed-seed random choice. Each implementation choice is a single constructor argument.

```python
class Impl:
    def __init__(self, granule=4,
                 clear_on_store=True,  # another hart's store → clear reservation
                 clear_on_lr=False,    # bug: another hart's LR also clears
                 sc='resv',            # 'value': value comparison
                 amo_atomic=True,      # if False, read and write in two steps
                 sbuf=False,           # store buffer
                 drain_on_rl=True,     # drain buffer on rl
                 fence_drains=True):   # drain buffer on FENCE
```

In the counter test, two harts each increment the same address 2,000 times, repeated with 20 seeds.

| Implementation | Average final value (expected 4,000) | Correct (out of 20) |
|---|---|---|
| `lw`/`addi`/`sw` | 2,623 | 0 |
| LR/SC, correct reservation | 4,000 | 20 |
| LR/SC, reservation kept despite another hart's store | 2,782 | 0 |
| AMO in one step | 4,000 | 20 |
| AMO in two steps | 3,072 | 0 |

On the QEMU side, I copied the startup code to set up a separate stack per hart (`start_smp.S`) and ran two harts simultaneously on two host threads with `-smp 2 -accel tcg,thread=multi`. Each hart incremented the counter 200,000 times with three methods, followed by the ABA test from section 2. Results from 20 runs:

| Test (200,000 per hart) | QEMU result (20 runs) |
|---|---|
| `lw`/`addi`/`sw` | 200,264–201,649 (expected 400,000) |
| `lr.w`/`sc.w` loop | Always 400,000 |
| `amoadd.w` | Always 400,000 |
| SC result code in the ABA test | Always 0 (success) |

The counters reach the same conclusion as the model. The ABA test does not. After `lr.w`, hart 0 waited until hart 1 wrote 99 and then 7 to x and raised a flag, and then executed `sc.w`. Another hart's store was observed between LR and SC, so by the specification SC must fail. QEMU let it succeed all 20 times. Looking at the QEMU source (`gen_sc` in `trans_rva.c.inc`), it remembers the address and value read at LR and executes SC as a host compare-and-swap (`tcg_gen_atomic_cmpxchg_tl`) that "writes if the address matches and the value is still the same." The model's `sc='value'` implementation produced the same result (rd = 0, x = 8). In code like a counter, where the value only moves in one direction, the difference doesn't show up, but when using QEMU as a hardware verification reference, this has to be taken into account for LR/SC.

QEMU's SC failures averaged 177,229 per hart (against 200,000 successes). This value depends on host thread scheduling and has nothing to do with hardware performance. The machine is a Mac M1 Max, QEMU 11.1.3, and `riscv64-elf-gcc` 16.2 (`-march=rv32ima_zicsr`).

## 7. Design checklist: the A extension

| Item | Rule |
|---|---|
| Reservation register | One per hart. LR overwrites it; SC clears it on both success and failure |
| Reservation size | Anything at least the word read. Commonly a word or a cache line |
| Reservation clear | When another hart's store, AMO, or successful SC touches the set (mandatory) |
| What must not clear it | Other harts' loads/LRs and shared cache evictions must not block forward progress |
| Device writes | Clear if they write to the bytes LR read (mandatory). Optional for the rest of the set |
| SC failure code | 1 (other values are reserved) |
| Alignment | If not 4-byte aligned, address-misaligned exception or access fault |
| Failed SC | May be treated like a store for memory protection purposes (permission check) |
| AMO atomicity | Block other masters' writes between read, operate, and write (bus lock, exclusive line, or memory-side execution) |
| AMO rd | The original memory value. If rd = x0, the read response can be skipped |
| aq | Automatic in a blocking in-order core. Otherwise, later accesses wait until the AMO completes |
| rl | With a store buffer, drain it before executing |
| FENCE | NOP if there is no reordering structure at all. Otherwise, wait for pred-side accesses to complete |
| FENCE fields | Ignoring pred/succ and always ordering everything is also allowed. Reserved fm values may be treated as a normal FENCE, and FENCE.TSO as FENCE RW,RW (section 2.7) |
| Context switch | Clearing the reservation is software's job with SC. Clearing on a trap is optional |

## Summary

- A reservation is one register per hart. Without the rule "clear it when another hart's store touches the set," the counter reached 69.6% of the expected value. Growing the reservation to 64 bytes kept correctness intact but caused 0.444 SC failures per success due to false sharing.
- If loads or LRs clear another hart's reservation, a constrained LR/SC loop can fall into livelock. In the model, 3 of the 10 phases never succeeded once in 20,000 steps.
- An AMO must do its read-modify-write as one unit (splitting it gave 76.8%). Choose between two uses of the core's MEM stage plus a bus lock, or execution on the memory/bus side.
- In an in-order core without a store buffer, aq, rl, and FENCE hold automatically and FENCE can be a NOP. With a store buffer, it must be drained on rl and FENCE. QEMU TCG's SC is a value comparison, so it succeeds in the ABA test, contrary to the specification.

**Next post:** Part 21 implements the C extension. Using Python models, it builds a decompressor that expands 16-bit compressed instructions to 32 bits in front of the decoder, a fetch buffer that pulls out the next instruction when 16-bit and 32-bit instructions are mixed within 32-bit fetch words, and handling of 32-bit instructions spanning two words and 2-byte-aligned branch targets, then compares a real rv32imc program against GNU objdump.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Chapter 14, "A" Extension (14.1 Specifying Ordering, 14.2 Zalrsc, 14.3 Eventual Success of Store-Conditional, 14.4 Zaamo), Chapter 17, RVWMO Memory Consistency Model, Section 2.7, FENCE. https://github.com/riscv/riscv-isa-manual
- D. Harris et al., *RISC-V System-on-Chip Design*, 2023. Atomic instructions and cache implementation (CORE-V Wally)
- D. Patterson, A. Waterman, *The RISC-V Reader*, 2017. Chapter 6, RV32A
- K. Gharachorloo et al., "Memory Consistency and Event Ordering in Scalable Shared-Memory Multiprocessors," ISCA 1990
- `gen_sc` in the QEMU source `target/riscv/tcg/insn_trans/trans_rva.c.inc` (checked on master, 2026-11). https://github.com/qemu/qemu
