---
id: "01M6K7YAQT9JD45WH9Y2S8WP2Z"
title_en: "C Extension: Decompressor, 16-Bit Fetch, and Misaligned 32-Bit Instructions"
summary_en: "Part 21 of the RISC-V series. I built the C extension's decompressor and fetch buffer in Python and checked all 49,152 16-bit values and a real rv32imc program against objdump. 36.7% of executed instructions spanned two words, and the fetch buffer cut word reads from 33,771 to 23,903."
---
:::note
**The 3 key points of this post**
- Implementing the C extension amounts to adding a fetch buffer and a decompressor to the IF stage. Each 16-bit instruction always expands to exactly one 32-bit instruction, so the decoder and execution units stay as they are, and a single "compressed or not" bit changes only pc+2/+4 and the link address.
- I expanded all 49,152 16-bit values with a Python decompressor and compared them against GNU objdump. All 28,823 valid and HINT encodings had the same meaning, and the 1,537 that differed were encodings reserved on RV32 (such as shift amount bit 5 = 1) that objdump simply decoded and displayed anyway.
- Running a real rv32imc program, 36.7% of executed instructions were 32-bit instructions spanning two 32-bit words, and 96.5% of branch targets sat at a 4-byte boundary + 2. A fetch buffer with three 16-bit slots cut word reads from 33,771 to 23,903 compared with re-reading every time without a buffer.
:::

In parts 19 and 20 we added M and A to the RV32I 5-stage pipelined core. Both were about the execution side (EX and MEM). The C extension, by contrast, changes only the front end (IF). The instruction formats were covered in part 11, so this time I build Python models of how to pull a stream of mixed 16-bit and 32-bit instructions out of memory organized in 32-bit words, how to expand 16 bits into 32 bits, and which 16-bit values must be treated as illegal, and compare them against GNU objdump and QEMU execution traces.

## 1. What the C extension asks of a core

With the C extension, a 32-bit instruction can start at any 2-byte boundary, not only at a 4-byte boundary (IALIGN = 16, Unprivileged spec section 26.1). The specification sets one overarching principle: **every compressed instruction expands to a single 32-bit instruction in the base ISA (or F/D).** The specification itself cites the benefit that "the hardware can simply expand it in the decode stage."

| What changes | What | Why |
|---|---|---|
| Fetch | Pull the next instruction in 16-bit units | A 32-bit instruction can span two words |
| Length detection | 32-bit if [1:0] of the first 16 bits is `11` | The low two bits `00`, `01`, `10` are the compressed quadrants |
| Decompressor | 16-bit → 32-bit table | To reuse the decoder as is |
| pc update | pc + 2 or + 4 | Two instruction lengths |
| Link address | pc + 2 for `c.jal` and `c.jalr` | An exception the specification deliberately spells out |
| Branch target | Valid if 2-byte aligned | The alignment exception condition disappears |

The last row is easy to miss. With C, a branch or jump target only needs to be a multiple of 2, and the specification states that "with the C extension, no instruction raises an instruction-address-misaligned exception." `jalr` already clears bit 0 of the target, so no case is left over.

## 2. The decompressor: 16 bits to 32 bits

The decompressor is pure combinational logic. It picks the instruction by quadrant (op = [1:0]) and funct3 ([15:13]), gathers the scattered immediate bits, and places them back into the 32-bit format. The 3-bit register fields (rd′, rs1′, rs2′) get `01` prepended to become x8–x15.

Let's expand by hand `0xf77d`, at the end of the CRC loop in the program I ran.

| Field | Bits | Value | Meaning |
|---|---|---|---|
| op | [1:0] | `01` | Quadrant 1 |
| funct3 | [15:13] | `111` | `c.bnez` |
| rs1′ | [9:7] | `110` | x8 + 6 = x14 (a4) |
| offset | [12], [6:5], [2], [11:10], [4:3] | 1, 11, 1, 01, 11 | offset[8], [7:6], [5], [4:3], [2:1] |

Lining up the offset bits from the most significant gives `1_1110_1110`, which is −18 as a 9-bit signed value. The expansion is `bne a4, x0, -18`, machine code `0xfe0717e3`, and −18 from 0x80000170 matches the target 0x8000015e that objdump showed. What matters is that the offset of the expanded 32-bit instruction is also relative to **the compressed instruction's own pc**. The pc doesn't change, so the branch address circuit is reused as is.

