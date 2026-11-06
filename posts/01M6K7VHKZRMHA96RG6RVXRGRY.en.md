---
id: "01M6K7VHKZRMHA96RG6RVXRGRY"
title_en: "The C Extension: Compressed Formats and Their 32-bit Equivalents"
summary_en: "Part 11 of the RISC-V series. Covers the nine 16-bit formats of the C extension and their 32-bit equivalents, then checks a hand-written decompressor against objdump for all 49,152 16-bit values. In real code, 51–69% of instructions became 16-bit, shrinking code by 26–34%."
---
:::note
**Key points in three lines**
- The C extension's 16-bit instructions are not new operations but shorthand for frequently used 32-bit instructions. Each one expands to exactly one 32-bit instruction, so hardware only needs a decompressor in front of the decoder and can reuse the rest of the pipeline as is.
- To fit in 16 bits, register fields were cut to 3 bits (`x8`~`x15`) and immediate ranges were narrowed. In code compiled without C, 57.3% of register names were these 8 registers.
- I wrote a decompressor in Python and checked all 49,152 16-bit values against GNU objdump. Only 1 differed on RV64 and 1,537 on RV32, and every one was an encoding the spec leaves reserved or custom that objdump decoded leniently. In real code, 51–69% of instructions became 16-bit, shrinking code by 26–34%.
:::

All the instructions of RV32I·RV64I and the M·A·F·D extensions seen so far were 32 bits. Part 11 covers the C extension (compressed instructions, version 2.0), which shortens the frequently used ones to 16 bits. It is the extension that gave the biggest reduction, `rv32im` → `rv32imc`, in the code-size experiment of part 1. This post covers the ISA side: the 16-bit formats and their 32-bit equivalents, reserved encodings, and real compression ratios. Building the decompressor in hardware and fetching instructions on 16-bit boundaries are covered in part 21.

## 1. The C Extension's Promise: It Is Just Shorthand

The C extension has a single design principle: **every 16-bit instruction expands to exactly one 32-bit instruction.** Compressed instructions have no behavior of their own. So the decompressor is pure combinational logic, and after expansion the existing decoder, ALU, and exception handling work unchanged. Compilers and assemblers likewise generate 32-bit code and then just swap qualifying instructions for 16-bit ones.

The two lowest bits tell the lengths apart. `11` means 32 bits; `00`·`01`·`10` mean 16 bits. These three values are called quadrants 0·1·2. This is why the low two bits of a 32-bit opcode were always `11` in part 2.

With C, the instruction alignment requirement also changes from 4 bytes to 2 bytes. A 32-bit instruction may start on a 2-byte boundary and cross a 4-byte boundary. Branch and jump targets need only be multiples of 2, so the instruction-address-misaligned exceptions that occur on a core without C do not occur on a core with C.

## 2. The Nine 16-bit Formats

![The C extension has nine 16-bit formats, CR·CI·CSS·CIW·CL·CS·CA·CB·CJ, and the 3-bit register fields use only two positions, [9:7] and [4:2], in every format|full](/img/01M6K7VHKZRMHA96RG6RVXRGRY/00492036e596.webp)

There were 6 32-bit formats, but there are 9 16-bit formats. The fields are so narrow that each group of instructions keeps only the fields it really needs.

| Format | Registers | Representative instructions |
|---|---|---|
| CR | rd/rs1, rs2 (5 bits) | `c.mv` `c.add` `c.jr` `c.jalr` |
| CI | rd/rs1 (5 bits) + 6-bit immediate | `c.li` `c.addi` `c.lui` `c.slli` `c.lwsp` |
| CSS | rs2 (5 bits) + 6-bit immediate | `c.swsp` `c.sdsp` |
| CIW | rd' (3 bits) + 8-bit immediate | `c.addi4spn` |
| CL·CS | rs1', rd'/rs2' (3 bits) + 5-bit immediate | `c.lw` `c.sw` `c.ld` `c.sd` |
| CA | rd'/rs1', rs2' (3 bits) | `c.sub` `c.xor` `c.or` `c.and` |
| CB | rs1' (3 bits) + 8-bit offset | `c.beqz` `c.bnez` `c.srli` `c.andi` |
| CJ | None + 11-bit offset | `c.j` `c.jal` |

