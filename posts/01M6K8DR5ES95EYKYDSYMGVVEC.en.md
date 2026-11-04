---
id: "01M6K8DR5ES95EYKYDSYMGVVEC"
title_en: "RISC-V Decoder Design: Decode Tables, Formats, and Illegal Instructions"
summary_en: "Part 15 of the RISC-V series. Fills in the RV32I decode table down to the control signals and defines five illegal instruction conditions, then checks it against GNU objdump on 174,608 words. 165,923 matched, and all 8,685 mismatches were explained by four causes."
---
:::note
**Key points in three lines**
- An RV32I decoder does three things. It takes the register numbers from fixed positions and sends them straight on, picks one row of the decode table (a bundle of control signals) using opcode, funct3, and funct7, and flags combinations not in the table as illegal instructions. Of the 32 major opcode slots, RV32I uses 11.
- The illegal conditions boil down to five: the 16-bit slots, missing opcodes, missing funct3 values, missing funct7 values (including inst[25] in shifts), and the remaining SYSTEM values. Conversely, the spare fields of FENCE and HINTs with rd=x0 must be let through.
- I checked a decoder written in Python against GNU objdump (restricted to rv32i) on 174,608 words. 165,923 matched, and all 8,685 mismatches were explained by four cases where objdump differs from the specification (0 unexplained mismatches).
:::

Parts 1~3 looked at the shape of RV32I (the six formats and the immediate layout) and what its integer operations compute. The checklist at the end of part 2 wrote down in a few lines the rules a decoder must follow. Part II of the series, starting with part 15, is about turning those rules into an actual core. The first component is the decoder. This post fills in the decode table for all of RV32I down to the control signals, defines the illegal instruction detection conditions more finely than part 2, and then sweeps the 32-bit space in bulk to match verdicts with GNU objdump. This set of posts (parts 15~18) explains the design with a cycle-level Python model instead of RTL. I wrote the model so that each line later becomes one block of RTL.

## 1. What the Decoder Does

When a 32-bit instruction comes in, the decoder outputs the following.

| Output | Where it goes | When it is determined |
|---|---|---|
| rs1, rs2, rd numbers | Register file | Format-independent, fixed positions |
| Format (I, S, B, U, J, R) | Immediate generator | From the opcode alone |
| Control signals | ALU, memory, selectors | opcode, funct3, funct7 |
| Illegal flag | Exception handling | When there is no row in the table |

![The RV32I decoder sends register numbers straight from fixed positions, looks up the decode table with opcode, funct3, and funct7, and groups combinations not in the table and the remaining SYSTEM values as illegal|full](/img/01M6K8DR5ES95EYKYDSYMGVVEC/6fe4e0df65fd.webp)

The first row is a big advantage of the RISC-V decoder. As we saw in part 2, rs1, rs2, and rd are in the same bits in every format. So the decoder does not "interpret" these fields. Wiring them to the register file addresses is all it takes. In the U and J formats, the rs1 position holds immediate bits, but the value read then is simply not used.

There is one thing to watch when building a pipeline, though. If you do not know "does this instruction really read rs1," hazard detection guesses wrong. For example, the rs1 position of `lui a0, 0x12345` holds part of the immediate (`0x08` here). If the hazard detector takes this as a real rs1, it stalls needlessly waiting for an earlier instruction that writes x8. So I also put a signal in the decode table for "does it actually read rs1/rs2" (used in part 17).

## 2. Major Opcodes and Format Determination

If inst[1:0] is `11`, it is a 32-bit instruction, and the remaining 5 bits inst[6:2] are the major opcode. The specification's base opcode map lays out these 32 slots by inst[6:5] (rows) and inst[4:2] (columns).

![Of the 32 major opcode slots, RV32I uses 11: LOAD, STORE, BRANCH, JALR, JAL, MISC-MEM, OP-IMM, OP, AUIPC, LUI, and SYSTEM; the other 21 are illegal in an RV32I decoder|full](/img/01M6K8DR5ES95EYKYDSYMGVVEC/15b0a3e70c94.webp)

