---
id: "01HMCXW400DHE6SYQH0HP0A9XD"
title_en: "Classifying the ARM Thumb Instruction Set: Data Processing Instructions"
summary_en: "Notes classifying the Cortex-M0 Thumb instruction set by type. Covers the syntax and encoding of register, immediate, logical, shift, and multiply data processing instructions; basic, offset, multiple, and PC-relative loads and stores; and branch instructions."
---
## Chapter 6: Instruction Set Overview

This chapter classifies the **Thumb instruction set** of the **ARM Cortex-M0** by type. It covers categories such as data processing, load/store, branch, and control flow, along with the syntax and encoding of each instruction.

## Thumb Instruction Set Categories

The ARM Cortex-M0 supports instructions in the following categories:

1. **Data processing**: ADD, SUB, MUL, AND, ORR, EOR, CMP, MOV
2. **Data transfer (Load/Store)**: LDR, STR, LDRB, STRB, LDRH, STRH, LDM, STM
3. **Branch**: B, BL, BX, BLX
4. **System**: SVC, WFI, WFE, DSB, DMB, ISB, MRS, MSR

## Data Processing Instructions

### Register-Register

- `ADD Rd, Rn, Rm`: Rd = Rn + Rm
- `SUB Rd, Rn, Rm`: Rd = Rn - Rm
- `ADC Rd, Rn`: Rd = Rd + Rn + C
- `SBC Rd, Rn`: Rd = Rd - Rn - ¬C

### Register-Immediate

- `ADD Rd, #imm`: Rd = Rd + imm (0~255 for Rd=R0~R7)
- `SUB Rd, #imm`: Rd = Rd - imm
- `MOV Rd, #imm`: Rd = imm
- `CMP Rd, #imm`: sets flags only

### Logical

- `AND Rd, Rn`: Rd = Rd AND Rn
- `ORR Rd, Rn`: Rd = Rd OR Rn
- `EOR Rd, Rn`: Rd = Rd XOR Rn (exclusive OR)
- `BIC Rd, Rn`: Rd = Rd AND NOT Rn
- `MVN Rd, Rn`: Rd = NOT Rn
- `TST Rn, Rm`: sets flags only (AND)

### Shift

- `LSL Rd, Rn, #imm`: Logical shift left
- `LSR Rd, Rn, #imm`: Logical shift right
- `ASR Rd, Rn, #imm`: Arithmetic shift right (sign-extend)
- `ROR Rd, Rn`: Rotate right

### Multiply

- `MUL Rd, Rn`: Rd = Rd × Rn (32-bit, lower 32-bit only)
- Single-cycle or 32-cycle multiplier (optional)

![Data Processing|full](/img/01HMCXW400DHE6SYQH0HP0A9XD/03a76cbbc388.webp)

## Load/Store Instructions

ARM is a load/store architecture: transfers between memory and registers happen only through LDR/STR.

### Basic

- `LDR Rt, [Rn, #imm]`: Rt = mem[Rn + imm] (word, 32-bit)
- `STR Rt, [Rn, #imm]`: mem[Rn + imm] = Rt
- `LDRB Rt, [Rn, #imm]`: byte load (zero-extended)
- `STRB Rt, [Rn, #imm]`: byte store
- `LDRH Rt, [Rn, #imm]`: halfword load
- `STRH Rt, [Rn, #imm]`: halfword store
- `LDRSB`, `LDRSH`: signed byte/halfword load

### Register Offset

- `LDR Rt, [Rn, Rm]`: Rt = mem[Rn + Rm]

### Multiple Load/Store

- `LDMIA Rn!, {Rlist}`: start at Rn, load the register list in order, update Rn
- `STMIA Rn!, {Rlist}`: the store version
- `PUSH {Rlist}`: push onto the stack (STMDB SP!, {Rlist})
- `POP {Rlist}`: pop from the stack (LDMIA SP!, {Rlist})

### PC-Relative Load

- `LDR Rt, [PC, #imm]`: load from a constant table

![Load/Store|full](/img/01HMCXW400DHE6SYQH0HP0A9XD/372372db7803.webp)

## Branch Instructions

### Unconditional Branch

- `B label`: PC = label (±2 KB)
- `BL label`: LR = PC+4, PC = label (±16 MB), a function call
- `BX Rm`: PC = Rm (register indirect, Thumb/ARM state switch)
- `BLX Rm`: LR = PC+4, PC = Rm

### Conditional Branch

- `B{cond} label`: branches only when the condition is true
  - BEQ, BNE, BCS/BHS, BCC/BLO, BMI, BPL, BVS, BVC
  - BHI, BLS, BGE, BLT, BGT, BLE

### Return from Subroutine

- `BX LR`: return from a function (to the address saved in LR)
- `POP {PC}`: restore the PC from the stack

![Branch|full](/img/01HMCXW400DHE6SYQH0HP0A9XD/1a366413e7c6.webp)

## System Instructions

- `SVC #imm`: Supervisor Call (OS service)
- `WFI`: Wait For Interrupt (sleep)
- `WFE`: Wait For Event
- `SEV`: Send Event
- `DMB`: Data Memory Barrier
- `DSB`: Data Synchronization Barrier
- `ISB`: Instruction Synchronization Barrier

### Status Register Access

- `MRS Rd, <reg>`: Read special register (APSR, IPSR, EPSR, MSP, PSP, PRIMASK, CONTROL)
- `MSR <reg>, Rn`: Write special register

![System instructions|full](/img/01HMCXW400DHE6SYQH0HP0A9XD/a6e675f498e8.webp)

## Miscellaneous

- `NOP`: No Operation (pipeline bubble)
- `BKPT #imm`: Breakpoint (debug)
- `YIELD`: Multi-threading hint
- `REV Rd, Rn`: Reverse byte order in word (endian swap)
- `REV16`, `REVSH`: swap in halfword units

## Setting Constants

- A 32-bit constant is assembled with **MOVS + LDR [PC, ...]** or with several instructions
- Thumb-2 loads it 16 bits at a time with MOVW/MOVT

### Example

```
MOV R0, #0x12       ; R0 = 0x12
LSL R0, R0, #24     ; R0 = 0x12000000
ADD R0, #0x34       ; R0 = 0x12000034
```

Or with the LDR pseudo-instruction:
```
LDR R0, =0x12345678   ; the assembler creates a literal pool
```

![Constants and examples|full](/img/01HMCXW400DHE6SYQH0HP0A9XD/f842fb56c54e.webp)

## Encoding Format

A 16-bit Thumb instruction consists of an opcode + register fields + an immediate.

Example: encoding of `ADD R0, R1, R2`:
- opcode[15:9]: `0001100`
- Rm[8:6]: R2
- Rn[5:3]: R1
- Rd[2:0]: R0

A 32-bit Thumb-2 instruction consists of two 16-bit words. If the top 5 bits of the first word are `11101`, `11110`, or `11111`, the instruction is 32-bit.

## Summary

| Category | Examples |
|---------|---|
| **Data processing** | ADD, SUB, MUL, AND, ORR, LSL, LSR |
| **Load/Store** | LDR, STR, PUSH, POP, LDMIA, STMIA |
| **Branch** | B, BL, BX, B{cond} |
| **System** | SVC, WFI, DMB, MRS, MSR |
| **Misc** | NOP, BKPT, REV |

ARM Cortex-M0 Thumb instructions are **simple and fixed-length (16-bit)**, so they are easy to decode and give good code density. Most embedded code can be written with this instruction set alone.

![Encoding example|full](/img/01HMCXW400DHE6SYQH0HP0A9XD/a3885086ca4d.webp)