![With a fetch buffer, length detection, and a decompressor in the IF stage, the decoder sees only 32-bit instructions, and a single is_rvc bit selects pc+2/+4 and the link address|full](/img/01M6K7YAQT9JD45WH9Y2S8WP2Z/cdef194f9907.webp)

Expanding a few more makes the pattern visible.

| 16-bit | Name | Expanded 32-bit |
|---|---|---|
| `0x4505` | `c.li a0,1` | `addi a0,x0,1` |
| `0x177d` | `c.addi a4,-1` | `addi a4,a4,-1` |
| `0x8fb9` | `c.xor a5,a4` | `xor a5,a5,a4` |
| `0x6105` | `c.addi16sp 32` | `addi sp,sp,32` |
| `0x8082` | `c.jr ra` | `jalr x0,0(ra)` |

Only `c.jalr` does not map exactly. It expands to `jalr x1, 0(rs1)`, but the link value is pc + 2, not pc + 4. The specification points this out too, saying that "supporting both offsets of 2 and 4 is a small change." In an implementation, the `is_rvc` bit the decompressor emits can serve as the select signal for the link address adder. The same goes for `c.jal`.

## 3. Illegal compressed instructions and HINTs

In the 16-bit space, 49,152 values have low two bits other than `11`. Only some of them are valid instructions in RV32C (no F or D). The decompressor must raise illegal for the rest. Here is how the model classified them.

| Class | Count | Examples |
|---|---|---|
| Valid | 28,460 | `c.lw`, `c.addi`, `c.j` … |
| HINT (no effect when executed) | 363 | `c.nop 1`, `c.li x0,5`, `c.mv x0,ra` |
| Illegal/reserved | 20,329 | Table below |

The illegal/reserved class mixes encodings of different kinds. Each becomes a condition in the decompressor.

| Encoding | Reason |
|---|---|
| `0x0000` (all zeros) | An illegal instruction defined by the specification (section 26.5.4) |
| `c.addi4spn` imm = 0, `c.addi16sp` imm = 0, `c.lui` imm = 0 | Reserved |
| `c.lwsp` rd = 0, `c.jr` rs1 = 0 | Reserved |
| shamt[5] = 1 for `c.slli`, `c.srli`, `c.srai` | Reserved for custom extensions in RV32C |
| `c.flw`, `c.fsw`, `c.fld`, `c.fsd` and their sp versions | Illegal without F/D |
| Quadrant 0 funct3 = `100`, the `c.subw`/`c.addw` slots in quadrant 1 | Reserved (or belonging to RV64/Zcb) |

HINTs are encodings that "have no effect because rd = x0." The decompressor can simply expand them to 32 bits. `c.li x0, 5` becomes `addi x0, x0, 5`, and since writes to x0 are discarded, it naturally does nothing. No special handling is needed.

I compared the model's results against objdump. I placed all 49,152 values at 4-byte intervals with `.insn`, assembled them, and compared the lines objdump read with the lines it read for my decompressor's expanded 32-bit instructions placed at the same addresses. Alias differences, such as `mv` meaning both `addi rd,rs,0` and `add rd,x0,rs`, were treated as having the same meaning.

| Comparison result | Count |
|---|---|
| Valid/HINT, same meaning | 28,823 |
| Both illegal | 18,792 |
| Illegal in my decompressor, decoded by objdump | 1,537 |
| Other differences | 0 |

The 1,537 are `c.slli` with shamt[5] = 1 (1,024), `c.srli`/`c.srai` with shamt[5] = 1 (512), and `c.addi16sp` with imm = 0 (1). objdump simply displayed these as `c.slli zero,0x20`, `srli s0,s0,0x20`, `addi sp,sp,0`, and so on. A disassembler is a display tool, so it reads leniently. A hardware decompressor must treat them as illegal, as the specification says. Executing shamt[5] = 1 on RV32 as a shift by 32 in a core without custom extensions is a common bug.

## 4. The fetch buffer: pulling out instructions that span words

Instruction memory (or the cache) delivers aligned 32-bit words. But instructions start at 2-byte boundaries, so the lower half of a 32-bit instruction can be in the upper part of one word and the upper half in the lower part of the next word. There are three cases.

| pc mod 4 | Instruction | Words needed |
|---|---|---|
| 0 | 16-bit or 32-bit | 1 |
| 2 | 16-bit | 1 (upper half) |
| 2 | 32-bit | 2 (upper half of this word + lower half of the next word) |

