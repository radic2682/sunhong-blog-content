---
id: "01M6K8E2MPEJ5G51N0JWQC7XQ0"
title_en: "RISC-V 5-Stage Pipeline: Forwarding and Stall Conditions, Handling x0"
summary_en: "Part 17 of the RISC-V series. I ran the same programs on a 5-stage pipeline model, matched them against QEMU, and measured CPI: 1.37 with forwarding, 1.83 without. Dropping the rd≠0 condition made a program using the HINT ntl.all write 0x80000000 instead of 0."
---
:::note
**Key points in three lines**
- The single-cycle datapath from part 16 is cut into five stages (IF, ID, EX, MEM, WB). Data hazards are resolved with EX/MEM and MEM/WB forwarding plus a 1-cycle load-use stall, and the register file resolves the rest with "write in the first half, read in the second half." The forwarding condition is `RegWrite ∧ rd≠0 ∧ rd=rs`.
- I ran five programs on a cycle-level model, and all results matched QEMU. Aggregate CPI was 1.37 with forwarding and 1.83 without. Of the 0.37 loss remaining with forwarding, 0.36 was branch/jump flushes and the load-use stall was 0.013.
- Dropping rd≠0 from forwarding still left the five compiled programs correct. But a program using the HINT `ntl.all` (`add x0, x0, x5`) wrote `0x80000000` into 256 slots that should have been 0, and in a design that decodes "instructions that don't write have rd=0," all six programs failed.
:::

The single-cycle core from part 16 finishes one instruction per cycle. The price is that the clock period is set by the length of the slowest instruction (usually `lw`). A pipeline cuts this path into five segments and processes a different instruction in each segment at the same time. Ideally, one instruction still finishes per cycle while the clock gets nearly five times faster. This post is about where that "ideally" breaks down: hazards. In particular, it uses programs that actually fail to show how RISC-V's x0 rule must enter the forwarding condition.

## 1. Five Stages and Pipeline Registers

Cut the part 16 diagram at the following boundaries.

| Stage | What it does | What it latches at the end (pipeline register) |
|---|---|---|
| IF | Fetch the instruction at pc, pc+4 | IF/ID: pc, instruction |
| ID | Decode, immediate, register read, hazard detection | ID/EX: pc, control signals, rs1/rs2 values, immediate, rs1/rs2/rd numbers |
| EX | Forwarding select, ALU, branch decision | EX/MEM: control signals, ALU result (or pc+4), store data, rd |
| MEM | Load/store | MEM/WB: control signals, value to write, rd |
| WB | Register write | (register file) |

It's easy to miss that ID/EX holds the rs1/rs2 **numbers**. The values were read in ID, but the forwarding check happens in EX, so the numbers have to travel to EX too. Control signals travel along until the stage that uses them, dropping off one by one: the ALU operation is last used in EX, MemRead/MemWrite in MEM, and RegWrite and the writeback select in WB.

The model (`17/pipe.py`) computes each stage's combinational logic every cycle in the order WB → MEM → EX → ID → IF, then updates all four pipeline registers at once at the end. The reason for computing in reverse is that in hardware every stage sees "the register values at the start of this cycle." Computing from the front stage makes it easy to slip into letting a later stage see a value changed in the same cycle. Empty slots (bubbles) flow through carrying a cause label and are counted by cause when they reach WB. That way `cycles = completed instructions + bubbles by cause` adds up exactly.

## 2. Data Hazards and Forwarding

GCC produced this for the CRC-32 inner loop.

```text
800000a0: srli a3,a5,0x1
800000a4: slli a1,a5,0x1f
800000a8: srai a5,a1,0x1f    # a1: result just before
800000ac: and  a5,a5,a2      # a5: result just before
800000b0: xor  a5,a3,a5      # a5: just before, a3: four back
800000b4: addi a4,a4,-1
800000b8: bnez a4,800000a0   # a4: result just before
```

When `srai` is in EX, `slli` is in MEM. The result of `slli` is in the EX/MEM register but not yet in the register file. The a1 that `srai` read in ID is the old value. This is a RAW (read-after-write) data hazard.

![Forwarding paths that route EX/MEM and MEM/WB results back to the ALU inputs in EX in a 5-stage pipeline, and the hazard detector that stalls the front stages on a load-use|full](/img/01M6K8E2MPEJ5G51N0JWQC7XQ0/29c565f9e82c.webp)

