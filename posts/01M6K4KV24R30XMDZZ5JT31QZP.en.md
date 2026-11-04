---
id: "01M6K4KV24R30XMDZZ5JT31QZP"
title_en: "RV32I 1: Registers, Six Instruction Formats, and Why Immediates Are Scrambled"
summary_en: "Part 2 of the RISC-V series. Looks at RV32I's registers and six instruction formats, and explains why the immediate bits are scrambled by counting multiplexer inputs in the immediate generator (70 vs. 98). A hand-written encoder and decoder were checked against GNU as on 5,000 instructions."
---
:::note
**Key points in three lines**
- Every RV32I instruction is 32 bits, and there are six formats: R, I, S, B, U, and J. The register numbers (rs1, rs2, rd) sit in the same bit positions in every format, and the sign bit of the immediate is always bit 31.
- The immediate bits look scrambled because that saves hardware. The bits were moved so that B shares as many positions as possible with S, and J with I. Counting multiplexer inputs in the immediate generator gives 70 for the RISC-V layout versus 98 for a naive layout that fills bits in order.
- I wrote an encoder and decoder in Python and checked them against the GNU assembler. All 6 hand-encoded instructions and all 5,000 random ones matched.
:::

Part 1 drew the big map of RISC-V. From part 2 on, we go into the base integer ISA, RV32I. This post covers the "shape" of instructions: registers, instruction formats, and immediate encoding. What each instruction does comes in part 3 (arithmetic) and part 4 (branches and memory). Formats come first because, when you design a core, the decoder and the immediate generator come straight out of this format table.

## 1. Registers

RV32I has 32 integer registers of 32 bits (`x0`~`x31`) and a program counter, `pc`. In RV64I they all become 64 bits. This bit width is called XLEN.

| Register | ABI name | Use |
|---|---|---|
| `x0` | `zero` | Always 0. Writes are discarded |
| `x1` | `ra` | Function return address |
| `x2` | `sp` | Stack pointer |
| `x3`, `x4` | `gp`, `tp` | Global pointer, thread pointer |
| `x5`~`x7`, `x28`~`x31` | `t0`~`t6` | Temporaries (a function call may overwrite them) |
| `x8`, `x9`, `x18`~`x27` | `s0`~`s11` | Saved registers (a function must restore them) |
| `x10`~`x17` | `a0`~`a7` | Function arguments and return values |

From the hardware's point of view, `x0` is the only special register. ABI names and uses are a convention among software (more in part 6), so the hardware treats `x1`~`x31` all the same. `pc` is not a general register, so instructions cannot read or write it directly. Only `auipc`, `jal`, `jalr`, and branches use `pc`.

The single rule that `x0` is always 0 saves many instructions. The register copy `mv a0, a1` is `addi a0, a1, 0`, negation `neg a0, a1` is `sub a0, x0, a1`, and the do-nothing `nop` is `addi x0, x0, 0`. When building the register file, you do not need real flip-flops for entry 0; just make it "reads return 0, writes are ignored."

## 2. The Overall Layout of a 32-Bit Instruction