RV32I uses 11 slots. Of the other 21, 17 belong to other standard extensions (F's LOAD-FP, A's AMO, RV64's OP-32, etc.), custom use (custom-0~3), or are reserved, and the 4 slots in the last column are set aside for instructions longer than 32 bits. In a core that implements only RV32I, all 21 are illegal.

Format determination finishes with the opcode alone. There is no need to look at funct3.

| Major opcode | Format | What funct3 distinguishes | Looks at funct7? |
|---|---|---|---|
| LUI, AUIPC | U | Nothing | No |
| JAL | J | Nothing | No |
| JALR, LOAD, OP-IMM | I | Size, operation | Shifts only |
| STORE | S | Size | No |
| BRANCH | B | Comparison condition | No |
| OP | R | Operation | Yes |
| MISC-MEM, SYSTEM | I | FENCE / ECALL, EBREAK | Separate rules |

Translated into logic, the format signals are a few ORs of the opcode decoder outputs (11 one-hot signals). For example, `isI = JALR | LOAD | OPIMM | MISCMEM | SYSTEM`. The immediate generator receives only these five signals (I, S, B, U, J) (part 16).

Let's trace one by hand. `sub a0, a0, a1` is `0x40b50533`.

```text
funct7  rs2   rs1   f3  rd    opcode
0100000 01011 01010 000 01010 0110011
```

Opcode `0110011` is OP, so it is R format. funct3 `000` is add or sub, and funct7 `0100000` (inst[30]=1) selects sub. rs1=10, rs2=11, and rd=10 come from the same positions regardless of format. Changing only funct7 to `0000001` gives `0x02b50533`, the M extension's `mul`. The OP row of an RV32I decoder has no funct7 `0000001`, so it is illegal.

## 3. The Decode Table, Down to the Control Signals

The 37 RV32I instructions plus FENCE, ECALL, and EBREAK make 40 rows, and that is the whole table. Below is the model's table (`15/rv32i.py`) printed as is. Similar rows within the same opcode are grouped.

```text
name        fmt  rw A B op    mem  wb branch/jump
lui          U   1  z i add   -    a  -
auipc        U   1  p i add   -    a  -
jal          J   1  - - -     -    4  jal
jalr         I   1  r i add   -    4  jalr
beq…bgeu     B   0  r r -     -    -  eq ne lt ge ltu geu
lb lh lw     I   1  r i add   r1 r2 r4  m  -
lbu lhu      I   1  r i add   r1u r2u   m  -
sb sh sw     S   0  r i add   w1 w2 w4  -  -
addi…andi    I   1  r i (f3)  -    a  -
slli srli    I   1  r i sll srl -  a  -
srai         I   1  r i sra   -    a  -
add…and      R   1  r r (f3,f7) -  a  -
fence        I   0  - - -     -    -  (does nothing)
ecall ebreak I   0  - - -     -    -  trap
```

The columns mean the following.

| Signal | Values | Meaning |
|---|---|---|
| rw | 0/1 | Register write (RegWrite) |
| A, B | r, p, z / r, i | ALU inputs: rs1, pc, 0 / rs2, immediate |
| mem | r, w, size | Load/store, byte count, u=unsigned |
| wb | a, m, 4 | Writeback: ALU, memory, pc+4 |

A few choices deserve explanation. For `lui`, ALU input A is set to 0 so it computes "0 + immediate." That way a single adder can be reused as is (the part 3 checklist). `auipc` puts pc into A. `jal` does not use the ALU. The target pc+imm is computed separately by the branch adder, and pc+4 goes into rd. `jalr` computes rs1+imm with the ALU and then clears the least significant bit. Branches have a comparator judge the condition instead of the ALU (part 16 looks at why they are split that way).

In OP-IMM, only the three shifts look at the funct7 position. In `addi`, inst[30] is just an immediate bit, and using it to select sub is the common bug flagged in part 3. So the OP-IMM ALU operation is chosen by funct3 alone, and only when funct3 is `101` does inst[30] separate srli from srai.

## 4. Illegal Instruction Detection Conditions

Rewriting the part 2 checklist as decoder circuit conditions gives five rows.

| Condition | Example | Basis |
|---|---|---|
| ① inst[1:0] ≠ `11` | `0x00000000` | 1.5 Length encoding |
| ② opcode outside the 11 | `0xffffffff` | Base opcode map |
| ③ Missing funct3 | `ld` (LOAD `011`) | 2.4~2.6 |
| ④ Missing funct7 | `mul`, shift with inst[25]=1 | 2.4 |
| ⑤ Remaining SYSTEM values | `0x00200073` | 2.8 |

① The all-zero word (`0x00000000`) is defined as illegal in section 1.5 of the specification, to catch jumps into erased memory. Without the C extension, a word with inst[1:0] = `00` cannot be a 32-bit instruction, so it naturally falls under this condition. Adding the C extension changes things. The 16-bit `0x0000` must be illegal on its own, so it has to be caught separately in the decompressor (part 21).

② The all-ones word is also defined as illegal in section 1.5 (on RV32I, where ILEN=32). inst[6:0]=`1111111` is a long-instruction slot, so it falls straight into the opcode check. The result is the same, but the model counted the reason separately.

④ inst[25] of a shift-immediate is the sixth bit of shamt in RV64I. In the RV32I instruction listing, this position is fixed at `0000000`/`0100000`, so inst[25]=1 is an encoding not in the table, i.e., a reserved encoding. Section 2.2 of the specification does not define the behavior on a reserved encoding (UNSPECIFIED). The common choice is an illegal instruction exception. The same place also notes that a platform may require an exception on opcodes reserved for standard use.

⑤ In the SYSTEM opcode, RV32I defines exactly two values: `ecall` (`0x00000073`) and `ebreak` (`0x00100073`). CSR instructions (funct3 ≠ 0) are the Zicsr extension (Chapter 7), and values such as `mret` and `wfi` come from the Privileged specification. The decoder in this post accepts only RV32I, so both are left illegal. A real M-mode core would add rows for Zicsr and the privileged instructions to the table (parts 23~26).

Conversely, there are two things that **must not be made illegal**.

- **The spare fields of FENCE.** Section 2.7 of the specification reserves FENCE's rs1 and rd for future use while stating that "base implementations shall ignore these fields." It also says reserved combinations of fm and pred/succ should be "treated as a normal FENCE with fm=0000." So in MISC-MEM, you only need to check funct3=`000`. FENCE.I (funct3 `001`) is the Zifencei extension (Chapter 6), so it is illegal if you implement only RV32I.
- **HINTs.** Arithmetic instructions with rd=x0 (`addi x0, x5, 3`, `add x0, x0, x5`, etc.) and FENCEs with pred or succ equal to 0 are HINTs (section 2.9 of the specification). Execute them as usual and just discard the write to x0. From the decoder's point of view, doing nothing special is the right answer. This "do nothing special" HINT becomes a trap in the part 17 pipeline.

## 5. Hands-On: Matching Verdicts with objdump

What should the decoder's "legal/illegal" verdict be compared against? As in part 2, I used the GNU tools as the reference, but this time the disassembler objdump rather than the assembler. When a 32-bit word goes in, if objdump prints an instruction name it counts as legal, and if it prints `.insn` it counts as illegal.

There were two pitfalls here.

1. If you feed words in with `.word 0x…`, objdump treats them as data and does not disassemble them. That is because the assembler attaches the mapping symbol `$d`, which marks data locations. You have to remove it with `objcopy --strip-symbol='$d'` for them to be read as instructions.
2. How far objdump decodes into extensions is set by the ELF's `Tag_RISCV_arch` attribute. Assembling with `-march=rv32i` sets it to `rv32i2p1`, and objdump decodes only this range unless given `-M max`. With `-M max`, another 58,361 words in the same sample became "legal" (mostly the `fmadd` family from F, D, Q, and Zfh). This objdump version has no separate `-M` option for choosing extensions, so restricting the range through the attribute is the right approach.

```python
def objdump(ws):
    src = '\n'.join(f'.word 0x{w:08x}' for w in ws)
    open('a.s', 'w').write(src)
    run(['riscv64-elf-as', '-march=rv32i',
         '-o', 'a.o', 'a.s'])
    run(['riscv64-elf-objcopy', '--strip-symbol=$d',
         'a.o', 'b.o'])
    out = run(['riscv64-elf-objdump', '-d',
               '-M', 'no-aliases', 'b.o'])
    # per line: address → first word (instruction name or .insn)
```

The sample has three sets (seed 15): (a) all 17-bit combinations of the 7 opcode bits, funct3, and funct7, with the remaining fields filled randomly, 4 copies; (b) 200,000 uniformly random 32-bit words; (c) all 4,096 values of the 12-bit immediate field for SYSTEM and MISC-MEM (with rs1 and rd zero and nonzero). After removing duplicates, that is 740,630 words. Of these, the 566,022 with inst[1:0]≠`11` or in a long-instruction slot are not read as 32-bit by objdump, so only my decoder judges them illegal; the remaining 174,608 were matched against objdump.

| My decoder | objdump | Count |
|---|---|---|
| Legal | Legal | 32,132 |
| Illegal | Illegal | 133,791 |
| Legal | Illegal | 8,633 |
| Illegal | Legal | 52 |

I broke the 8,685 mismatches down by cause.

| Cause | Count | Who is right |
|---|---|---|
| FENCE rd, rs1, fm fields | 8,633 | The spec (my decoder) |
| Privileged instructions | 38 | Scope difference |
| 6-bit shamt on rv32 | 13 | The spec (my decoder) |
| `unimp` (`0xc0001073`) | 1 | Scope difference |

- **FENCE.** objdump prints `fence` only when rd and rs1 are 0 and fm is `0000`, or for the exact FENCE.TSO (`0x8330000f`). Other values are `.insn`. Of the 8,633 mismatches, 4,771 had rd≠0, 4,780 had rs1≠0, and 8,337 had a reserved fm (with overlap). According to section 2.7 of the specification, all of these must execute as a normal FENCE. It seems the disassembler was designed this way because it is a tool that shows only "the forms standard software uses" (an estimate). Changing the decoder to match objdump would violate the specification.
- **Privileged instructions.** objdump decoded `mret`, `sret`, `wfi`, `dret`, and `sfence.vma` regardless of `-march` (32 were `sfence.vma`). The no-longer-used `uret` and `hret` also appeared. It is simply a different scope from a decoder that implements only RV32I.
- **6-bit shamt.** Of 714 shifts with inst[25]=1, objdump also judged 701 illegal. 13 differed, all cases where the funct7 position was `0000001` (slli, srli) or `0100001` (srai). If it has the shape of an RV64-style 6-bit shamt, objdump accepts it even on rv32. For example, it printed `0x02009013` as `slli zero,ra,0x20`. On RV32, a 32-bit shift is an impossible value.
- **`unimp`.** The CSR write `csrrw x0, cycle, x0` is the idiom for "unimplemented instruction" and has the name `unimp`. objdump printed this one value by name even without Zicsr.

Excluding these four, there are 0 unexplained mismatches. In the same sample, 560 of the words the decoder judged legal were HINTs, and the decoder let them through as is.

Looking only at the 199,973 random 32-bit words, 8,967 (4.5%) were legal RV32I. Three quarters (150,087) failed on inst[1:0]≠`11`; 32,585 had missing opcodes, 6,138 missing funct3, 1,798 missing funct7, 202 shift inst[25], and 196 remaining SYSTEM values. Because the legal space is this narrow, a jump to a wrong address has a high chance of being caught quickly by an illegal instruction exception.

## 6. Design Checklist: The Decoder

| Item | Rule |
|---|---|
| Register numbers | Do not interpret; wire from fixed positions. Build a separate "actually reads" signal |
| Format | Decide from the opcode alone. Do not wait for funct3 |
| inst[30] in OP-IMM | Meaningful only when funct3=`101`. In `addi` it is an immediate bit |
| Shift inst[25] | A reserved encoding on RV32. The common choice is an illegal exception |
| FENCE | Check only funct3=`000`. Ignore rs1, rd, fm, pred, succ (including reserved values) |
| FENCE.I | Illegal without Zifencei (funct3 `001`) |
| SYSTEM | With only RV32I, everything other than `0x00000073` and `0x00100073` is illegal. Adding Zicsr or privileged instructions adds rows to the table |
| Zero and all ones | Both illegal. With the C extension, the 16-bit `0x0000` must be caught separately |
| HINT | Operations with rd=x0, FENCE with pred/succ 0. Let them through without an exception |
| Reserved encodings | Behavior is UNSPECIFIED. Most designs make them illegal exceptions (match the verification reference) |
| Reference for checking | objdump differs from the spec on FENCE fields, 6-bit shamt, and privileged instructions. Filter these out when using it as a reference |

The last row connects to the verification story in parts 33~34. Only by knowing that the reference model can be wrong do you avoid blindly changing your design when a mismatch shows up.

## Summary

- The decoder sends register numbers straight from fixed positions, picks the format from the opcode, and picks one row of control signals from opcode, funct3, and funct7. RV32I uses only 11 of the 32 major opcode slots.
- The control signals are RegWrite, ALU input A/B selection, ALU operation, memory read/write and size, writeback, branch condition, jump type, and "does it actually read rs1/rs2."
- The illegal conditions are the 16-bit slots, missing opcode/funct3/funct7 (including shift inst[25]), and the remaining SYSTEM values. The spare fields of FENCE and HINTs must be let through.
- Checking 174,608 words against objdump, 165,923 matched, and all 8,685 mismatches were explained by four causes: FENCE fields, privileged instructions, 6-bit shamt, and `unimp`.

**Next post:** Part 16 takes the format chosen by the decoder and turns the immediate generator, which builds the immediate, directly from the multiplexer table of part 2 into a circuit, then connects the PC, instruction memory, register file, ALU, data memory, and writeback selector to complete a single-cycle datapath. It runs five actually compiled programs on that model and matches them instruction by instruction against QEMU.

## References

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. 1.5 Base Instruction-Length Encoding (zero and all ones), 2.2 Base Instruction Formats (reserved encodings are UNSPECIFIED), 2.4 Integer Computational Instructions (shift encodings), 2.7 Memory Ordering Instructions (FENCE fields to ignore), 2.8 ECALL and EBREAK, 2.9 HINT Instructions, Chapter 6 Zifencei, Chapter 7 Zicsr, and the base opcode map in the instruction listing chapter (RV32/64G Instruction Set Listings). https://github.com/riscv/riscv-isa-manual
- D. Harris, S. Harris, *Digital Design and Computer Architecture: RISC-V Edition*, 2021. 7.3 Single-cycle processor control unit (main decoder, ALU decoder)
- GNU Binutils 2.47 objdump documentation, RISC-V disassembler options (`-M max`, `no-aliases`). https://sourceware.org/binutils/docs/binutils/objdump.html
