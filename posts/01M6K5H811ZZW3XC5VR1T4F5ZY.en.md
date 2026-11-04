---
id: "01M6K5H811ZZW3XC5VR1T4F5ZY"
title_en: "RV32I 2: Integer Ops, Compares, Shifts, and Building Constants with lui/auipc"
summary_en: "Part 3 of the RISC-V series. Checks RV32I integer arithmetic, compares, shifts, and flagless overflow handling on QEMU with 15 edge cases, then examines the +1 correction for building 32-bit constants and addresses with lui and auipc through 1 million random trials."
---
:::note
**The 3 key points of this post**
- RV32I's integer operations are 10 register-register instructions (`add` `sub` `and` `or` `xor` `sll` `srl` `sra` `slt` `sltu`), 9 immediate versions, and `lui`/`auipc`. That's all. There is no overflow flag and no exception, and instructions like `mv`, `not`, `neg`, and `seqz` are combinations of these (pseudo-instructions).
- A 32-bit constant is built with `lui` (upper 20 bits) and `addi` (lower 12 bits). Because the `addi` immediate is sign-extended, if the top bit of the lower 12 bits is 1, the `lui` part must be incremented by 1. Of 1 million random constants, 499,908 needed this correction.
- I actually ran 15 edge cases on QEMU and confirmed that all of them match a Python reference model. The numbers point out places that are easy to get wrong in a design, such as the `sltiu` immediate being sign-extended and `sll` looking only at the lower 5 bits of the shift amount.
:::

Part 2 looked at the shape of instructions (the six formats and immediate encoding). Part 3 covers what the integer operations inside those shapes actually compute. Nearly half of the RV32I instructions belong here, and this list is exactly what the ALU (arithmetic logic unit) has to do in a core design. At the end, we walk through by hand how to build 32-bit constants and addresses with `lui` and `auipc`.

From this part on, I verify instructions by actually running them. I loaded RISC-V programs without an operating system (bare metal) onto the `virt` board of QEMU 11.1.3 and printed the results over UART. The machine is a Mac M1 Max, and the compiler is `riscv64-elf-gcc` 16.2.

## 1. Integer operations at a glance

| Kind | Register-register (R format) | Immediate (I format) | Computation |
|---|---|---|---|
| Add/subtract | `add`, `sub` | `addi` | rs1 + rs2, rs1 − rs2 |
| Logical | `and`, `or`, `xor` | `andi`, `ori`, `xori` | Bitwise operation |
| Shift | `sll`, `srl`, `sra` | `slli`, `srli`, `srai` | Left, right (zero fill), right (sign fill) |
| Compare | `slt`, `sltu` | `slti`, `sltiu` | 1 if rs1 < rs2, otherwise 0 |
| Upper immediate | | `lui`, `auipc` (U format) | imm << 12, pc + (imm << 12) |

Something is conspicuously missing: there is no `subi`, because you can just give `addi` a negative number. The immediate is always sign-extended as a 12-bit signed value (−2048 to 2047). So keep in mind that the immediates of `andi`, `ori`, and `xori` are sign-extended too. Section 4 shows where this becomes a trap.

Many commonly used operations are not instructions but **pseudo-instructions**. The assembler turns each into a single real instruction.

| Pseudo-instruction | Actual instruction | Meaning |
|---|---|---|
| `nop` | `addi x0, x0, 0` | Do nothing |
| `mv rd, rs` | `addi rd, rs, 0` | Copy |
| `not rd, rs` | `xori rd, rs, -1` | Bitwise invert |
| `neg rd, rs` | `sub rd, x0, rs` | Negate |
| `seqz rd, rs` | `sltiu rd, rs, 1` | 1 if rs == 0 |
| `snez rd, rs` | `sltu rd, x0, rs` | 1 if rs ≠ 0 |
| `sltz rd, rs` | `slt rd, rs, x0` | 1 if rs < 0 |
| `sgtz rd, rs` | `slt rd, x0, rs` | 1 if rs > 0 |

The `x0`=0 rule from part 2 shines here. Eight useful operations were gained without adding a single instruction to the hardware.

## 2. Overflow: no flags, no exceptions

