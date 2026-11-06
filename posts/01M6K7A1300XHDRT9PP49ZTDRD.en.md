---
id: "01M6K7A1300XHDRT9PP49ZTDRD"
title_en: "RV64I: W Instructions, Sign-Extension Rules, and What Differs from 32-Bit"
summary_en: "Part 5 of the RISC-V series. I checked RV64I's W instructions and sign-extension rules on QEMU. 0x7fffffff+1 became 0xffffffff80000000 with addw, and 64-bit constants took an average of 7.887 instructions with li. I also look at why utoa10 from part 1 grew."
---
:::note
**Key points in three lines**
- RV64I widens the registers to 64 bits and adds 12 instructions (`ld`, `sd`, `lwu`, and 9 W instructions). A W instruction computes using only the lower 32 bits and then copies bit 31 into the upper 32 bits. On QEMU, `0x7fffffff + 1` became `0x0000000080000000` with `add` and `0xffffffff80000000` with `addw`.
- The purpose of this rule is a convention: "a 32-bit value always sits in a register in sign-extended form." That is why C's `int`→`long` conversion, comparisons, and logical operations need no extra instructions, and only addition and shifts have W versions.
- `lui` also sign-extends its result, so 64-bit constants are expensive. Expanding 5,000 random 64-bit constants with `li` took an average of 7.887 instructions, 8 at most. A 32-bit constant always took 2. All 34 execution and encoding results matched the reference model.
:::

Parts 2 through 4 covered all 40 RV32I instructions. Part 5 is RV64I, the same ISA widened to 64 bits. Nearly every application processor that runs Linux is RV64, so if I later build an RV64GC core, the rules in this post will show up all over the datapath. The RV64I chapter of the specification (Chapter 4) only describes how it differs from RV32I. This post does the same: what gets wider, what gets added, and why it was designed that way.

The experiments ran bare-metal on the `virt` board of `qemu-system-riscv64` (11.1.3). I compiled with `riscv64-elf-gcc` 16.2 using `-march=rv64i -mabi=lp64`.

## 1. What Gets Wider and What Gets Added

XLEN becomes 64. Registers `x1`–`x31` and `pc` are 64 bits, and so is the address space. RV32I instructions keep their names and encodings, but they operate on 64 bits. For example, `add` is a 64-bit addition and `slt` is a 64-bit comparison.

There are 12 new instructions.

| Kind | Instructions | What they do |
|---|---|---|
| 64-bit memory | `ld`, `sd` | 8-byte load and store |
| 32-bit load | `lwu` | Zero-extends 4 bytes (`lw` sign-extends) |
| W immediate | `addiw`, `slliw`, `srliw`, `sraiw` | Compute in 32 bits, then sign-extend |
| W register | `addw`, `subw`, `sllw`, `srlw`, `sraw` | Compute in 32 bits, then sign-extend |

W instructions use two opcodes that RV32I did not have (OP-IMM-32 `0011011`, OP-32 `0111011`). `ld` and `sd` use funct3 `011`, and `lwu` uses `110`. Part 4's rule, "funct3[1:0] is the size, funct3[2] means zero-extend," carries straight over. Size `11` simply became 8 bytes.

What deserves attention are the instructions that **don't** exist. There is no `andw`, `orw`, or `xorw`, no `sltw`, and no branch like `beqw`. The reason lies in the convention in section 2.

## 2. W Instructions and the "Keep It Sign-Extended" Convention

The specification defines W instructions this way: **ignore** the upper 32 bits of the inputs, compute a 32-bit result from the lower 32 bits, **sign-extend** it to 64 bits, and write it to rd. Bits 63 through 31 of the result all end up equal.

![On QEMU rv64, 0x7fffffff+1 becomes 0x0000000080000000 with add and 0xffffffff80000000 with addw, and li 0x7fffffff produces the right value with lui 0x80000 followed by addiw -1|full](/img/01M6K7A1300XHDRT9PP49ZTDRD/56f4ffbaa1db.webp)

Here are the actual results run on QEMU.

```text
op     rs1               rs2  result
add    000000007fffffff  1    0000000080000000
addw   000000007fffffff  1    ffffffff80000000
addw   1234567800000001  1    0000000000000002
subw   0000000000000000  1    ffffffffffffffff
addiw  1234567880000000  0    ffffffff80000000
```

If you think in terms of C's `int`, `addw` gives the right answer. For an `int`, `0x7fffffff + 1` wraps around in 32 bits to −2,147,483,648, and in a 64-bit register that value looks like `0xffffffff80000000`. The third line shows that `addw` ignores garbage (`0x12345678`) in the upper 32 bits. `addiw rd, rs, 0` becomes an instruction that only sign-extends the lower 32 bits, and it has its own pseudo-instruction name, `sext.w`.

