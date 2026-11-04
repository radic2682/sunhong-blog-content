---
id: "01M6K8E6ZY3K442JTG9QPV3ZGG"
title_en: "RISC-V Branches and Jumps: Resolve Stage, JAL/JALR, Return-Address Stack Hints"
summary_en: "Part 18 of the RISC-V series. I added the branch resolve stage, JAL redirect in ID, BTFN static prediction, and a RAS that follows the hints in spec section 2.5 to a pipeline model and compared them. CPI dropped from 1.368 to 1.227, and an 8-entry RAS predicted 98.3% of fib's returns."
---
:::note
**Key points in three lines**
- Most of the remaining loss in the part 17 pipeline came from branches and jumps. Resolving branches in EX costs 2 cycles per taken branch and moving it up to ID costs 1, but resolving in ID requires moving the comparator and forwarding into ID, and it stalls when comparing the result of the immediately preceding instruction. JAL knows its target from decode alone, while JALR depends on a register value and needs prediction.
- I added the resolve stage, a JAL redirect in ID, static prediction (backward means taken), and a return-address stack (RAS) that follows the rd·rs1 hints of spec section 2.5 to the same model, and measured the combined CPI of five programs: 1.368 (all in EX) → 1.332 (+JAL in ID) → 1.251 (+static prediction) → 1.227 (+8-entry RAS). Resolving everything in ID gave 1.300.
- An 8-entry RAS predicted 98.3% of fib's 21,892 `ret`s, with zero mispredictions (misses happened only when the stack was empty). For five cases of misaligned jump and branch targets, whether an exception occurred and the mepc·mtval values all matched QEMU.
:::

In part 17, the 5-stage pipeline with forwarding had a CPI of 1.37, and 0.36 of the 0.37 above 1 was cycles thrown away on branches and jumps. This is the control hazard. It arises because the next instruction has to be fetched without knowing where it is. This post covers how a pipeline handles RISC-V branches and jumps: which stage resolves them, static prediction, the difference between JAL and JALR, and the return-address stack hints the spec hides in register numbers. Dynamic branch predictors such as gshare and TAGE are not specific to RISC-V, so they are left for another category.

## 1. Where to Resolve Branches

The part 17 pipeline resolved branches in EX. The comparator next to the ALU learns taken in EX, and at that point the two instructions that have entered IF/ID and ID/EX are discarded. That is 2 cycles per taken branch.

![For a taken branch, resolving in EX costs 2 bubbles and resolving in ID costs 1, but comparing the result of the immediately preceding instruction stalls ID for a cycle|full](/img/01M6K8E6ZY3K442JTG9QPV3ZGG/34f37ead25b8.webp)

Moving the comparator to ID cuts the discarded instructions to one. There are three costs.

| Cost | Details |
|---|---|
| Hardware | A 32-bit comparator (equal, signed, and unsigned comparison) and a pc+imm adder move to ID. New forwarding paths (EX/MEM·MEM/WB → ID) and their multiplexers are needed |
| Dependency stalls | If the immediately preceding instruction produces the value to compare, it is only available at the end of EX → 1-cycle stall. 2 cycles if a load produces it |
| Critical path | ID gets "register read → forwarding select → 32-bit compare → next-pc select → PC" in series. ID easily becomes the longest stage |

The third hurts most in design. When resolving in EX, the comparator sits alongside the ALU, and the result only has to pass through the next-pc multiplexer. Resolving in ID adds the compare time on top of the register file read time. If the clock period grows, you lose what you gained in CPI. So designs that resolve in ID also try hard to keep the comparison simple (COD 4.9 gives an example that moves only the equality comparison of `beq` to ID). RISC-V also has magnitude comparisons like `blt`·`bltu`, so an equality comparison alone is not enough.

Let's count the cost by hand. The end of the inner loop of CRC-32 from part 17 looks like this.

```text
800000b4: addi a4,a4,-1
800000b8: bnez a4,800000a0
```

`bnez` compares the result of the `addi` right before it. With EX resolution, the `addi` result arrives through EX/MEM forwarding for the comparison, and if taken, 2 cycles are discarded. With ID resolution, `bnez` waits in ID for a cycle while `addi` is in EX, and then 1 cycle is discarded if taken. Both total 2 cycles. In loops like this, ID resolution gains nothing.

## 2. JAL and JALR

| | JAL | JALR |
|---|---|---|
| Target | pc + J immediate | (rs1 + I immediate) & ~1 |
| When it is known | At decode (ID) | Needs the rs1 value (EX, or ID with forwarding) |
| Main uses | Function call `call`, unconditional jump `j` | Return `ret`, indirect calls, jump tables |