The principle followed in the 32-bit formats shows up here too. The 5-bit rd/rs1 sits in [11:7], the same place as rd in 32-bit instructions, and the 3-bit fields appear only in two positions, [9:7] and [4:2], regardless of format. The decompressor can extract register numbers before it has identified the format.

## 3. Why the 3-bit Register Field Means x8~x15

Fitting two register numbers (5 bits each), an opcode, and an immediate into 16 bits is hard. So the CIW·CL·CS·CA·CB formats cut register fields to 3 bits. Three bits can select only 8 registers, and the spec chose `x8`~`x15`. A 3-bit value r' is read as register number 8 + r'.

Why these 8? In ABI names they are `s0`·`s1`·`a0`~`a5`. Function arguments and return values (`a0`~`a5`) and the first saved registers to be used (`s0`·`s1`) are the most frequent in compiled code. I actually counted.

![In bench.c compiled without C, 641 of 1,118 register names (57.3%) were x8~x15|xl](/img/01M6K7VHKZRMHA96RG6RVXRGRY/5129ab32ec21.webp)

In the test code compiled without C (`rv32im`, `-O2`), register names appeared 1,118 times, and 641 of them (57.3%) were `x8`~`x15`. So 8 of 32 registers (25%) account for more than half the uses. The spec adds two more reasons. The numbers are contiguous and start at a multiple of 8, so the decompressor only has to prepend `01` (r' → `01`r'). And they work unchanged on RV32E·RV64E, which have only 16 integer registers.

Some formats keep 5-bit fields. The most common instructions, such as `c.mv`·`c.add`·`c.li`·`c.addi`, must be usable with any register, so the CR·CI formats keep 5-bit fields. The trade-off is a narrow 6-bit immediate.

## 4. 32-bit Equivalents

From the spec's RVC listing, here are the instructions common to RV32 and RV64 and their expansions. Most offset immediates drop their low bits so they can only express multiples of 4 or 8 (×4 for the lw·sw family, ×8 for the ld·sd family).

| Compressed | Expanded 32-bit | Range/condition |
|---|---|---|
| `c.lwsp rd` | `lw rd, off(x2)` | off 0~252, rd≠0 |
| `c.swsp rs2` | `sw rs2, off(x2)` | off 0~252 |
| `c.lw rd', rs1'` | `lw rd', off(rs1')` | off 0~124 |
| `c.sw rs2', rs1'` | `sw rs2', off(rs1')` | off 0~124 |
| `c.j` | `jal x0, off` | ±2KiB |
| `c.jr rs1` | `jalr x0, 0(rs1)` | rs1≠0 |
| `c.jalr rs1` | `jalr x1, 0(rs1)` | rs1≠0 |
| `c.beqz rs1'` | `beq rs1', x0, off` | ±256B |
| `c.bnez rs1'` | `bne rs1', x0, off` | ±256B |
| `c.li rd` | `addi rd, x0, imm` | −32~31 |
| `c.lui rd` | `lui rd, imm` | imm≠0, rd≠0·2 |
| `c.addi rd` | `addi rd, rd, imm` | −32~31 |
| `c.addi16sp` | `addi x2, x2, imm` | Multiple of 16, ≠0 |
| `c.addi4spn rd'` | `addi rd', x2, imm` | Multiple of 4, 4~1020 |
| `c.slli rd` | `slli rd, rd, sh` | |
| `c.srli`·`c.srai rd'` | `srli`·`srai rd', rd', sh` | |
| `c.andi rd'` | `andi rd', rd', imm` | −32~31 |
| `c.mv rd, rs2` | `add rd, x0, rs2` | rs2≠0 |
| `c.add rd, rs2` | `add rd, rd, rs2` | rs2≠0 |
| `c.sub`·`xor`·`or`·`and` | Same R format, rd'=rs1' | |
| `c.ebreak` | `ebreak` | |

Where RV32 and RV64 differ, the same encoding is used for different instructions.

| Encoding | RV32C | RV64C |
|---|---|---|
| Quadrant 1, funct3 001 | `c.jal` (`jal x1`) | `c.addiw` |
| Quadrant 0, funct3 011 | `c.flw` | `c.ld` |
| Quadrant 0, funct3 111 | `c.fsw` | `c.sd` |
| Quadrant 2, funct3 011·111 | `c.flwsp`·`c.fswsp` | `c.ldsp`·`c.sdsp` |
| Quadrant 1, CA with [12]=1 | Reserved | `c.subw`·`c.addw` |