### Why Sign Extension

The key is in the specification's commentary. The compiler and calling convention **always keep 32-bit values in 64-bit registers in sign-extended form.** Even unsigned 32-bit integers (`unsigned`) have bit 31 copied upward. This one convention makes a lot of things free.

- `int` → `long` conversion: the value is already sign-extended, so no instruction is needed.
- Conversion between `int` and `unsigned`: the register looks the same, so no instruction is needed.
- Comparisons and branches: comparing two sign-extended 32-bit values with 64-bit `slt`/`blt` gives the same answer. The same goes for unsigned comparisons (`sltu`/`bltu`), because the upper 32 bits are filled by the same rule, so ordering is preserved.
- Logical operations: `and`/`or`/`xor` on sign-extended values produce a sign-extended result.

Addition, subtraction, and shifts, on the other hand, push bits that overflow 32 bits above bit 31 and break the convention. So only these have W versions. In the specification's words, the W instructions add only the few needed to get reasonable performance on 32-bit values.

I confirmed this with actual compiler output (`-O2`).

| C function | RV64 instructions |
|---|---|
| `int add(int a, int b)` | `addw a0,a0,a1` |
| `unsigned add(unsigned, unsigned)` | `addw a0,a0,a1` |
| `long i2l(int a)` | None (just `ret`) |
| `int lt(int a, int b)` | `slt a0,a0,a1` (plain 64-bit comparison) |
| `unsigned long u2ul(unsigned a)` | `slli a0,a0,32` + `srli a0,a0,32` |

The last row is the cost of this convention. Turning an `unsigned` into a 64-bit `unsigned long` means clearing the upper 32 bits to 0, and RV64I has no instruction for that, so it takes two shifts. The same thing happens when indexing an array with an `unsigned` index (`slli` 32 + `srli` 29 does the zero extension and the ×8 at once). The Zba extension includes `zext.w` (= `add.uw rd, rs, zero`) to cut this cost.

### Why `utoa10` from Part 1 Grew on RV64

In part 1, `utoa10` (converting an unsigned 32-bit value to a decimal string) got larger on RV64. This time I recompiled it without the C extension, as `rv32im` and `rv64im`, and counted instructions: 33 versus 39. The difference came from three places.

- The constant `0xcccccccd`, used to replace division with multiplication: 2 instructions on RV32 (`lui`+`addi`), 4 on RV64. This value has bit 31 set, so it cannot be built as a sign-extended 32-bit constant (section 4).
- Zero-extending `uint32_t v` before feeding it into a 64-bit multiply: 2 instructions (`slli`+`srli`). With Zba enabled this became a single `zext.w`, bringing the count down to 38.
- The remaining 2 came from differences in register allocation and loop shape.

On the other hand, 32-bit operations like `n++`, `v - q*10`, and `'0' + r` finished with 6 W instructions (2 `addiw`, 2 `slliw`, `addw`, `subw`), the same count as on RV32. Without W instructions, each of these operations would have needed an extra sign extension (`sext.w`) (my estimate). W instructions keep the growth of RV32 code ported to RV64 to about this much.

## 3. Shift Amounts: 6 Bits and 5 Bits

A 64-bit shift must be able to shift by 0–63, so the shift amount becomes 6 bits. W shifts shift a 32-bit value, so they stay at 5 bits.

```text
op     rs1               amt result
sllw   0000000000000001  31  ffffffff80000000
sllw   0000000000000001  32  0000000000000001
sll    0000000000000001  32  0000000100000000
sll    0000000000000001  64  0000000000000001
srliw  ffffffff80000000  4   0000000008000000
srli   ffffffff80000000  4   0ffffffff8000000
sraiw  ffffffff80000000  4   fffffffff8000000
```

Shifting 1 left by 31 with `sllw` sets bit 31, so everything above it fills with 1s. Shifting by 32 only looks at the lower 5 bits (0), so the value stays 1. `sll` looks at 6 bits, so 32 really means 32, and 64 means 0. Comparing the three right-shift lines makes the meaning of W clear. `srliw` shifts only the lower 32 bits `0x80000000`, filling with 0 (`0x08000000`), and since bit 31 of the result is 0, the upper half is 0 too. The 64-bit `srli` drags the 1s from the upper 32 bits down along with it.

In the encoding, the position that was "reserved if inst[25]=1" on RV32 becomes shamt[5] in RV64's `slli`, `srli`, and `srai` (part 2). Conversely, encodings of `slliw`, `srliw`, and `sraiw` with imm[5] set to 1 are reserved. If a decoder supports both RV32 and RV64, the meaning of this one bit changes with XLEN.

## 4. Loads, Sign Extension in `lui`/`auipc`, and 64-Bit Constants