JAL's target is entirely inside the instruction. It does not need to read a register, so it has no dependency stalls. Redirecting to pc+imm in ID as soon as the immediate is formed costs only 1 cycle. This improvement is possible regardless of where branches are resolved. Going further, predecoding instructions in the IF stage or a branch target buffer (BTB) can bring it down to 0 cycles, but the model in this post only goes as far as ID.

JALR is different: its target is in a register. The most common JALR, `ret` (`jalr x0, 0(ra)`), comes a few instructions after the function restores ra with `lw ra, 12(sp)` at its end. Computing it in ID requires forwarding ra, and if the load is close, it has to wait. Computing it in EX discards 2 cycles. Either way, reducing the loss requires knowing the target ahead of time without the register, and that is the return-address stack.

If the target of JAL·JALR is not a multiple of 4, an instruction-address-misaligned exception occurs (without the C extension, spec section 2.5). JALR checks after clearing the least significant bit, so what it really looks at is bit 1. Section 5 checks this rule against QEMU.

## 3. Static Prediction: Backward Means Taken

Instead of waiting for a branch, prediction guesses the outcome and fetches ahead. The "keep fetching pc+4" used so far is actually a prediction too: "always not-taken." Spec section 2.5 gives software this advice: lay out the frequently executed path sequentially, and assume backward branches are predicted taken and forward branches not-taken. This is BTFN (backward taken, forward not-taken) static prediction. Hardware needs to look at only one bit, the sign bit of the immediate (inst[31]). The end of a loop is a backward branch, and since loops usually repeat, it is likely to be right.

In the model, if the B immediate is negative in ID, the core redirects to the target (1 cycle), and the actual resolution happens in EX, discarding 2 cycles if wrong. I counted how often BTFN was right on five programs using execution traces from the single-cycle model.

| Program | Backward branches taken | Forward branches taken | BTFN accuracy |
|---|---|---|---|
| qsort | 49.4% (5,687) | 51.6% (790) | 49.3% |
| crc32 | 90.0% (20,494) | None | 90.0% |
| fib | 7/8 | 50.0% (21,891) | 50.0% |
| matmul | 93.5% (67,911) | 7.3% (62,976) | 93.1% |
| sieve | 92.3% (27,252) | 0.0% (7,998) | 94.0% |

qsort is the problem. It has many short loops like `while (v[i] < p) i++`, so only about half of its backward branches were taken. In programs like this, BTFN can be worse than not-taken. In fib, the single `n < 2` check is a forward branch, so it is fifty-fifty.

## 4. The Return-Address Stack and the Spec's Hints

A function's return target is the address right after the `call` that invoked it. Calls and returns come in pairs, so pushing pc+4 onto a stack on a call and popping it on a return predicts the target without reading the register. This is the return-address stack (RAS).

The question is how hardware knows "is this JALR a return?" Some other ISAs put separate hint bits in their jump instructions. RISC-V uses no bits; it signals through register numbers. The reference is x1 (ra), the return address register of the standard calling convention, and x5 (t0), the alternate link register. The rules in spec section 2.5 are as follows.

- JAL: push only when rd is x1 or x5.
- JALR: see the table below.

| rd is x1/x5 | rs1 is x1/x5 | rd=rs1 | RAS action |
|---|---|---|---|
| No | No | - | None |
| No | Yes | - | Pop |
| Yes | No | - | Push |
| Yes | Yes | No | Pop, then push |
| Yes | Yes | Yes | Push |

Applied to real code:

| Code | Encoding | RAS |
|---|---|---|
| `call f` | `jal ra, f` | Push |
| `ret` | `jalr x0, 0(ra)` | Pop |
| Indirect call | `jalr ra, 0(a5)` | Push |
| Jump table | `jalr x0, 0(a5)` | None |
| Coroutine switch | `jalr ra, 0(t0)` | Pop, then push |
| Far call | `auipc ra, …` + `jalr ra, …(ra)` | Push |

The last two rows are interesting. When rd and rs1 are different link registers, it is a coroutine switch that "returns and calls at the same time," so it both pops and pushes. When they are the same link register, it only pushes. The spec cites macro-op fusion as the reason: executing `lui ra, …; jalr ra, …(ra)` and `auipc ra, …; jalr ra, …(ra)` as one operation. These pairs are effectively "far calls," not returns. The spec also explains why x5 is the alternate link register: to avoid touching ra when calling short routines (millicode) such as register save/restore in compressed code, and because x5's encoding differs from x1's by only one bit, which makes decoding cheap.