The simplest approach is to re-read the needed words for every instruction. A spanning instruction is read twice. If instead you keep a small buffer that holds the leftover 16 bits, each word is read only once during sequential execution. The model's buffer has three 16-bit slots. It checks the length from [1:0] of the front slot and emits the instruction once the needed slots are full. When two or more slots are empty, it reads the next word to fill them.

```python
def ready(self):
    if not self.q: return 0
    n = 1 if self.q[0][1] & 3 != 3 else 2   # number of 16-bit slots
    return n if len(self.q) >= n else 0

def redirect(self, pc):
    self.q = []; self.fetch_addr = pc & ~3
    self.skip = pc & 2   # if the target is at +2, drop the lower half of the first word
```

When a branch or jump changes the pc, the buffer is flushed and reading restarts from the word that contains the target. If the target is at a 4-byte boundary + 2, the lower half of the first word is dropped. If the target instruction is 32-bit, the first word alone isn't enough, so one more read is needed. That adds one more bubble cycle to the fetch.

![The branch target srli of the CRC inner loop spans two words at 0x8000015e, so every branch reads two words and adds one more bubble|full](/img/01M6K7YAQT9JD45WH9Y2S8WP2Z/ae9da1550e49.webp)

I measured how often this case comes up in a real program. I built a C program (`prog.c`) mixing sorting, CRC32, string reversal, and matrix multiplication with `-march=rv32imc -O1`. Its .text was 880 bytes, 23% smaller than the rv32im build of the same program (1,144 bytes).

## 5. Hands-on 1: checking the static instruction stream against objdump

First, I pulled the whole .text through the fetch buffer in order from the start. For each instruction pulled, I compared the address, bits, and length with the corresponding `riscv64-elf-objdump -d` line, and for compressed instructions I read my expanded 32-bit instruction back through objdump to confirm it had the same meaning as the original line.

| Item | Result |
|---|---|
| Instructions pulled / objdump instructions | 286 / 286 |
| Address order matches | Yes |
| Address, bits, length match | 286 / 286 |
| Compressed expansions have the same meaning | 132 / 132 |
| Compression ratio | 46.2% |
| 32-bit instructions spanning two words | 67 / 154 (43.5%) |

Almost half of the 32-bit instructions span two words. Once a single compressed instruction is inserted, the 32-bit instructions after it are shifted by 2 bytes, so this is expected. The 46.2% compression ratio for this program is slightly below the "typically 50–60%" the specification mentions.

## 6. Hands-on 2: counting fetches in execution order

Next is the actual execution order. I ran QEMU with one translation block per instruction (`-accel tcg,one-insn-per-tb=on -d exec,nochain`) and recorded the executed pcs in order. I cut off the spin loop at the end of the trace (`j 1b`; the number of iterations until QEMU stopped varied every run). Feeding this pc sequence into the fetch buffer model, whenever the next pc is not pc + length, it is treated as a branch and the buffer is flushed. I ran it while checking that the address of each instruction the model pulled matched QEMU's pc every time.

| Item (rv32imc, -O1) | Value |
|---|---|
| Instructions executed | 24,713 |
| Compressed instruction ratio | 55.6% |
| 32-bit instructions spanning two words | 9,058 (36.7% of the total) |
| Changes in pc flow | 3,409 |
| Of those, target at a 4-byte boundary + 2 | 3,291 (96.5%) |
| Of those, target is a spanning 32-bit instruction | 3,184 |

That nearly all targets sit at +2 is particular to this program. The two most frequent targets are the CRC inner loop (1,792 times) and the sort inner loop (1,059 times), and both happened to start with a 32-bit instruction at a +2 position. So I rebuilt with GCC's alignment options (`-falign-loops=4 -falign-jumps=4 -falign-labels=4`) and repeated the measurement.

![Running the same program read 33,771 32-bit words without a buffer, 23,903 with the fetch buffer, and 22,028 with targets aligned to 4 bytes|xl](/img/01M6K7YAQT9JD45WH9Y2S8WP2Z/bab11eb43a49.webp)

| Configuration | Words read | Fetch bubbles |
|---|---|---|
| rv32im (no compression) | 24,713 | |
| rv32imc, re-read every time | 33,771 | |
| rv32imc, fetch buffer | 23,903 | 6,594 |
| rv32imc, fetch buffer + target alignment | 22,028 | 3,411 |

Fetch bubbles are the cycles spent waiting for the first instruction after a branch to be fully assembled. Each branch costs at least 1 (reading the target word), and 1 more if the target is a spanning 32-bit instruction. 3,409 + 3,184 + 1 at the start = 6,594, which checks out. Aligning the targets cut spanning targets down to 1, reducing bubbles to 3,411, at the cost of the code growing from 880 to 900 bytes and executed instructions rising slightly to 24,738 (when execution passes through alignment `nop`s).