x86 and ARM look at the result of an addition and set flags such as carry and overflow. RISC-V has no such flags, and it raises no exception on overflow. The result is simply truncated to 32 bits (it wraps around in two's complement).

| Operation | rs1 | rs2 | Result | Explanation |
|---|---|---|---|---|
| `add` | `0x7fffffff` | `1` | `0x80000000` | Largest positive + 1 = smallest negative |
| `sub` | `0x80000000` | `1` | `0x7fffffff` | Smallest negative − 1 = largest positive |

Without flags, what does software that needs overflow detection (big-integer arithmetic, languages that check for overflow) do? The specification gives idioms that check it with a few instructions.

- **Unsigned addition:** `add t0, t1, t2` followed by `bltu t0, t1, overflow`. If the result is smaller than an operand, it overflowed.
- **Signed addition (general):** `add t0, t1, t2`, `slti t3, t2, 0`, `slt t4, t0, t1`, `bne t3, t4, overflow`. If "is the added value negative" differs from "did the result decrease," it overflowed.

Let's apply the last idiom to `0x7fffffff + 1`. t2=1 is not negative, so t3=0. The result `0x80000000` is −2,147,483,648 as a signed number, which is less than t1=2,147,483,647, so t4=1. They differ, so it is an overflow. QEMU also gave 1.

From the design side, this decision is a big simplification. With no flag register, there is no need to track hidden dependencies between instructions (a later instruction reading an earlier instruction's flags), and since integer operations never raise exceptions, the trap path can be left out of the ALU.

## 3. Compare: `slt` and `sltu`

`slt` (set less than) compares two values as signed numbers, and `sltu` compares them as unsigned numbers. The same bits can give opposite answers depending on how they are read.

| Operation | rs1 | rs2 | Result | Reason |
|---|---|---|---|---|
| `slt` | `0xffffffff` | `1` | `1` | Read as signed, −1 < 1 |
| `sltu` | `0xffffffff` | `1` | `0` | Read as unsigned, 4,294,967,295 < 1 is false |
| `sltiu` | `5` | immediate `-1` | `1` | The immediate is sign-extended to `0xffffffff`, then compared as unsigned |
| `sltiu` | `0` | immediate `1` | `1` | `seqz`: 0 < 1 |

The third row matters. `sltiu` is an "unsigned" compare, but its immediate, like other I formats, is **sign-extended first**. The immediate −1 becomes `0xffffffff` and is compared as "the largest unsigned number." So `sltiu rd, rs, -1` is 0 only when rs is `0xffffffff`. If you build the hardware thinking "it's an unsigned instruction, so zero-extend the immediate," you get it wrong.

There are two ways to build `slt` in hardware: look at the sign of the subtraction result while correcting for overflow, or compare the sign bits first.

$$\text{slt}(a, b) = \begin{cases} a_{31} & a_{31} \ne b_{31} \\ (a - b)_{31} & a_{31} = b_{31} \end{cases}$$

If the signs differ, the negative one is smaller, so a's sign bit is the answer. If the signs are the same, the subtraction cannot overflow, so the sign bit of the subtraction result is the answer. For `sltu`, do the subtraction in 33 bits and check whether a borrow occurs. Both share the adder, so there is no need for a separate comparator.

## 4. Shifts and logical operations

There are three shifts. `sll` shifts left and fills the vacated bits with 0, `srl` shifts right and fills with 0, and `sra` shifts right and fills with the sign bit.

| Operation | rs1 | Shift amount | Result | Explanation |
|---|---|---|---|---|
| `srai` | `0x80000f00` | 8 | `0xff80000f` | Upper bits filled with the sign (1) |
| `srli` | `0x80000f00` | 8 | `0x0080000f` | Upper bits filled with 0 |
| `sra` | `0x80000000` | 31 | `0xffffffff` | |
| `srl` | `0x80000000` | 31 | `0x00000001` | |
| `sll` | `1` | 33 | `0x00000002` | Only the lower 5 bits of the shift amount are used (33 → 1) |

Look at the last row. Register shifts use only the **lower 5 bits** of rs2 as the shift amount. 33 is `100001` in binary, so its lower 5 bits are 1. That's why 1 was shifted once, not 33 times, giving 2. In C, shifting by 32 or more is undefined behavior, but RISC-V hardware gives this well-defined answer.

`srai` and `srli` share the same opcode and funct3. A single bit, inst[30], tells them apart (the funct7 position in the part 2 figure, the same bit as for `sub` and `sra`). Immediate shifts are I format, but the shift amount uses only the 5 bits of inst[24:20], and the field above is used like funct7.

Logical operations again call for care with immediate sign extension.

| Operation | rs1 | Immediate | Result |
|---|---|---|---|
| `xori` | `0x0000f0f0` | `-1` (`0xffffffff`) | `0xffff0f0f` |
| `andi` | `0x12345678` | `0xff` | `0x00000078` |
| `andi` | `0x12345678` | `-2048` (`0xfffff800`) | `0x12345000` |

As in the third row, `andi rd, rs, -2048` clears only the lower 11 bits, not the lower 12. That's because the immediate `0x800` is sign-extended to `0xfffff800`. Clearing the lower 12 bits would need `0xfffff000`, which a 12-bit immediate cannot represent. In that case, shift out and back with `srli`/`slli`, or build the mask with `lui`.

## 5. `lui` and 32-bit constants

Instructions are 32 bits, so a 32-bit constant cannot fit in a single instruction. RISC-V splits the constant into its upper 20 bits and lower 12 bits and builds it with two instructions. `lui rd, imm` (load upper immediate) places a 20-bit immediate in the upper bits and fills the lower 12 bits with 0. Then `addi` adds the lower 12 bits.

![A 32-bit constant is built with lui for the upper 20 bits and addi for the lower 12 bits; if bit 11 of the lower 12 bits is 1, the lui value is incremented by 1 in advance|full](/img/01M6K5H811ZZW3XC5VR1T4F5ZY/da4ee4436943.webp)

The problem is that the `addi` immediate is signed. If you try to build `0x12345fff` with `lui a0, 0x12345` and `addi a0, a0, 0xfff`, `0xfff` is read as −1 and you get `0x12344fff`. So the upper 20 bits are incremented to `0x12346`, and adding −1 brings it back. The general formula is:

$$hi = \left\lfloor \frac{c + \texttt{0x800}}{2^{12}} \right\rfloor, \qquad lo = c - hi \cdot 2^{12}$$

Adding `0x800` to c means "if the lower 12 bits are `0x800` or more, carry 1 upward." Then lo always falls within −2048 to 2047. The assembler's `%hi(c)` and `%lo(c)` perform this calculation.

I applied this formula to 1 million random 32-bit constants and 7 edge cases. Recombining hi and lo always gave back the original value, and lo was always in range. Of these, 499,908 (about half) received the +1 correction. That is expected, since the top bit of the lower 12 bits is 1 half the time.

The actual assembler expanded the `li` (load immediate) pseudo-instruction like this.

| `li a0, …` | Expansion | Instruction count |
|---|---|---|
| `100`, `2047`, `-2048`, `-1` | `addi a0, x0, …` | 1 |
| `0x12345000` | `lui a0, 0x12345` | 1 |
| `2048` (`0x800`) | `lui a0, 0x1` + `addi a0, a0, -2048` | 2 |
| `0x12345678` | `lui a0, 0x12345` + `addi a0, a0, 1656` | 2 |
| `0x12345fff` | `lui a0, 0x12346` + `addi a0, a0, -1` | 2 |
| `0x7ffff800` | `lui a0, 0x80000` + `addi a0, a0, -2048` | 2 |
| `0xfffff800` | `addi a0, x0, -2048` | 1 |

2048 is exactly 1 past the 12-bit signed range, so it needs two instructions. Conversely, `0xfffff800` is −2048, so a single `addi` does it. For `0x7ffff800`, `lui` takes `0x80000` to make `0x80000000` (a negative number), then −2048 is added. This makes direct use of 32-bit wraparound.

## 6. `auipc`: pc-relative addresses

`auipc rd, imm` (add upper immediate to pc) shifts a 20-bit immediate up and adds it to **the address of that `auipc` instruction itself**. It builds an address from the distance to the current location instead of from a constant.

![la sp, _stack_top run on QEMU: auipc sp, 0x4 adds 0x4000 to pc 0x80000000, and addi adds 0x1d0 to make 0x800041d0|full](/img/01M6K5H811ZZW3XC5VR1T4F5ZY/17f53b677391.webp)

The `la sp, _stack_top` in the startup code run on QEMU is a good example. The assembler expanded it into `auipc sp, 0x4` and `addi sp, sp, 464`. The `auipc` at 0x80000000 adds `0x4 << 12 = 0x4000` to get `0x80004000`, and then `addi` adds 464 (`0x1d0`) to give the stack top `0x800041d0`. The machine code of these two instructions contains no absolute address. Even if the whole program is loaded at a different address, the same machine code finds the same relative location. This is the foundation of position-independent code (PIC).

I checked inside `main` as well. The pc read with `auipc` was `0x80000020`, and the address of the global variable `target` obtained with `la` and C's `&target` were both `0x800001c0` (a distance of `0x1a0`).

The partner of `auipc` depends on the situation. If you need an address, `addi` adds the lower 12 bits; to read the value at that address, `lw`; to call a distant function, `jalr` (`call` = `auipc` + `jalr`). With this combination you can reach anywhere within ±2 GiB of the pc. The lower 12 bits of the partner instruction are also sign-extended, so the +1 correction from section 5 applies here in exactly the same way (the assembler's `%pcrel_hi` and `%pcrel_lo`).

## 7. Hands-on: QEMU results vs. a reference model

Every number in the tables above is a value actually executed on QEMU. I put one instruction at a time into C with inline assembly and printed the result over UART.

```c
#define R(op, a, b) ({ uint32_t r_; \
    __asm__ volatile(op " %0,%1,%2" : "=r"(r_) : "r"(a), "r"(b)); r_; })
#define I(op, a, imm) ({ uint32_t r_; \
    __asm__ volatile(op " %0,%1," #imm : "=r"(r_) : "r"(a)); r_; })

show("sltu", m1, one, R("sltu", m1, one));      /* 0xffffffff < 1 ? */
show("sll", one, s33, R("sll", one, s33));      /* shift amount 33 */
show("sltiu", 5, -1, I("sltiu", 5, -1));        /* immediate sign extension */
```

To keep the compiler from computing the constants ahead of time, the inputs were placed in `volatile` variables. The output was compared against a Python reference model (a few lines that write out the 32-bit mask and the signed interpretation directly).

| Test | Count | Matches |
|---|---|---|
| QEMU results = Python reference model | 15 | 15 |
| Constant split formula (recombining hi/lo, lo range) | 1,000,007 | 1,000,007 |

On the first run, 1 of the 15 was wrong. It was my own mistake: I put `0xff` as the `andi` immediate in the test code but wrote `0xfff` in the table. Without comparing against a reference model, a wrong number would have ended up in the post.

## 8. Design checklist: the ALU and its surroundings

| Item | Rule |
|---|---|
| ALU operations | 10: add, sub, and, or, xor, sll, srl, sra, slt, sltu. Selected by funct3 and inst[30] |
| `addi` and inst[30] | In OP-IMM, inst[30] selects the operation only for `srai`. Do not treat `addi` as a subtraction just because bit 10 of its immediate is 1 (a common bug) |
| Immediate extension | The immediates of `sltiu`, `andi`, `ori`, and `xori` are also sign-extended |
| Shift amount | Register shifts use only rs2[4:0]. For immediate shifts, inst[25]=1 is reserved on RV32 (illegal, part 2) |
| `slt`/`sltu` | Share the adder. slt is a[31] if the signs differ, (a−b)[31] if they are the same |
| Overflow | No flags, no exceptions. The ALU needs no trap path |
| `lui` | Result = immediate (lower 12 bits 0). Handling it in the ALU as "0 + imm" reuses the adder |
| `auipc` | Result = that instruction's pc + imm. A selector that feeds pc into the ALU's first input is needed |
| rd = x0 | The result is discarded. No exception (HINT) |

The `auipc` row affects datapath design. Most instructions feed rs1 into the ALU's first input, but `auipc` (and the `jal` and branch address calculations in part 4) feed in the pc. That's why even a single-cycle core gets an "ALU input A = rs1 or pc" selector.

## Summary

- RV32I's integer operations are 10 register-register instructions, 9 immediate instructions, and `lui`/`auipc`. That's all. Operations like `mv`, `not`, `neg`, and `seqz` are pseudo-instructions that make use of `x0`.
- There are no overflow flags or exceptions. When needed, software checks with `slt`/`bltu`, and the hardware sheds flag dependencies and the trap path.
- The immediates of `sltiu`, `andi`, and others are also sign-extended, and register shifts use only the lower 5 bits. All 15 QEMU runs matched the reference model.
- 32-bit constants and addresses are built with `lui`/`auipc` (upper 20 bits) and a partner instruction (lower 12 bits), and if bit 11 of the lower 12 bits is 1, the upper part is incremented by 1. Of 1 million random constants, 499,908 received the correction.

**Next post:** Part 4 covers the rest of RV32I: branches, jumps, and memory access. It runs on QEMU the six branches `beq` through `bgeu`, how function calls and returns are built with `jal` and `jalr`, sign extension of byte and halfword loads, how misaligned accesses are handled, and `fence`, `ecall`, and `ebreak`.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Section 2.4, Integer Computational Instructions (including the overflow-check idioms). https://github.com/riscv/riscv-isa-manual
- RISC-V Assembly Programmer's Manual (pseudo-instructions, `%hi`, `%lo`, `%pcrel_hi`). https://github.com/riscv-non-isa/riscv-asm-manual
- D. Patterson, A. Waterman, *The RISC-V Reader: An Open Architecture Atlas*, 2017. Chapter 2, RV32I; Chapter 3, Assembly Language
- D. Harris, S. Harris, *Digital Design and Computer Architecture: RISC-V Edition*, 2021. Section 5.2, Arithmetic Circuits (adders, comparators, shifters)
- QEMU `virt` board documentation. https://www.qemu.org/docs/master/system/riscv/virt.html