The model's RAS operates in ID. When a JAL·JALR passes through ID, it pushes and pops according to the table above, and a popping JALR redirects immediately to the popped address (1 cycle). The real target is computed in EX, and if it differs, the core redirects again (2 cycles). If the stack is empty, no prediction is made. On overflow, the oldest entry is dropped. In this 5-stage pipeline, the next instruction passes ID only after the branch is resolved in EX, so wrong-path instructions never touch the RAS. A deeper pipeline would need a mechanism to roll back the RAS on a misprediction.

```python
def _ras_update(self, d, pc):
    rd_l, rs_l = d.rd in (1, 5), d.rs1 in (1, 5)
    pop = (d.j == 'jalr' and rs_l
           and (not rd_l or d.rd != d.rs1))
    push = d.j in ('jal', 'jalr') and rd_l
    top = None
    if pop and self.ras: top = self.ras.pop()
    if push:
        self.ras.append((pc + 4) & 0xFFFFFFFF)
        if len(self.ras) > self.o['ras']: self.ras.pop(0)
    return top
```

## 5. Hands-On: CPI by Configuration and Alignment Exceptions

I added options to the part 17 model to build five configurations. In every configuration, the results of all five programs matched QEMU (UART output, pc sequence, final registers).

| Configuration | Description | Combined CPI |
|---|---|---|
| A | Branches, JAL, JALR all in EX (part 17) | 1.368 |
| B | A + JAL in ID | 1.332 |
| C | B + BTFN static prediction | 1.251 |
| D | C + 8-entry RAS | 1.227 |
| E | Branches, JAL, JALR all resolved in ID | 1.300 |

![CPI by branch-handling configuration is A 1.37, B 1.33, C 1.25, D 1.23, E 1.30, and only qsort gets worse with BTFN|xl](/img/01M6K8E6ZY3K442JTG9QPV3ZGG/0fec7f349e9f.webp)

Breaking down lost cycles by cause shows what each configuration changed (total over 5 programs, 1,076,364 instructions).

| Cause | A | C | D | E |
|---|---|---|---|---|
| Branch flushes | 251,732 | 164,724 | 164,724 | 125,866 |
| ID dependency stalls | 0 | 0 | 0 | 118,136 |
| JAL flushes | 78,862 | 39,431 | 39,431 | 39,431 |
| JALR flushes | 52,512 | 52,512 | 26,633 | 26,256 |

- **B: JAL in ID.** Each of the 39,431 JALs discarded 1 fewer cycle. For the price of one more adder in decode, it is the easiest improvement.
- **C: BTFN.** Branch losses dropped by 87,008 cycles. Programs with long loops like crc32, matmul, and sieve improved a lot (matmul 1.354 → 1.224), while qsort got worse, from 1.310 to 1.386, exactly as in the Section 3 table.
- **D: RAS.** JALR losses were nearly halved, because a correctly predicted return discards only 1 cycle. fib went from 1.290 to 1.227.
- **E: Everything in ID.** Branch flushes are lowest at 125,866 (1 per taken branch), but 118,136 cycles of ID dependency stalls appeared. That shows how common the `addi; bnez` pattern from Section 1 is. crc32 actually got worse (1.272 → 1.286), and matmul improved only slightly, from 1.364 to 1.333. On top of that, E lengthens the critical path. On CPI alone, C and D beat E, and the hardware does not need a comparator in ID.

![In fib, growing the RAS depth through 0, 1, 2, 4, 8, and 16 entries raises the ret hit rate from 0% to 50%, 69%, 88%, 98.3%, and 100%, and lowers CPI from 1.290 to 1.226|xl](/img/01M6K8E6ZY3K442JTG9QPV3ZGG/14aa59b004b5.webp)

I measured RAS depth with fib, which recurses deeply. fib(20) reaches a call depth of about 20, so with 8 entries the stack runs empty when returning from the deepest levels. With 8 entries it predicted 21,515 of 21,892 (98.3%), and the remaining 377 were cases where the stack was empty and no prediction was made. Regardless of depth, **there were zero mispredictions**, because in code that follows the calling convention, pushes and pops pair up exactly. qsort got 259 of 265 with 4 entries and all of them with 8.

Finally, I rechecked x0. In part 17, all compiled programs still passed even after removing the rd≠0 term from the forwarding condition, because jumps were resolved in EX, so a jump and its target instruction never met inside the pipeline. In configuration B (only JAL in ID), all five still passed. In configuration E, qsort and fib did not finish. When `ret` is resolved in ID, `ret` is in WB when the first target instruction enters EX. The instruction after fib's last `ret` is main's `li a4,28` (`addi a4, x0, 28`), and `ret`'s pc+4 (`0x8000004c`) was forwarded into the x0 slot, breaking the iteration count of the output loop. Just changing the resolve stage exposed a hidden bug.