There are two fixes: wait until the value reaches the register file (stall), or route the already computed value back to EX (forward). By distance between producer and consumer:

| Distance | Producer location | Resolution |
|---|---|---|
| 1 back | MEM (EX/MEM) | EX/MEM forwarding |
| 2 back | WB (MEM/WB) | MEM/WB forwarding |
| 3 back | Writing the register file | Same-cycle write→read |
| 4 or more | In the register file | Plain read |

The condition for the forwarding mux (in front of the ALU input) matches the equation below the figure. Written for rs1 (rs2 is identical):

```text
if   EX/MEM.RegWrite and EX/MEM.rd != 0
     and EX/MEM.rd == ID/EX.rs1:  A = EX/MEM.result
elif MEM/WB.RegWrite and MEM/WB.rd != 0
     and MEM/WB.rd == ID/EX.rs1:  A = MEM/WB.result
else:                             A = ID/EX.rs1 value
```

The order matters. If the instructions 1 back and 2 back both write the same register, the more recent one, EX/MEM, must win. The `xor a5,a3,a5` after `and a5,a5,a2` in the loop above is that case. Both `srai` and `and` wrote a5, but what `xor` must receive is the result of `and`.

It's also easy to miss that the value forwarded from EX/MEM must be "the value this instruction will write to the register," not "the ALU output." For `jal ra, f`, the ALU output is a useless value; what goes into ra is pc+4. The model builds `result = pc+4 if wb=='4' else ALU` in EX ahead of time and latches it into EX/MEM.

## 3. Load-Use Stall and the Register File

A load's value only comes out at the end of MEM. For the very next instruction to use it at the start of EX would require going back in time. So the pipeline stalls for one cycle. The `xor a5,a5,a4` after `lbu a4,0(a0)` in the CRC-32 outer loop is that case.

```text
cycle       1   2   3   4   5   6   7
lbu a4      IF  ID  EX  MEM WB
xor a5,a4       IF  ID  ID  EX  MEM WB
                        ↑ bubble, MEM/WB → EX forwarding
```

Detection happens in ID. If the instruction in ID/EX (the one now in EX) is a load and its rd matches rs1/rs2 of the instruction now in ID, the PC and IF/ID are held and a bubble is inserted into ID/EX. This is where part 15's "does it actually read rs1/rs2" signal is used. Comparing without it causes needless stalls when immediate bits of `lui` or `jal` happen to match.

A result 3 back is resolved not by forwarding but by the register file. In the cycle the producer writes in WB, the consumer reads the same register in ID. If the register file writes in the first half of the clock (or on the falling edge) and reads in the second half, the consumer sees the new value (the approach in DDCA 7.5). The alternative is a bypass inside the register file from write data → read output. Without one or the other, results are wrong. When I turned this feature off in the model, all six programs failed (section 5).

## 4. x0: Why rd≠0 Belongs in the Condition

An instruction that writes x0 discards its result. Inside the pipeline, however, that "to-be-discarded value" rides along in EX/MEM and MEM/WB as is. If rd≠0 is dropped from the condition, a following instruction that reads x0 gets that value forwarded instead of 0 from the register file. Instructions that read x0 are extremely common: `li a0,5` (`addi a0,x0,5`), `beqz a0` (`beq a0,x0`), and `sw zero,0(a5)` all read x0.

So which instructions write a nonzero value to x0? Using the part 16 model, I counted the instructions with "rd=x0 and RegWrite=1" that the five programs actually executed.

| Program | `jal x0` (j) | `jalr x0` (ret) | Other |
|---|---|---|---|
| qsort | 1,679 | 265 | 0 |
| crc32 | 0 | 1 | 0 |
| fib | 10,945 | 21,892 | 0 |
| matmul | 0 | 4,097 | 0 |
| sieve | 551 | 1 | 0 |

`j` and `ret` "write" pc+4 to x0, a nonzero value. Yet in this pipeline it causes no problem. Jumps are resolved in EX, and the two instructions that followed them in are flushed at that point. By the cycle the first instruction at the jump target reaches EX, the jump has already passed WB and left the pipeline. There is nothing left to forward from. Indeed, running with rd≠0 removed, all five still matched QEMU. GCC doesn't generate code that writes computed results to x0 other than `nop`, and these five programs didn't execute any `nop` either.

Concluding from this that "the x0 condition isn't needed" would be wrong. I checked two cases.