On RV64, `addiw` is far more common than `jal`, and 64-bit loads are more common than single-precision loads, so the slots were swapped. The double-precision `c.fld`·`c.fsd`·`c.fldsp`·`c.fsdsp` are the same on both.

Two mappings deserve attention from the design side. `c.addi4spn` and `c.addi16sp` are dedicated to the stack pointer. Setting up stack frames at function entry and exit and forming addresses of local variables are so common that they got their own instructions. And `c.mv` expands to `add rd, x0, rs2`, not `addi`. This differs from the pseudo-instruction `mv` (= `addi rd, rs, 0`). The spec adds that a core that treats `mv` specially, such as one with register renaming hardware, may expand `c.mv` to `addi`. The result is the same.

## 5. The Immediate Bits Are Scrambled Again

Part 2 showed why the immediate bits of the B and J formats are scrambled: putting the same bit in the same position across formats reduces multiplexers. The 16-bit formats are scrambled even more. Let's decode one by hand: `c.lw s1, 20(a0)` from the compiled code, 16-bit value `0x4944`.

```text
0x4944 = 010 010 010 1 0 001 00
         f3  [12:10] rs1' [6] [5] rd' op
f3=010 (c.lw), op=00 (quadrant 0)
[12:10]=010 → off[5:3] = 2 → 16
[6]=1       → off[2]   = 1 → 4
[5]=0       → off[6]   = 0
rs1' = 010 → x10 (a0),  rd' = 001 → x9 (s1)
off = 16 + 4 = 20 → lw x9, 20(x10) = 0x01452483
```

The offset bits are not in order; they are scattered with 5:3 in [12:10], 2 in [6], and 6 in [5]. Unlike 32-bit `lw`, the offset is a multiple of 4, so there is no off[1:0]. `c.ld` uses the same positions as [12:10]=5:3, [6:5]=7:6 (being a multiple of 8, it has no off[2] but gains off[7]).

`c.addi16sp`, which changes the stack pointer, is scrambled even more. The immediate of the function's first instruction, `c.addi16sp sp, -304` (`0x7169`), is scattered as [12]=9, [6]=4, [5]=6, [4:3]=8:7, [2]=5. Decoded, it is −304, and the expansion is `addi sp, sp, -304` (`0xed010113`).

The spec explains the scrambling as an effort to keep as many bits as possible in the same position across all instructions. For example, the sign bit of the CI format is always [12]. The only cost is that humans find it hard to read; for hardware, moving bits is just wiring and costs nothing.

## 6. Reserved and Illegal Encodings, and HINTs

The 16-bit space (quadrants 0~2, 49,152 values) has quite a few unused values. The decompressor must treat them as illegal instructions.

| Encoding | Spec treatment | Count on RV32 |
|---|---|---|
| `0x0000` | Permanently illegal instruction | 1 |
| `c.addi4spn` imm=0 | Reserved | 7 |
| `c.addi16sp` imm=0 | Reserved | 1 |
| `c.lui` imm=0 | Reserved | 31 |
| `c.lwsp` rd=0, `c.jr` rs1=0 | Reserved | 64 + 1 |
| Quadrant 0, funct3 100 | Reserved (used by Zcb) | 2,048 |
| CA with [12]=1 | Reserved (partly used on RV64) | 256 |
| RV32 shift with sh[5]=1 | For custom extensions | 1,536 |

The reason `0x0000` is illegal is interesting. If execution jumps by mistake into erased memory or a zero-filled region, an illegal instruction exception fires immediately, catching the error early. In 32-bit, an all-ones value like `0xffffffff` is also illegal.

HINTs are different from reserved encodings. Encodings that discard their result, such as `c.li`·`c.addi`·`c.mv` with rd=`x0`, are not illegal but HINTs: just execute the expanded 32-bit instruction (nothing happens). These slots are kept for future uses such as performance hints. `c.addi` with rd=0, imm=0 is `c.nop`.

