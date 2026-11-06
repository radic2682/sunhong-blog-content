---
id: "01M6K7XXTWKTP2N0FCF1DVQEFY"
title_en: "Implementing the M Extension: Multiply/Divide Units and Divide-by-Zero"
summary_en: "Part 19 of the RISC-V series. I implemented the M extension's multiplier and divider as bit-exact Python models. A single 33-bit multiplier handles all three mulh variants, with zero mismatches across 300,000 pairs and 648 QEMU results, and I found a trap when dividing a negative number by zero (5,039 mismatches)."
---
:::note
**Key points in three lines**
- The 8 M-extension instructions are built from one multiplier and one divider. Extending each operand by 1 bit to 33 bits lets a single 33×33 signed multiply cover `mul`, `mulh`, `mulhsu`, and `mulhu`.
- An unsigned iterative divider (restoring or non-restoring) produces the spec values for divide-by-zero (quotient all ones, remainder = dividend) with no special handling. But building signed division as "divide the absolute values and restore the sign" makes the quotient 1 for negative ÷ 0. Of 10,072 tests, 5,039 were wrong, and one extra condition fixed it.
- Six bit-exact Python models (2 multipliers, 4 dividers) had zero mismatches against a reference on 5,184 edge-case pairs and 300,000 random pairs, and all 648 QEMU results matched too. Adding early termination cut the average number of division iterations from 32 to 5.98 (with operand sizes drawn uniformly).
:::

The RISC-V core implementation group has gone through the decoder, the datapath, the 5-stage pipeline, and branches. Starting with this part, I add standard extensions one at a time to that RV32I 5-stage pipelined core. The first is the M extension (multiply and divide). As seen in Part 7, there are only 8 instructions, but implementing them means fitting units that don't finish in one cycle into the pipeline. Instead of RTL, this post writes the multiplier and divider as bit-exact Python models, and checks directly whether the spec's divide-by-zero and overflow rules fall out of the algorithms naturally.

## 1. What the M Extension Asks of the Hardware

All M-extension instructions are R format, with opcode OP (`0110011`) and funct7 `0000001`. funct3 selects one of the 8 (Unprivileged spec, chapter 13).