**First, HINTs.** Part 15 described HINTs as instructions that "just execute normally and drop the write to x0." The non-temporal hint `ntl.all` from the Zihintntl extension (Chapter 9 of the specification) is encoded as `add x0, x0, x5`. It hints that cache pollution should be reduced when storing data that won't be reused, and an RV32I core that doesn't know this extension must still be able to execute it as is. I wrote a program (`17/progs/ntl.c`) that fills a 512-entry buffer and then clears the even entries to 0 with `ntl.all` attached. The key part of the compiled output is two lines.

```text
8000008c: add zero,zero,t0    # ntl.all (HINT)
80000090: sw  zero,0(a5)      # store 0: reads x0 as rs2
```

The value `add` computes in EX is 0 + t0 = t0. In the next cycle, when `sw` selects rs2=x0 in EX, this value in EX/MEM gets forwarded if there is no rd≠0 check. t0 held `0x80000000`, left there by QEMU's reset ROM.

| Configuration | UART output | Result |
|---|---|---|
| QEMU, correct model | `ntl 4aefd727 00000100 00000000` | 256 zero entries, buf[0]=0 |
| Model without rd≠0 | `ntl 63a7a3ea 00000000 80000000` | 0 zero entries, buf[0]=`0x80000000` |

All 256 entries that should have been cleared got `0x80000000`. The program finishes cleanly and only the numbers are wrong: the hardest kind of bug to find.

**Second, designs that use rd in place of RegWrite.** If the decoder is defined so that "instructions that don't write a register (stores, branches, FENCE) output rd as 0," the forwarding condition can drop RegWrite and check only `rd≠0 ∧ rd=rs`. It's a common way to save one comparator. In this design, if you also drop rd≠0, the ALU result of every store and branch (the address, or the sum used for comparison) gets forwarded to a following x0 read.

| Forwarding condition | With x0 condition | Without x0 condition |
|---|---|---|
| RegWrite ∧ rd≠0 ∧ rd=rs | 6/6 correct | 5/6 correct (ntl wrong) |
| rd≠0 ∧ rd=rs (rd=0 if no write) | 6/6 correct | 0/6 correct |

The failures in the second row were loud. qsort jumped to a misaligned address and stopped on an exception, fib's output was garbled from the first character (`rib 10000014…`), and matmul, sieve, and ntl never finished. In this scheme, rd≠0 is not "special handling for x0" but the write enable itself.

Part 18 adds one more case. If jumps are resolved early in ID, a jump and its target instruction meet inside the pipeline, and then the pc+4 of `j`/`ret` really does get forwarded. A bug that was hidden with EX resolution surfaces the moment you change the microarchitecture. The right answer is to put rd≠0 in the condition from the start.

## 5. Experiment: CPI and Stall Causes

The model configuration is the baseline: branches and jumps resolve in EX, and meanwhile IF keeps fetching from pc+4 (no prediction = not-taken). Taken branches and all jumps flush IF/ID and ID/EX, losing 2 cycles. Memory is assumed to take 1 cycle for both instructions and data (no caches). I ran the same five programs and checked them against QEMU the same way as in part 16 (UART, the pc sequence in completion order, final registers); all six programs matched both with forwarding on and off.

```python
def _fwd(self, r, v):              # one EX-stage input
    if not self.o['fwd']: return v
    if self._match(self.exmem, r): return self.exmem.res
    if self._match(self.memwb, r): return self.memwb.res
    return v

def _match(self, p, r):            # shared by forwarding and hazard detection
    return (not isinstance(p, str) and p.d.rw
            and p.d.rd == r
            and (r != 0 or not self.o['x0']))
```

![Aggregate CPI is 1.37 with forwarding and 1.83 without, and most of the loss remaining with forwarding is branch/jump flushes|xl](/img/01M6K8E2MPEJ5G51N0JWQC7XQ0/1da4c4751070.webp)

| Loss cause (total) | Forwarding on | Forwarding off |
|---|---|---|
| Load-use stall | 13,458 | (included in data stalls) |
| Data hazard stall | 0 | 511,397 |
| Branch flush | 251,732 | 251,732 |
| `jal` flush | 78,862 | 78,862 |
| `jalr` flush | 52,512 | 52,512 |
| CPI (1,076,364 instructions) | 1.368 | 1.831 |

Let's read the numbers.