Other extensions have already started using the C extension's free slots. Quadrant 0 funct3 100 and part of CA are used by Zcb (`c.lbu`·`c.mul`, and others). This post and its experiments assume `rv32gc`·`rv64gc` without Zcb.

## 7. Hands-On 1: Exhaustive Check of a Decompressor Against objdump

I wrote the decompressor in Python. A single function takes a 16-bit value and returns either a 32-bit instruction or a "reserved" reason. The core is the 3-bit register translation and the immediate reassembly.

```python
def expand(c, xlen=32):
    op, f3 = c & 3, b(c, 15, 13)
    rd, rs2 = b(c, 11, 7), b(c, 6, 2)
    rdp, rs2p = 8 + b(c, 4, 2), 8 + b(c, 9, 7)
    imm6 = sx(b(c, 12) << 5 | b(c, 6, 2), 6)
    ...
    if op == 0 and f3 == 2:      # c.lw
        uw = b(c,5) << 6 | b(c,12,10) << 3 | b(c,6) << 2
        return I(uw, rs2p, 2, rdp, LOAD), 'c.lw'
```

The comparison worked like this. I assembled all 49,152 16-bit values in quadrants 0~2 with `.insn 2, value` and decoded them with objdump; I assembled the 32-bit values my decompressor produced with `.insn 4, value` and decoded them again for an architecture without C; then I compared the two texts. Branch targets differ by address, so I compared them as pc-relative distances. A value I classified as reserved counted as a match only if objdump also printed it as `.insn` (undecodable).

| Comparison | RV32 (`rv32gc`) | RV64 (`rv64gc`) |
|---|---|---|
| 16-bit values | 49,152 | 49,152 |
| Matched | 47,615 | 49,151 |
| └ of which HINTs | 331 | 363 |
| Mismatched | 1,537 | 1 |

Every mismatch was a case where objdump was more lenient than the spec. The 1 shared by RV64 and RV32 is `c.addi16sp` with imm=0 (`0x6101`). The spec calls it reserved, but objdump decoded it as `addi sp,sp,0`. The other 1,536 on RV32 are `c.slli`·`c.srli`·`c.srai` with shift amount sh[5]=1. Shifting by 32 or more is meaningless on RV32, so the spec leaves these for custom extensions, but objdump decoded them as, for example, `c.slli zero,0x20`.

The first comparison had more than 3,000 mismatches. Most were notation differences. objdump prints HINTs with compressed names such as `c.slli zero,0x0`, and prints `c.mv` as the pseudo-instruction `mv`. For the former, I replaced the compressed name with the expanded text before comparing; for `c.mv`, since the spec's expansion is `add` as seen above, I rewrote `mv rd, rs` as `add rd, zero, rs`. No bugs in the decompressor turned up in the process.

## 8. Hands-On 2: Compression Ratio of Real Code

The compression ratio depends on the code. I measured two programs. One is test code (`bench.c`, about 90 lines) containing SHA-256, quicksort, string functions, decimal output, a linked list, and a hash table; the other is `kernels.c` from part 1 (7 functions). I compiled the same file with and without C and compared `.text` sizes (GCC 16.2).

| Code (options) | Without C | With C | 16-bit share |
|---|---|---|---|
| bench.c (-O2, RV32) | 2,092 B | 1,546 B (73.9%) | 52.2% |
| bench.c (-Os, RV32) | 1,900 B | 1,358 B (71.5%) | 57.1% |
| bench.c (-O2, RV64) | 2,100 B | 1,562 B (74.4%) | 51.2% |
| kernels.c (-O2, RV32) | 668 B | 438 B (65.6%) | 68.9% |

The instruction count was the same with or without C (523 in both cases for bench.c -O2 RV32). As promised in Section 1, compression is a one-to-one mapping: it shortens instructions rather than removing them. More than half the instructions became 16-bit, and code shrank by 26–34%. I also ran bench.c on QEMU as `rv32im` and `rv32imc` to confirm the outputs matched and that the SHA-256 result matched Python's `hashlib`.

![Of the 273 16-bit instructions in bench.c (-O2, rv32imc), c.mv is the most common at 68, followed by c.add at 41 and c.addi at 33|xl](/img/01M6K7VHKZRMHA96RG6RVXRGRY/691ba495355e.webp)