| funct3 | Instruction | Result |
|---|---|---|
| 000 | `mul` | Lower 32 bits of the product |
| 001 | `mulh` | Upper 32 bits of signed × signed |
| 010 | `mulhsu` | Upper 32 bits of signed (rs1) × unsigned (rs2) |
| 011 | `mulhu` | Upper 32 bits of unsigned × unsigned |
| 100 | `div` | Signed quotient (rounded toward zero) |
| 101 | `divu` | Unsigned quotient |
| 110 | `rem` | Signed remainder (sign = dividend's sign) |
| 111 | `remu` | Unsigned remainder |

funct3[2] separates multiply (0) from divide (1). The decoder only needs this one bit to decide which unit to send the instruction to.

There are no exceptions at all. Dividing by zero, or overflowing the quotient as in `-2^31 ÷ -1`, produces a defined value without a trap. The rules from Table 11 of the spec are as follows.

| Case | Quotient (`div`, `divu`) | Remainder (`rem`, `remu`) |
|---|---|---|
| Divide by zero | All bits 1 (`0xffffffff`) | The dividend unchanged |
| `0x80000000 ÷ -1` (`div` only) | `0x80000000` | 0 |

The spec explains that these values were chosen because they are "what a simple unsigned division circuit naturally produces." Section 4 of this post checks with the model how far that claim holds.

## 2. Multiplier Options

Choosing a multiplier is a trade-off between area and latency. For a 5-stage pipelined core, there are three common options.

| Approach | Latency | Area | Pipeline integration |
|---|---|---|---|
| Array (single cycle) | 1 | Largest, lengthens the EX path | No stalls |
| Pipelined multiplier | 2–3 | Large | Fixed latency, back-to-back issue |
| Iterative radix-2 | 33 | One adder | Stall while busy |
| Iterative radix-4 Booth | 17 | Adder + 2x select | Stall while busy |

An array multiplier adds 1,024 partial products of a 32×32 multiply in one cycle. On FPGAs, this is often built by combining a few DSP blocks (a guess), while on ASICs it lengthens the critical path of the EX stage and cuts the clock. A pipelined multiplier splits the same array into 2–3 stages with registers. The latency is fixed and it can accept a new multiply every cycle, so only dependent instructions need to wait for the result.

The iterative approach uses one adder to add one partial product (radix-2) or two (radix-4) per cycle. It has the smallest area and suits microcontroller-class cores. The model in this post implements the two iterative versions. Their latency is long and what happens each cycle is clear, so they map directly to a state machine when moved to RTL.

## 3. One Multiplier, Four Instructions: 33-Bit Sign Extension

`mulh`, `mulhsu`, and `mulhu` multiply the same bits under different interpretations. We cannot have three multipliers, so the interpretation difference is absorbed on the input side. Each operand gets 1 extra bit on top: when read as signed, bit 31 is copied (sign extension), and when read as unsigned, a 0 is added. Then a 32-bit unsigned value is also exactly representable as a 33-bit signed value, and the multiplier only ever has to do a 33×33 signed multiply.

![Extending rs1 and rs2 by 1 bit to 33 bits lets a single 33×33 signed multiplier handle mul, mulh, mulhsu, and mulhu|full](/img/01M6K7XXTWKTP2N0FCF1DVQEFY/27164c12d354.webp)

Let's read the same bits, `0xffffffff × 0xffffffff`, three ways. These are results run on QEMU.

| Instruction | Two inputs extended to 33 bits | Product (64 bits) | Upper 32 bits |
|---|---|---|---|
| `mulh` | −1 × −1 | 1 | `0x00000000` |
| `mulhsu` | −1 × 4,294,967,295 | −4,294,967,295 | `0xffffffff` |
| `mulhu` | 4,294,967,295 × 4,294,967,295 | `0xfffffffe_00000001` | `0xfffffffe` |

The lower 32 bits are `0x00000001` in all three. In two's complement, the lower half does not depend on the signedness. That is why `mul` can pick sa and sb arbitrarily.

A 33×33 product is 66 bits, but the top 2 bits are discarded. If both inputs are 33-bit signed values, the magnitude of the product does not exceed $2^{64}$, so the lower 64 bits alone determine the result.

The iterative radix-2 model works like this. An accumulator A (34 bits) sits next to a multiplier register Q (33 bits), and each cycle, if the lowest bit of Q is 1, the multiplicand is added to A, then {A, Q} as a whole is arithmetically shifted right by 1 bit. In a 33-bit signed number, the top bit has a weight of $-2^{32}$, so only the last (33rd) cycle subtracts instead of adding. Radix-4 Booth extends the multiplier to 34 bits, looks at 2 bits at a time, and adds −2, −1, 0, +1, or +2 times the multiplicand, so it takes 17 cycles.

```python
def mul_radix2(a33, b33):
    A, Q, Mc = 0, b33 & ((1 << 33) - 1), a33
    for i in range(33):
        if Q & 1:
            A = wrap(A - Mc if i == 32 else A + Mc, 34)
        Q = (Q >> 1) | ((A & 1) << 32)  # {A,Q} >> 1
        A >>= 1                          # arithmetic shift
    return ((A << 33) | Q) & ((1 << 64) - 1), 33
```

`wrap(v, 34)` discards the bits that overflow a 34-bit register. Running 300,000 pairs at this width never produced an error, so 34 bits is enough for A.

## 4. The Divider and Divide-by-Zero: Checking It Directly

Division is harder to make fast than multiplication. Most small cores use an iterative divider that produces one quotient bit per cycle. The basic flow is the same as long division by hand. Shift the remainder register R left by 1 bit while bringing in the next bit of the dividend; if the divisor can be subtracted from R, subtract it and record a quotient bit of 1, otherwise record 0.

Let's trace `11 ÷ 3` (`1011 ÷ 0011`) with 4 bits.

| Step | R (after bringing in a bit) | R − 3 (new R if ≥ 0) | Quotient bit |
|---|---|---|---|
| 1 | 1 | −2 → R stays 1 | 0 |
| 2 | 2 | −1 → R stays 2 | 0 |
| 3 | 5 | 2 → R = 2 | 1 |
| 4 | 5 | 2 → R = 2 | 1 |

Quotient `0011` = 3, remainder 2. This is **restoring** division: if the subtraction result is negative, it "restores" the original R. **Non-restoring** division does not restore; while R is negative, the next step adds the divisor instead of subtracting it. Each step needs only one adder with an add/subtract select, and at the end, if R is negative, the divisor is added once more as a correction. Both iterate 32 times for 32 bits.

Now what happens when the divisor is 0? Doing `1011 ÷ 0000` in the same 4-bit example, R − 0 ≥ 0 at every step, so the quotient bit is always 1. The quotient is `1111`, and R takes in the dividend bits one by one, ending up as `1011`. That is exactly the spec value (quotient all ones, remainder = dividend). Non-restoring also never lets R go negative, so it gives the same result. Here are the results of running 10,072 divide-by-zero cases (72 edge cases + 10,000 random) on the models.

| Implementation | Mismatches for `divu`/`remu` or `div` |
|---|---|
| Restoring, unsigned, no special handling | 0 / 20,144 |
| Non-restoring, unsigned, no special handling | 0 / 20,144 |
| Signed (absolute value + sign restore), `div` | **5,039 / 10,072** |
| Signed, `rem` | 0 / 10,072 |

Signed division usually reuses the unsigned circuit. Divide the absolute values of the two operands, then negate the quotient if the two signs differ and negate the remainder if the dividend is negative. But look at `-7 ÷ 0`. |−7| ÷ 0 = quotient `0xffffffff`, remainder 7. The dividend is negative and the divisor 0 reads as "positive," so the signs differ and the quotient is negated. `-0xffffffff` truncated to 32 bits is `0x00000001`. The spec requires `0xffffffff`. Every negative dividend (about half) was wrong. The remainder negates 7 to get −7, which is the dividend unchanged, so it is correct.

The fix is a single condition.

```python
neg_q = (sa < 0) != (sb < 0) and b != 0
```

"If the divisor is 0, don't restore the quotient's sign." The spec's "natural value from an unsigned circuit" is true, but the sign-restore step needs this one condition.

Overflow needed no special handling. `0x80000000 ÷ -1` in absolute values is `0x80000000 ÷ 1` = quotient `0x80000000`, remainder 0, and the two signs are the same so nothing is negated. The result `0x80000000` and 0 matches the spec value exactly. You just have to treat the 32-bit absolute value `0x80000000` as an unsigned number.

## 5. Early Termination: Fewer Iterations and Its Trap

Most divisions in real programs are between small numbers (a guess). Running all 32 iterations is wasteful, so many cores add early termination. I modeled two approaches.

- **Skip leading zeros:** pre-shift left by the number of leading zero bits of the dividend, clz(a), and iterate only 32 − clz(a) times. The quotient bits for the leading zero bits are obviously 0.
- **Align the divisor:** shift the divisor left up to the height of the dividend and iterate only clz(b) − clz(a) + 1 times. If a < b, the quotient is 0 and the remainder is a right away.

![On 100,000 pairs with operand sizes drawn uniformly, the average iteration count dropped well below 32, to 15.47 for skipping leading zeros and 5.98 for aligning the divisor|xl](/img/01M6K7XXTWKTP2N0FCF1DVQEFY/26271f72ac1a.webp)

| Operand distribution (100,000 pairs) | Skip-zeros average | Align average | Align ≤ 8 iterations |
|---|---|---|---|
| Uniform 32-bit | 31.0 | 1.67 | 99.7% |
| Uniform size (bit count) | 15.47 | 5.98 | 72.4% |

With a uniform distribution, a < b half the time, so aligning almost always finished in 1 iteration (1 iteration in 66.5% of cases). Drawing sizes uniformly is probably closer to reality (a guess). The cost is hardware. Skipping zeros needs a leading-zero counter (LZC) and one shifter, and aligning needs two LZCs and a shifter.

There is a trap. With zero skipping, **if the divisor 0 is not blocked separately**, the result is wrong. In the skipped leading steps, R − 0 ≥ 0 means the quotient bits should have been 1, but because they were skipped, they stay 0. The quotient of `1000 ÷ 0` became `0x000003ff` (only the lower 10 bits are 1) instead of `0xffffffff`. Of 10,072 divide-by-zero cases, 5,033 were wrong. The property from section 4, that "the algorithm naturally produces the right value," breaks the moment you cut iterations. That is why cores with early termination detect a zero divisor and have a path that produces the spec value in 1 cycle. After adding this path, both early-termination models had zero mismatches on 300,000 pairs.

## 6. Pipeline Integration: Busy, Stalls, and Result Writeback

The simplest way to attach a multicycle unit to a 5-stage pipeline is this. When a multiply or divide instruction enters EX, give the unit a start signal, and the unit holds busy at 1 until it finishes. While busy, block writes to the IF and ID pipeline registers (stall) and insert a bubble into the EX/MEM register. When done arrives, load the result into the ALU result slot and send it to MEM. From then on, it is forwarded just like a normal ALU result and written in WB.

![Behind a mul that stays in EX for 4 cycles, add and sub stall in IF and ID for 3 cycles, and add receives the result forwarded from the MEM stage|full](/img/01M6K7XXTWKTP2N0FCF1DVQEFY/1952aa44ba84.webp)

The figure shows the results computed by the model's timeline function. Instructions behind a `mul` with latency L = 4 stall for L − 1 = 3 cycles. `sub` has nothing to do with `mul` yet stalled along with it. That is the cost of the simple approach. To stall only dependent instructions, later instructions must pass through EX while the unit runs, but then the multiply result and a later instruction's result may want the WB port in the same cycle, which calls for a scoreboard on rd and WB port arbitration. Cores that don't want to sit idle for 32 division cycles take this path.

There is one more thing that is easy to miss in integration. If an earlier instruction flushes the pipeline with a trap or branch misprediction while the unit is running, the unit must discard the operation it started (a kill signal). Conversely, if an operation already committed is discarded because of an interrupt, the instruction mepc points to runs again, so the result is the same but cycles are wasted (this is revisited in Part 27, the trap post).

## 7. mulh + mul Fusion: Values, Not Register Names

When both the upper and lower halves are needed, the spec recommends writing `mulh[[s]u] rdh, rs1, rs2` followed by `mul rdl, rs1, rs2`. If the source register order is the same and rdh differs from rs1 and rs2, the microarchitecture can fuse the two instructions into a single multiply (section 13.1). For division, `div[u] rdq, rs1, rs2` followed by `rem[u] rdr, rs1, rs2` (rdq ≠ rs1, rs2) plays the same role.

But here is what GCC 16.2 produced when compiling `(uint64_t)a * b` for rv32im.

```text
mv     a5,a1
mulhu  a1,a0,a1
mul    a0,a0,a5
```

Because `mulhu` writes its result to a1 (= rs2), `mul` reads a5, a copy made beforehand, instead of a1. The values are the same but the register names differ, so a fusion detector that checks the spec's recommended order by register name misses this pair. On the other hand, an approach where the multiplier remembers the previous operation's operand values and full 66-bit product, and answers immediately if the next multiply's operand values are the same (result reuse), catches it. Here are the cycles for running these three lines on the radix-2 model (33 cycles).

| Approach | Cycles |
|---|---|
| No fusion | 67 |
| Check spec order by register name | 67 |
| Compare previous operand values | 35 |

For the division pair, GCC emitted `divu a5,a0,a1` followed by `remu a0,a0,a1`, exactly in the spec's order. The divider holds both the quotient and remainder at once, so if it stores them together, `rem` finishes in 1 cycle. The value-comparison approach costs two 32-bit comparators and a few registers, but it doesn't need to separately track whether the operands changed (whether something wrote to the same register in between).

## 8. Hands-On: Bit-Exact Models vs. QEMU

The model is a single file (`model.py`). I compared 2 multiplier cores (radix-2, radix-4 Booth) and 4 divider cores (restoring, non-restoring, zero skipping, divisor alignment) against reference functions that translate the spec rules into Python integers. The inputs are all 5,184 pairs of 72 edge values (0, ±1, `0x7fffffff`, `0x80000000`, powers of two and those minus 1, etc.), plus 300,000 random pairs drawn from three distributions: uniform, uniform size, and small numbers (seed 19).

| Test | Count | Mismatches |
|---|---|---|
| 2 multiplier cores × 4 instructions | 305,184 pairs | 0 |
| 4 divider cores × 4 instructions (including zero handling) | 305,184 pairs | 0 |
| QEMU result = model = reference | 648 | 0 |

On the QEMU side, I ran the 8 instructions as inline assembly on 81 pairs of 9 edge values (0, 1, −1, 2, −7, 7, `0x7fffffff`, `0x80000000`, `0x12345678`) and printed them over UART (`muldiv.c`). The divide-by-zero and overflow lines were as follows.

```text
div  0xfffffff9, 0  → 0xffffffff
rem  0xfffffff9, 0  → 0xfffffff9
divu 0xfffffff9, 0  → 0xffffffff
div  0x80000000, -1 → 0x80000000
rem  0x80000000, -1 → 0x00000000
```

The setup was a Mac M1 Max, `riscv64-elf-gcc` 16.2, QEMU 11.1.3, and Python via `uv run`, and running the full model took about 40 seconds.

## 9. Design Checklist: The M Extension

| Item | Rule |
|---|---|
| Decode | OP + funct7 `0000001`. funct3[2]: 0 multiply, 1 divide |
| Shared multiplier | Inputs to 33 bits: sa = `mulh`/`mulhsu`, sb = `mulh`. Product[63:32] or [31:0] |
| Divide by zero | Quotient `0xffffffff`, remainder = rs1. No exception |
| Sign restore | If the divisor is 0, don't negate the quotient (a common bug) |
| Overflow | `0x80000000 ÷ -1` → quotient `0x80000000`, remainder 0. Correct automatically if absolute values are handled as 33-bit or unsigned |
| Remainder sign | The dividend's sign if nonzero |
| Early termination | Separate path for divisor 0. Skipped steps leave quotient bits at 0 |
| rd = x0 | Execute but discard the result. Same as the register file's x0 rule |
| Flush while busy | Kill the in-flight operation on an earlier instruction's trap or branch misprediction |
| Fusion (optional) | Comparing previous operand values fits GCC's code better than checking spec order |
| Zmmul | Multiply-only subset. Implement only the 4 multiplies, no divider |

What the spec leaves to the implementation is latency and structure. Whether an instruction takes 1 cycle or 33, only the result has to match. So even within RV32IM, multiply latency ranges from 1 to 33, and the compiler emits the same code without knowing.

## Summary

- A single 33×33 signed multiplier, with 1 bit added on top of each operand, handles `mul`, `mulh`, `mulhsu`, and `mulhu`. Radix-2 takes 33 cycles and radix-4 Booth takes 17.
- An unsigned iterative divider produces the spec's divide-by-zero values naturally. The signed wrapper needed the condition "don't negate the quotient if the divisor is 0" (without it, 5,039 of 10,072 mismatched). Overflow needed no special handling.
- Early termination cuts average iterations from 32 to 5.98, but without separate handling of a zero divisor, divide-by-zero results are wrong.
- The simplest integration stalls the front stages while busy (L − 1 cycles). For mulh+mul fusion, GCC renames registers, so the value-comparison approach cut 67 cycles to 35.

**Next post:** Part 20 implements the A extension. Using a Python model where two harts share memory, I check how large to make the LR/SC reservation set and when to clear it, the conditions under which SC must fail and the forward-progress guarantee the spec requires, whether to execute AMOs in the core's ALU or on the memory side, and how an in-order core honors aq/rl and FENCE.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Chapter 13, "M" extension (13.1 multiplication, 13.2 division and Table 11, 13.3 Zmmul). https://github.com/riscv/riscv-isa-manual
- D. Harris et al., *RISC-V System-on-Chip Design*, 2023. Multiply and divide unit implementation (CORE-V Wally)
- D. Harris, S. Harris, *Digital Design and Computer Architecture: RISC-V Edition*, 2021. Section 5.2, multipliers and dividers
- D. Patterson, J. Hennessy, *Computer Organization and Design: RISC-V Edition*. Sections 3.3–3.4, multiplication and division hardware
- A. D. Booth, "A Signed Binary Multiplication Technique", *Quarterly Journal of Mechanics and Applied Mathematics*, 1951