- **The effect of forwarding.** With it off, instructions that use the result from 1 or 2 back wait up to 2 cycles each. Data stalls came to 511,397 cycles, 0.48 per instruction. crc32 rose as high as 2.36, because almost every instruction uses the result right before it, as in the loop above.
- **What remains with it on.** Load-use stalls are only 13,458 cycles (0.013 per instruction). 97% of the remaining loss is branch/jump flushes. They line up exactly with the numbers from part 16: 125,866 taken branches × 2 = 251,732, 39,431 `jal` × 2 = 78,862, and 26,256 `jalr` × 2 = 52,512.
- **By program.** sieve was highest at 1.44: 26% of its instructions are branches and 71% of those are taken. fib has large jump flushes (0.32 per instruction for `jal` and `jalr` combined), and qsort had the most load-use stalls at 0.08 per instruction (it compares array elements right after loading them).

The forwarding circuit is just two 32-bit 3-input muxes, four 5-bit comparators, and the rd≠0 check. In return, CPI dropped 25%, from 1.83 to 1.37. The remaining loss, on the other hand, is mostly control hazards. Part 18 tackles that.

## 6. Design Checklist: The 5-Stage Pipeline

| Item | Rule |
|---|---|
| Forwarding condition | `RegWrite ∧ rd≠0 ∧ rd=rs`. Separately for rs1 and rs2; EX/MEM takes priority over MEM/WB |
| Forwarded value | The value to be written, not the ALU output (pc+4 for `jal`/`jalr`) |
| Load result | Comes out at the end of MEM, so 1 back stalls; from 2 back, MEM/WB forwarding |
| Hazard check targets | Only rs fields that are actually read. The rs1 position in U/J formats and the rs2 position in I format are immediate bits |
| Register file | Same-cycle write→read pass-through (first-half write or internal bypass). Without it, 3 back is wrong |
| x0 | The register file doesn't store register 0, and forwarding/stall conditions also include rd≠0 |
| Decoder that zeroes rd | If instructions that don't write output rd as 0, the RegWrite comparison can be dropped. rd≠0 then stands in for the write enable |
| HINT | Executes normally and "writes" x0. Include it in test programs to check that forwarding filters it out |
| Store data | rs2 is also a forwarding target (selected in EX and latched into EX/MEM) |
| Flush | On a redirect from EX, clear IF/ID and ID/EX. Takes priority over a stall in ID |

The HINT row in the checklist connects to verification. If you test only with compiler-generated programs, x0 bugs slip through. That is why the integer instruction tests in riscv-tests separately test writing results to x0 with the `TEST_RR_ZERODEST` and `TEST_IMM_ZERODEST` macros (part 34).

## Summary

- A 5-stage pipeline cuts the single-cycle datapath into IF, ID, EX, MEM, and WB, and the pipeline registers carry values, control signals, and register numbers to the next stage.
- Data hazards are resolved by forwarding for 1 and 2 back, by the register file's write→read pass-through for 3 back, and by a 1-cycle stall right after a load. Forwarding brought aggregate CPI from 1.83 to 1.37.
- With forwarding on, 97% of the remaining loss was branch/jump flushes (0.36 per instruction). The load-use stall was 0.013.
- Dropping rd≠0 from the forwarding condition broke a program that uses a HINT, and in a design that uses rd in place of RegWrite, every program broke. Passing with compiled programs does not mean x0 handling is correct.

**Next post:** Part 18 tackles the remaining loss: control hazards. In the same model, it compares by CPI the cost of resolving branches in EX versus moving them up to ID (flush cycles versus added forwarding, comparators, and critical path), static prediction that treats backward branches as taken, JAL whose target is known at decode versus register-dependent JALR, the return address stack hints the specification defines by rd and rs1 numbers, and the misaligned target exception.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. 2.1 Programmers' Model (x0), 2.9 HINT Instructions, Chapter 9 Zihintntl (`ntl.all` encoding). https://github.com/riscv/riscv-isa-manual
- D. Harris, S. Harris, *Digital Design and Computer Architecture: RISC-V Edition*, 2021. 7.5 Pipelined Processor (forwarding, load stalls, falling-edge register file writes)
- D. Patterson, J. Hennessy, *Computer Organization and Design: RISC-V Edition*, 2nd ed., 2020. 4.6 An Overview of Pipelining, 4.7 Pipelined Datapath and Control, 4.8 Data Hazards: Forwarding versus Stalling
- riscv-tests, `TEST_RR_ZERODEST` and `TEST_IMM_ZERODEST` in `isa/macros/scalar/test_macros.h`. https://github.com/riscv-software-src/riscv-tests