I placed the word `0x80000000` in memory, followed by the word `0x00000001`, and read them back. `lw` gave `ffffffff80000000`, `lwu` gave `0000000080000000`, and `ld` combined the two words into `0000000180000000`.

`lui` and `auipc` also change on RV64. They **sign-extend to 64 bits the 32-bit value** formed by shifting the 20-bit immediate left by 12.

| Instruction | Result | Explanation |
|---|---|---|
| `lui 0x80000` | `ffffffff80000000` | Bit 31 is 1 → upper bits are 1 |
| `lui 0x7ffff` | `000000007ffff000` | Bit 31 is 0 → upper bits are 0 |
| `auipc 0xfffff` | pc − `0x1000` | pc `0x80000398` → `0x7ffff398` |
| `auipc 0x80000` | pc − 2 GiB | pc `0x800003b8` → `0x000003b8` |

Because `auipc` sign-extends, pc-relative addressing on RV64 still reaches ±2 GiB in each direction. The fourth row pointed exactly to `0x3b8`, 2 GiB below `0x800003b8`.

### `li 0x7fffffff` and `addiw`

On RV64, building `0x7fffffff` works differently from RV32 in part 3. The assembler chose `lui a0, 0x80000` and `addiw a0, a0, -1`. `lui` produces `0xffffffff80000000`, and `addiw` adds −1 in 32 bits, wrapping around to `0x7fffffff`, then sign-extends. Bit 31 is 0, so the upper half becomes 0, giving the correct `0x000000007fffffff`. Using a 64-bit `addi` here would have produced `0xffffffff7fffffff`. That is why RV64 constant generation needs `addiw`.

For the same reason, a positive value with bit 31 set, such as `0x80000000`, cannot be built with a single `lui`. The assembler built it with `addiw a0, zero, 1` and `slli a0, a0, 31`.

### How Many Instructions Does a 64-Bit Constant Take

I collected what `li` expanded into (`-march=rv64i`, no C).

```text
0x7ff                addi
0x800                lui + addiw
0x7fffffff           lui + addiw
0x80000000           addiw + slli
0xffffffff           addiw + slli + addi
0xcccccccd           lui + addiw + slli + addi
0x8000000000000000   addiw + slli
0x123456789abcdef0   lui + addiw + (slli + addi) x3
```

The rule goes like this. Values within the signed 32-bit range finish with `lui` + `addiw` (2 at most), as in part 3. For anything else, the upper part is built first, then "shift left by about 12 bits and add the next 12 bits" is repeated. If the low end has many zeros, a single shift finishes it; values with every bit populated go up to 8 instructions.

![Instruction counts from li expansion: constants of 12 bits or less always took 1, 32-bit constants always took 2, and random 64-bit constants averaged 7.887 (4,718 took 8)|xl](/img/01M6K7A1300XHDRT9PP49ZTDRD/3dc500e439da.webp)

I drew 5,000 random values for each kind of constant and assembled them. Every value of 12 bits or less took 1 instruction, and every signed 32-bit value took 2. Of the random 64-bit values, 4,718 took 8, 281 took 6, and 1 took 7, for an average of 7.887. I re-executed the expanded instructions in Python and confirmed that all 15,000 produced the original constant.

The compiler takes a different approach. GCC `-O2` built `0x123456789abcdef0` by constructing the upper and lower halves **separately in two registers**, then combining them with `slli` 35 and `add`. That is 7 instructions, but the two chains don't wait on each other, so they finish quickly on a superscalar core. For `0x1234567812345678`, where the upper and lower 32 bits are identical, it built the half once and finished in 4 instructions with `slli` 32 + `add`. Still, 64-bit constants are expensive, so large values used often, such as addresses, are better produced as `auipc`-relative addresses or loaded from a constant pool in memory.

## 5. Experiment: QEMU Results Checked Against a Reference Model

I ran each W instruction one at a time through inline assembly and printed the 64-bit result.

```c
#define R(op, a, b) ({ uint64_t r_;            \
  __asm__ volatile(op " %0,%1,%2"              \
                   : "=r"(r_) : "r"(a), "r"(b)); r_; })

volatile uint64_t big = 0x7fffffff, one = 1;
show("add",  big, one, R("add",  big, one));
show("addw", big, one, R("addw", big, one));
```

`run.sh` picks `qemu-system-riscv64` when `-march` starts with `rv64`. The reference model is a few lines of Python. A W operation is "compute from the lower 32 bits → truncate to 32 bits → sign-extend."