Also worth noting is that rv32im and rv32imc execute the same 24,713 instructions. Compression changes only the size, not the instructions. Yet re-reading every time without a buffer actually reads 37% more than uncompressed code. To save instruction memory bandwidth with the C extension, a fetch buffer is essential.

The machine is a Mac M1 Max, with `riscv64-elf` GCC 16.2 and binutils 2.47, and QEMU 11.1.3. Running the whole model (including checking every 16-bit value) took about 1 second.

## 7. Design checklist: the C extension

| Item | Rule |
|---|---|
| Length detection | 32-bit if [1:0] of the first 16 bits = `11`. Encodings of 48 bits or more are not yet in the standard |
| Register fields | rd′, rs1′, rs2′ = x8–x15 (prepend `01`) |
| Immediates | Bit placement differs per instruction, as does sign extension (e.g. the `c.lw` offset is zero-extended, `c.addi` is sign-extended) |
| Link address | pc + 2 for `c.jal` and `c.jalr` |
| Next pc | pc + 2 / pc + 4. Branch offsets are relative to the compressed instruction's own pc |
| Alignment exception | With C, no instruction-address-misaligned exception. If `misa.C` can be turned off, the rules in that case follow the Privileged spec |
| Illegal | `0x0000`, the three reserved imm = 0 cases, `c.lwsp`/`c.jr` with x0, RV32 shamt[5] = 1, FP loads/stores without F/D |
| HINT | Expand and execute as is (no effect because rd = x0) |
| mtval | If the instruction bits are written on an illegal instruction trap, use the original 16 bits, not the expanded 32 bits |
| Fetch buffer | Holds leftover 16 bits. Flush on a branch; drop the lower half if the target is at +2 |
| Spanning two words | The second word can cross a cache line or page boundary. A fetch fault on the second word is also this instruction's fault |
| LR/SC loops | Compressed instructions also count as "base I instructions" under the constrained LR/SC loop rules (section 26.6) |

Of the last two rows, the page boundary issue comes up again from part 30 on, which deals with virtual memory. If the upper half of a spanning instruction is on the next page and that page is missing, the fault address (mtval) may point not to the instruction's start address but to the part that actually failed (within what the Privileged spec allows; to be checked in that part).

## Summary

- Implementing the C extension comes down to the fetch buffer, length detection, and decompressor in the IF stage. The decoder and execution units stay as they are, and a single `is_rvc` bit selects pc+2/+4 and the link address for `c.jal`/`c.jalr`.
- I expanded all 49,152 16-bit values and compared them against objdump. 28,823 valid/HINT encodings had the same meaning, and the 1,537 that objdump decoded are reserved in RV32C, so hardware must treat them as illegal.
- In a real program, 36.7% of executed instructions were 32-bit instructions spanning two words. A fetch buffer with three 16-bit slots cut word reads from 33,771 to 23,903, fewer than uncompressed code (24,713).
- If a branch target is a 32-bit instruction at a +2 position, one more fetch bubble occurs. In this program, aligning targets cut bubbles from 6,594 to 3,411 at the cost of 20 more bytes of code.

**Next post:** Part 22 covers the last instruction-side problem, FENCE.I and self-modifying code. Using QEMU and a Python model, it examines why freshly written code is invisible on a core with split instruction and data caches, what FENCE.I does and does not guarantee (same hart only), and why even a simple core without caches must flush this part's fetch buffer.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. Chapter 26, "C" Extension (26.1 Overview, 26.2 Formats, 26.4 Control Transfer, 26.5 Integer Computational and 26.5.4 Defined Illegal Instruction, 26.6 LR/SC Ordering, 26.7 HINTs, 26.8 Instruction Listings). https://github.com/riscv/riscv-isa-manual
- D. Harris et al., *RISC-V System-on-Chip Design*, 2023. Compressed instruction decompressor and fetch implementation (CORE-V Wally)
- D. Patterson, A. Waterman, *The RISC-V Reader*, 2017. Chapter 7, RV32C
- A. Waterman, "Improving Energy Efficiency and Reducing Code Size with RISC-V Compressed," M.S. thesis, UC Berkeley, 2011
- QEMU logging options (`-d exec`, `one-insn-per-tb`). https://www.qemu.org/docs/master/devel/tcg.html