![The six RV32I instruction formats: the register number fields are in the same place in every format, and the immediate's sign bit is always bit 31|full](/img/01M6K4KV24R30XMDZZ5JT31QZP/651239ebf905.webp)

Start with the fields common to all formats.

| Field | Bits | Meaning |
|---|---|---|
| `opcode` | [6:0] | Instruction type (major class). The low two bits are always `11` |
| `rd` | [11:7] | Destination register number |
| `funct3` | [14:12] | Distinguishes operations within the same opcode |
| `rs1` | [19:15] | First source register number |
| `rs2` | [24:20] | Second source register number |
| `funct7` | [31:25] | Further distinction (R format only) |

As we saw in part 1, an instruction whose low two bits are `11` is a 32-bit instruction (anything else is a 16-bit C extension instruction). So of the 7 `opcode` bits, the 5 bits [6:2] are what actually distinguish the type. Values where [4:2] is `111` are reserved for instructions of 48 bits or longer.

The key to this layout is that **the register number fields are in the same place regardless of format**. Before you know whether an instruction is R or S, you can send [19:15] and [24:20] straight to the register file as addresses. This is what lets a pipelined core do decode and register read in parallel in the same cycle. Some formats have no rs2 (I, U, J), but then the value read is simply not used. Reading needlessly is faster than figuring out the format first and then reading.

## 3. The Six Formats

| Format | Used for | Registers | Immediate |
|---|---|---|---|
| R | Register-register ops `add`, `sub`, `sll` … | rd, rs1, rs2 | None |
| I | Immediate ops `addi`, loads `lw`, `jalr` | rd, rs1 | 12 bits, imm[11:0] |
| S | Stores `sw`, `sh`, `sb` | rs1, rs2 | 12 bits, in two pieces |
| B | Conditional branches `beq`, `blt` … | rs1, rs2 | 13 bits (even), imm[12:1] |
| U | Upper 20 bits `lui`, `auipc` | rd | imm[31:12] |
| J | Jump `jal` | rd | 21 bits (even), imm[20:1] |

In effect there are four formats, R, I, S, and U, and B and J are variants of S and U. B has rs1, rs2, and a 12-bit field like S, but because it is a branch distance it only uses multiples of 2; J has a 20-bit field like U, but because it is a jump distance it only uses multiples of 2. That is why the specification introduces B and J as variants of S and U.

Targets are always multiples of 2 because of the C extension. With 16-bit instructions, instructions sit on 2-byte boundaries. So imm[0] is always 0, and instead of storing it, that bit is used to double the range. Branches reach ±4KiB and `jal` reaches ±1MiB. A core without C support raises an instruction address misaligned exception when the target is not a multiple of 4 (the checklist in part 1).

## 4. Why Are the Immediate Bits Scrambled?

Look at B and J in the figure and the order of the immediate bits is strange. B has imm[12] at the far left and imm[11] at the right end (bit 7), and J runs imm[20], imm[10:1], imm[11], imm[19:12]. It is awkward for humans to read. From the hardware's point of view, though, the reason is clear.

The immediate generator is the circuit that takes the 32 instruction bits and produces the 32-bit immediate for the format. Each output bit is a multiplexer that decides "which instruction bit to pick, depending on the format." If the same immediate bit comes from the same instruction bit across formats, the multiplexer has fewer inputs. RISC-V moved the bits to follow this rule.

| Immediate bit | I | S | B | U | J | Distinct sources |
|---|---|---|---|---|---|---|
| imm[0] | inst[20] | inst[7] | 0 | 0 | 0 | 3 |
| imm[4:1] | inst[24:21] | inst[11:8] | inst[11:8] | 0 | inst[24:21] | 3 |
| imm[10:5] | inst[30:25] | inst[30:25] | inst[30:25] | 0 | inst[30:25] | 2 |
| imm[11] | inst[31] | inst[31] | **inst[7]** | 0 | **inst[20]** | 4 |
| imm[19:12] | inst[31] | inst[31] | inst[31] | inst[19:12] | inst[19:12] | 2 |
| imm[30:20] | inst[31] | inst[31] | inst[31] | inst[30:20] | inst[31] | 2 |
| imm[31] | inst[31] | inst[31] | inst[31] | inst[31] | inst[31] | 1 |

A few things stand out in the table.

- **imm[10:5] is inst[30:25] in all four formats.** So it has only two sources (the instruction bits, or U's 0).
- **For imm[4:1], B uses the same position as S, and J the same as I.** Since B does not store imm[0], S's inst[7] slot is free, and imm[11] was put there. Thanks to that, B's imm[4:1] and imm[10:5] line up exactly with S.
- **J works the same way.** J's imm[11] was put in I's inst[20] slot (imm[0] in I), so imm[10:1] lines up with I's inst[30:21]. imm[19:12] is inst[19:12], the same as U.
- **imm[31] is always inst[31].** Because the sign bit is fixed in one place, sign extension can start before the format is fully decoded. No multiplexer is needed either (1 source).

I counted the effect myself. For each output bit, I gathered the sources across the five formats (I, S, B, U, J) and took the number of distinct sources as the multiplexer input count (the constant 0 also counts as one input). For comparison I defined a "naive layout" that uses the same bit fields but fills the immediate in order from the low bit up: B puts imm[12:1] in order into S's fields, and J puts imm[20:1] in order into inst[31:12].

![Counting multiplexer inputs per immediate output bit gives a total of 70 for the RISC-V layout and 98 for the naive in-order layout|xl](/img/01M6K4KV24R30XMDZZ5JT31QZP/5113473079e9.webp)

The RISC-V layout had 70 inputs in total, with the widest multiplexer at 4 inputs (imm[11]). The naive layout had 98, with the widest at 5 inputs. The difference is largest in the low bits. In the naive layout, imm[1] of B and J is offset by one position from S and I, so almost every bit from imm[1] to imm[11] gains one more source. Fewer multiplexer inputs mean less area and also a shorter path through the immediate generator. That path easily ends up on the critical path of the decode stage.

## 5. Encoding by Hand

The best way to learn the formats is to fill in the bits yourself. I picked one instruction from each of the six formats, encoded it by hand, and compared it with the value produced by the GNU assembler (`riscv64-elf-as`).

**`addi x5, x6, -20` (I format).** Written as a 12-bit two's complement value, -20 is `1111 1110 1100` (0xFEC). Concatenating rs1=6 (`00110`), funct3=0, rd=5 (`00101`), and opcode=OP-IMM (`0010011`) gives the following.

```text
imm[11:0]      rs1   f3  rd    opcode
111111101100  00110 000 00101 0010011
= 0xfec30293
```

**`beq x1, x2, -16` (B format).** Written in 13 bits, -16 is `1 1111 1111 0000`. Here imm[12]=1, imm[11]=1, imm[10:5]=`111111`, imm[4:1]=`1000`, and imm[0] is not stored. Place these in B's field order.

```text
[12] [10:5]  rs2   rs1   f3  [4:1] [11] opcode
 1   111111 00010 00001 000 1000   1   1100011
= 0xfe2088e3
```

The branch distance is relative to the address of the branch instruction itself. So in the assembler I wrote it as a distance from the current location (`.`), as in `beq x1,x2,.-16`.

| Instruction | Format | Hand-computed | GNU as |
|---|---|---|---|
| `add x3,x4,x5` | R | `005201b3` | `005201b3` |
| `addi x5,x6,-20` | I | `fec30293` | `fec30293` |
| `sw x7,8(x2)` | S | `00712423` | `00712423` |
| `beq x1,x2,.-16` | B | `fe2088e3` | `fe2088e3` |
| `lui x10,0x12345` | U | `12345537` | `12345537` |
| `jal x1,.+2048` | J | `001000ef` | `001000ef` |

`jal x1,.+2048` is a fun one. 2048 is the value with only imm[11] set. In the J format imm[11] sits at inst[20], so the result is `0x00100000`, with only bit 20 of the 32-bit value set, plus the opcode and rd: `0x001000ef`.

## 6. Hands-On: Writing an Encoder and Decoder and Checking Them Against the Assembler

A few hand calculations are not enough, so I wrote an RV32I encoder and decoder in Python and checked them. The machine was a Mac M1 Max, and the reference was GNU binutils (`riscv64-elf-as` and `objdump`, installed with GCC 16.2). The immediate part of the decoder is a direct transcription of the table above.

```python
def decode_imm(fmt, w):
    b = lambda hi, lo: (w >> lo) & ((1 << (hi - lo + 1)) - 1)
    if fmt == 'I': return sext(b(31, 20), 12)
    if fmt == 'S': return sext(b(31, 25) << 5 | b(11, 7), 12)
    if fmt == 'B': return sext(b(31, 31) << 12 | b(7, 7) << 11
                               | b(30, 25) << 5 | b(11, 8) << 1, 13)
    if fmt == 'U': return sext(b(31, 12), 20) << 12
    if fmt == 'J': return sext(b(31, 31) << 20 | b(19, 12) << 12
                               | b(20, 20) << 11 | b(30, 21) << 1, 21)
```

`sext(v, n)` sign-extends an n-bit value. The test went like this. I randomly picked 5,000 instructions from 34 RV32I instructions representing the six formats (excluding shift-immediates, FENCE, and system instructions), with registers and immediates also chosen at random within their ranges (seed fixed at 2). Each instruction was built once with my encoder and once by writing it as an assembly statement and running it through GNU as, and the 32-bit values were compared. Then I read the values produced by GNU as back with my decoder to see whether the original register numbers and immediates came out.

| Test | Count | Matched |
|---|---|---|
| Hand encoding | 6 | 6 |
| Random encoding (my encoder = GNU as) | 5,000 | 5,000 |
| Random decoding (read back with my decoder) | 5,000 | 5,000 |

The same method applies when the decoder moves into hardware: compare against a reference model (GNU as here, later an instruction set simulator such as Spike) with random inputs. Transcribe even one table wrong and hundreds of the 5,000 fail immediately, so mistakes are hard to hide.

## 7. Design Checklist: What the Decoder Must Get Right

An RV32I decoder determines the format from opcode bits [6:2]. RV32I uses nine major opcodes.

| opcode[6:0] | Name | Format | Instructions |
|---|---|---|---|
| `0110111` | LUI | U | `lui` |
| `0010111` | AUIPC | U | `auipc` |
| `1101111` | JAL | J | `jal` |
| `1100111` | JALR | I | `jalr` |
| `1100011` | BRANCH | B | `beq` `bne` `blt` `bge` `bltu` `bgeu` |
| `0000011` | LOAD | I | `lb` `lh` `lw` `lbu` `lhu` |
| `0100011` | STORE | S | `sb` `sh` `sw` |
| `0010011` | OP-IMM | I | `addi` `slti` `sltiu` `xori` `ori` `andi` `slli` `srli` `srai` |
| `0110011` | OP | R | `add` `sub` `sll` `slt` `sltu` `xor` `srl` `sra` `or` `and` |

On top of these come `FENCE` (MISC-MEM, `0001111`) and `ECALL`/`EBREAK` (SYSTEM, `1110011`) (part 4). Things that are easy to miss when building the decoder:

| Item | Rule |
|---|---|
| Unlisted opcode | Any opcode not listed above raises an illegal instruction exception |
| Unlisted funct3 | E.g., funct3 `011` for LOAD (RV32 has no `ld`), and `010`/`011` for BRANCH, are illegal |
| funct7 | For OP, anything other than `0000000` and `0100000` (sub, sra) is illegal (without the M extension) |
| Shift immediates | RV32I `slli`/`srli`/`srai` have a 5-bit shift amount. If inst[25] is 1, it is a reserved value; treat it as illegal |
| `jalr` | Illegal if funct3 is not `000` |
| All zeros, all ones | The instructions `0x00000000` and `0xffffffff` are defined as illegal by the specification, to catch jumps into erased memory |
| rd = x0 | An operation that writes its result to x0, such as `addi x0, x5, 3`, is not illegal but a HINT. It must do nothing and move on without an exception |
| Field extraction | Extract rs1, rs2, and rd from their fixed positions before determining the format. Take the immediate's sign straight from inst[31] |

The second-to-last row is a common mistake. If you treat "instructions that write x0 are useless, so illegal," perfectly valid programs will stop with exceptions on encodings the specification reserved as hints for future extensions. `nop` itself is `addi x0, x0, 0`.

## Summary

- RV32I has 32 registers of 32 bits and a `pc`. The only register the hardware treats specially is `x0`, which is always 0.
- Instructions are 32 bits, in six formats: R, I, S, B, U, and J. rs1, rs2, and rd are always in the same place, so decode and register read can happen at once, and the immediate's sign is always inst[31].
- The immediate bits were moved so that B shares positions with S, and J with I. This cut multiplexer inputs from 98 in the naive layout to 70.
- The hand-written encoder and decoder matched GNU as on all 6 (by hand) and all 5,000 (random) instructions. The checklist above sums up the decoder's illegal instruction and HINT rules.

**Next post:** Part 3 looks at RV32I's integer operations. It follows by hand what the add, logical, shift, and compare instructions compute, how `slt` and `sltu` differ, and how to combine `lui`/`auipc` with `addi` to build 32-bit constants and addresses (including the +1 correction caused by sign extension).

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Chapter 2 RV32I (2.1 Programmers' Model, 2.2 Instruction Formats, 2.3 Immediate Encoding Variants, 2.9 HINT Instructions), 1.5 Base Instruction-Length Encoding. https://github.com/riscv/riscv-isa-manual
- D. Patterson, A. Waterman, *The RISC-V Reader: An Open Architecture Atlas*, 2017. Chapter 2 (RV32I)
- D. Harris, S. Harris, *Digital Design and Computer Architecture: RISC-V Edition*, Morgan Kaufmann, 2021. 6.4 Machine Language, 7.3 Immediate Extension
- RISC-V ELF psABI (register ABI names). https://github.com/riscv-non-isa/riscv-elf-psabi-doc