```python
sx = lambda v, b: ((v & ((1 << b) - 1))
                   ^ (1 << (b - 1))) - (1 << (b - 1))
w = lambda v: sx(v, 32) & M64      # 32-bit result → 64-bit
REF['addw'] = lambda a, b: w(a + b)
REF['srlw'] = lambda a, b: w((a & M32) >> (b & 31))
REF['lui']  = lambda a, b: sx(a << 12, 32) & M64
```

| Test | Count | Matched |
|---|---|---|
| QEMU execution (W, shifts, loads, `lui`, `auipc`) | 21 | 21 |
| Re-running the 10 `li` examples to get the original value | 10 | 10 |
| 3 batches of random constants (5,000 each) | 3 | 3 |

34 in total, all matched. Had I first written the `srlw` reference model as `w(a >> (b & 31))`, the upper 32 bits would have shifted down into the result and it would have been wrong. You have to translate the specification's "ignore the upper 32 bits of the inputs" literally and apply `a & M32` first. The same holds for hardware.

## 6. Design Checklist: The RV64 Datapath

| Item | Rule |
|---|---|
| W result | Copy bit 31 of the lower 32-bit result into bits 63:32 |
| W inputs | Ignore bits 63:32 of rs1 and rs2. In particular, `srlw`/`sraw` shift only the lower 32 bits |
| Shift amount | rs2[5:0] for 64-bit, rs2[4:0] for W |
| Immediate shifts | For `slli` and friends, inst[25] = shamt[5]. For `slliw` and friends, imm[5]=1 is reserved |
| `lui`/`auipc` | Sign-extend the 32-bit value. `auipc` then adds it to the 64-bit pc |
| `lw`/`lwu`/`ld` | Sign-extend / zero-extend / as is. funct3 `010` / `110` / `011` |
| Missing W | Logical ops, comparisons, and branches use the 64-bit versions as is |
| ALU structure | Handle W with one mux that sign-extends the lower 32 bits of the 64-bit adder result (a common choice) |
| W shifter | `sraw` must use bit 31 as the sign. A common approach is to sign- or zero-extend the input before feeding it to the 64-bit shifter |
| RV32 mode | If an RV64 core supports a 32-bit mode via `mstatus.UXL` and similar, additional result sign-extension rules apply (optional feature) |

Let me unpack the ALU structure row. For `addw`, you can do the full 64-bit addition and then pick just the lower 32 bits of the result and sign-extend them, because garbage in the upper 32 bits of the inputs doesn't change the sum of the lower 32 bits. Shifts are different. For `srlw`/`sraw`, the upper 32 bits must not shift down into the result, so the input must be zero-extended (`srlw`) or sign-extended (`sraw`) before it goes into the shifter. The `srliw` result `0x0000000008000000` from section 3 is the correct answer for this handling.

## Summary

- RV64I widens XLEN to 64 and adds `ld`, `sd`, `lwu`, and 9 W instructions. W instructions ignore the upper 32 bits of their inputs and sign-extend the 32-bit result (`0x7fffffff + 1` → `addw` `0xffffffff80000000`).
- Thanks to the convention that 32-bit values are always kept sign-extended, `int`→`long` conversion, comparisons, and logical operations need no instructions. Only `unsigned`→`unsigned long` costs a zero extension (`slli`+`srli`, or Zba's `zext.w`).
- The shift amount is 6 bits for 64-bit shifts and 5 bits for W. `lui`/`auipc` sign-extend a 32-bit value, so `li 0x7fffffff` needs `addiw`.
- 64-bit constants averaged 7.887 instructions with `li` (5,000 random), and 32-bit constants always took 2. The growth of part 1's `utoa10` from 33 to 39 instructions on RV64 was also due to constant generation and zero extension.

**Next post:** Part 6 covers assembly and the ABI. By compiling real C functions, it traces what register names like `a0`–`a7` and `s0`–`s11` promise under the calling convention, and how stack frames, prologues, and epilogues come about. It also uses readelf and objdump to look at what happens with more than 8 arguments and when returning a struct, the differences between ABI names like `ilp32` and `lp64d`, how pseudo-instructions like `call`, `tail`, and `la` expand, and ELF relocations and linker relaxation.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Chapter 4 RV64I (4.2 Integer Computational Instructions, 4.3 Load and Store Instructions), 2.3 Immediate Encoding Variants. https://github.com/riscv/riscv-isa-manual
- RISC-V Bit-Manipulation ISA-extensions (Zba's `add.uw` and `zext.w`), Version 1.0.0
- RISC-V Assembly Programmer's Manual (`li`, `sext.w`). https://github.com/riscv-non-isa/riscv-asm-manual
- RISC-V ELF psABI (sign-extension rule for LP64 integer arguments). https://github.com/riscv-non-isa/riscv-elf-psabi-doc
- D. Patterson, A. Waterman, *The RISC-V Reader: An Open Architecture Atlas*, 2017. Chapter 9 RV64