I also looked at which compressed instructions were used most. The register copy `c.mv` was the most common, followed by `c.add`·`c.addi`, the stack load/store `c.lwsp`·`c.swsp`, the function return `c.jr` (`ret`), and the constant `c.li`. Instructions in the CR·CI·CSS formats, which keep 5-bit register fields, take the top spots. The 3-bit-field `c.lw`·`c.sw` appeared 10 and 7 times. On RV64, `c.sdsp`·`c.ldsp` (64-bit stack store/load) were second with 26 each: code that saves and restores registers at function entry and exit.

## 9. Design Checklist: Decoding Compressed Instructions

| Item | Rule |
|---|---|
| Length detection | If inst[1:0]≠11, it is 16 bits. A core without C treats it as illegal |
| Alignment | With C, instruction addresses need only be multiples of 2. The branch-target misalignment exception condition changes |
| 3-bit registers | Number = `01` + r'. Only two positions, [9:7] and [4:2] |
| `0x0000` | Illegal instruction (not reserved; permanently illegal) |
| Zero immediates | `c.addi4spn`·`c.addi16sp`·`c.lui` with imm=0 are reserved → illegal |
| rd·rs1=0 | `c.lwsp`·`c.ldsp`·`c.addiw` (rd=0) and `c.jr` (rs1=0) are reserved. `c.li`·`c.addi`·`c.mv`·`c.add` with rd=0 are HINTs (execute) |
| RV32 shifts | sh[5]=1 is for custom extensions. If unused, treat as illegal |
| `c.jalr` and `c.ebreak` | In quadrant 2 funct3 100 with [12]=1 and rs2=0, rs1=0 means `ebreak` |
| `c.mv` | The spec says `add rd, x0, rs2`. Expanding to `addi` is also allowed |
| Differences by XLEN | funct3 001 (`c.jal`/`c.addiw`), 011·111 (`flw`/`ld` family) |
| `mtval` | If instruction bits are written on an illegal instruction exception, write the original 16 bits (upper 16 bits zero) |
| No F·D | The `c.flw`·`c.fld` family is illegal if F·D are absent or disabled |

The last two rows connect to other parts. What to write in `mtval` is a rule of the privileged spec (part 25), and compressed floating-point instructions become illegal when the FPU is off (`mstatus.FS`=Off) because the rule from part 10 applies unchanged to the expanded 32-bit instruction.

## Summary

- Every 16-bit instruction of the C extension expands to one 32-bit instruction. If the low two bits are not `11`, it is 16 bits, and there are 9 formats.
- The 3-bit register fields point to the most used registers, `x8`~`x15` (`s0`·`s1`·`a0`~`a5`). In the test code, 57.3% of register names were these 8.
- `0x0000` is permanently illegal; some encodings such as a zero immediate or rd=0 are reserved; and encodings that discard their result are HINTs. In the decompressor check, all 1,537 values (RV32) that differed from objdump were reserved or custom slots under the spec.
- In real code, 51–69% of instructions became 16-bit, and code shrank by 26–34%. The instruction count stays the same.

**Next:** Part 12 is a bundle of small extensions. Running on QEMU, I check how Zicsr's `csrrw`·`csrrs`·`csrrc` change their read and write side effects when rd or rs1 is `x0`, plus Zifencei, the counters (Zicntr·Zihpm), the bit-manipulation B extension (Zba·Zbb·Zbs), branchless select Zicond, and cache management Zicbom.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Chapter 26, C extension version 2.0 (26.2 Compressed Instruction Formats, 26.3 Load and Store Instructions, 26.4 Control Transfer Instructions, 26.5 Integer Computational Instructions and Defined Illegal Instruction, 26.7 HINT Instructions, 26.8 RVC Instruction Set Listings), Chapter 27, Zc* extensions (Zcb). https://github.com/riscv/riscv-isa-manual
- A. Waterman, "Improving Energy Efficiency and Reducing Code Size with RISC-V Compressed," UCB/EECS-2011-63, 2011. https://www2.eecs.berkeley.edu/Pubs/TechRpts/2011/EECS-2011-63.html
- D. Patterson, A. Waterman, *The RISC-V Reader: An Open Architecture Atlas*, 2017. Chapter 7, RV32C
- RISC-V ELF psABI (register ABI names). https://github.com/riscv-non-isa/riscv-elf-psabi-doc