For misaligned targets, I wrote five cases in assembly and compared them with QEMU (`-cpu rv32,c=false`; the C extension must be off for alignment exceptions to occur). For QEMU, I extracted the exception cause, mepc, and mtval from the `-d int` log.

| Case | QEMU | Model (single-cycle, EX, ID) |
|---|---|---|
| Not-taken branch, target pc+6 | No exception | No exception |
| `jalr`, target label+1 | No exception | No exception |
| `jalr`, target label+2 | mepc `0x8000002c` | Same |
| Taken branch, target pc+6 | mepc `0x80000024` | Same |
| `jal`, target pc+6 | mepc `0x80000024` | Same |

In all three cases that raised an exception, the cause was 0 (instruction address misaligned), mepc was the address of the jump or branch instruction itself, and mtval was the misaligned target (`0x80000032` for `jalr`, `0x8000002a` for the other two). The second row is exactly per the spec: label+1 is odd, but JALR clears the least significant bit, so the address becomes aligned. The first row is also per the spec: if a branch is not taken, there is no exception even with a strange target. So the alignment check can only happen after taken is determined, and it must be done together with resolution (in EX or ID).

## 6. Design Checklist: Branches and Jumps

| Item | Rule |
|---|---|
| Branch resolve stage | EX: 2 cycles per taken branch. ID: 1 cycle + dependency stalls (1 after an ALU op, 2 after a load), comparator and forwarding in ID, longer critical path |
| JAL | The target is inside the instruction. Redirect in ID (1 cycle); 0 with predecode or a BTB |
| JALR target | (rs1+imm) with bit 0 cleared. imm is not doubled |
| Alignment exception | If bit 1 of the target of a taken branch, JAL, or JALR is 1 (without C), that jump raises an exception. mepc=jump pc, mtval=target. Not-taken branches are not checked |
| Link value | rd ← pc+4. Even when rd=x0, pc+4 travels down the pipeline (rd≠0 is required in forwarding) |
| Static prediction | BTFN only needs inst[31]. In code with many short loops it can be worse than not-taken |
| RAS push | rd of JAL·JALR is x1 or x5 |
| RAS pop | rs1 of JALR is x1 or x5, and rd is not a link register or rd≠rs1 |
| rd=rs1=link | Push only (to support fusing `auipc ra; jalr ra, …(ra)`) |
| RAS overflow/empty | The common choice is a circular buffer (overwrite the oldest); no prediction when empty. Deep pipelines must roll back wrong-path updates |
| Prediction check | Carry the predicted next pc in the pipeline registers, compare it with the actual value at the resolve stage, and flush on mismatch |

## Summary

- Resolving branches in EX loses 2 cycles per taken branch; resolving in ID loses 1. ID resolution requires moving the comparator and forwarding into ID and brings dependency stalls and a longer critical path. In this measurement, 118,136 cycles of dependency stalls canceled out most of the flushes saved.
- JAL knows its target at decode, so a redirect in ID costs 1 cycle; JALR depends on a register and needs prediction. If the target is not a multiple of 4, that jump raises an exception, and five cases matched QEMU.
- Spec section 2.5 defines RAS push and pop by whether rd·rs1 are x1/x5. An 8-entry RAS predicted 98.3% of fib's returns with no mispredictions.
- Combined CPI went 1.368 (all in EX) → 1.332 (+JAL in ID) → 1.251 (+BTFN) → 1.227 (+RAS), and resolving everything in ID gave 1.300.

**Next:** Part 19 implements the M extension. It looks at how to attach a multiplier and a divider to the pipeline (multi-cycle operations and stalls), and how to produce exactly the division-by-zero and overflow results the spec defines. It also measures how the numbers in this series change when the `__mulsi3` loop, which made up 31% of matmul's branches in part 16, becomes a single `mul`.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. 2.5 Control Transfer Instructions (JAL·JALR, return-address stack hint table, alignment exceptions, static prediction advice). https://github.com/riscv/riscv-isa-manual
- The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Version 20240411. Chapter 3, Machine-Level ISA (mcause exception codes, mepc, mtval)
- D. Patterson, J. Hennessy, *Computer Organization and Design: RISC-V Edition*, 2nd ed., 2020. 4.9 Control Hazards (moving branch resolution to ID, static prediction)
- D. Harris, S. Harris, *Digital Design and Computer Architecture: RISC-V Edition*, 2021. 7.5.3 Hazards (control hazards)
- QEMU RISC-V CPU properties (`-cpu rv32,c=false`) and logging (`-d int`). https://www.qemu.org/docs/master/system/target-riscv.html
