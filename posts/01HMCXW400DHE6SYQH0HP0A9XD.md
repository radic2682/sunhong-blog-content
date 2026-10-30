---
id: "01HMCXW400DHE6SYQH0HP0A9XD"
slug: "arm-thumb-data-processing"
title: "ARM Thumb 명령어 집합 분류: 데이터 처리 명령어 정리"
category: "01M651PC003HYJYHCQPHT4KAWF"
tags: []
summary: "Thumb/Cortex-M0 Instruction Set"
thumbnail: "/img/01HMCXW400DHE6SYQH0HP0A9XD/03a76cbbc388.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-20T00:44:30+09:00"
---
## 6장 Instruction Set 개요

**ARM Cortex-M0**의 **Thumb Instruction Set**을 종류별로 정리한다. Data processing, Load/Store, Branch, Control flow 등 카테고리와 각 명령어의 문법 및 encoding을 다룬다.

## Thumb Instruction Set 분류

ARM Cortex-M0는 다음 카테고리의 명령어를 지원:

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
- `CMP Rd, #imm`: flags만 set

### Logical

- `AND Rd, Rn`: Rd = Rd AND Rn
- `ORR Rd, Rn`: Rd = Rd OR Rn
- `EOR Rd, Rn`: Rd = Rd XOR Rn (exclusive OR)
- `BIC Rd, Rn`: Rd = Rd AND NOT Rn
- `MVN Rd, Rn`: Rd = NOT Rn
- `TST Rn, Rm`: flags만 set (AND)

### Shift

- `LSL Rd, Rn, #imm`: Logical shift left
- `LSR Rd, Rn, #imm`: Logical shift right
- `ASR Rd, Rn, #imm`: Arithmetic shift right (sign-extend)
- `ROR Rd, Rn`: Rotate right

### Multiply

- `MUL Rd, Rn`: Rd = Rd × Rn (32-bit, lower 32-bit only)
- Single-cycle or 32-cycle multiplier (옵션)

![Data Processing|full](/img/01HMCXW400DHE6SYQH0HP0A9XD/03a76cbbc388.webp)

## Load/Store Instructions

ARM은 load/store architecture — 메모리와 register 간 전송은 LDR/STR로만.

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

- `LDMIA Rn!, {Rlist}`: Rn에서 시작, reg list에 순차 load, Rn update
- `STMIA Rn!, {Rlist}`: 저장 버전
- `PUSH {Rlist}`: Stack에 push (STMDB SP!, {Rlist})
- `POP {Rlist}`: Stack에서 pop (LDMIA SP!, {Rlist})

### PC-Relative Load

- `LDR Rt, [PC, #imm]`: 상수 테이블 로드

![Load/Store|full](/img/01HMCXW400DHE6SYQH0HP0A9XD/372372db7803.webp)

## Branch Instructions

### Unconditional Branch

- `B label`: PC = label (±2 KB)
- `BL label`: LR = PC+4, PC = label (±16 MB) — function call
- `BX Rm`: PC = Rm (register indirect, Thumb/ARM state switch)
- `BLX Rm`: LR = PC+4, PC = Rm

### Conditional Branch

- `B{cond} label`: condition true일 때만 branch
  - BEQ, BNE, BCS/BHS, BCC/BLO, BMI, BPL, BVS, BVC
  - BHI, BLS, BGE, BLT, BGT, BLE

### Return from Subroutine

- `BX LR`: return from function (LR에 저장된 주소로)
- `POP {PC}`: PC를 stack에서 복원

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

![System 명령어|full](/img/01HMCXW400DHE6SYQH0HP0A9XD/a6e675f498e8.webp)

## Miscellaneous

- `NOP`: No Operation (pipeline bubble)
- `BKPT #imm`: Breakpoint (debug)
- `YIELD`: Multi-threading hint
- `REV Rd, Rn`: Reverse byte order in word (endian swap)
- `REV16`, `REVSH`: halfword 단위 swap

## 상수 설정

- 32-bit 상수는 **MOVS + LDR [PC, ...]** 또는 여러 명령어로 조립
- Thumb-2에서는 MOVW/MOVT로 16-bit씩 로드

### 예시

```
MOV R0, #0x12       ; R0 = 0x12
LSL R0, R0, #24     ; R0 = 0x12000000
ADD R0, #0x34       ; R0 = 0x12000034
```

또는 LDR pseudo-instruction:
```
LDR R0, =0x12345678   ; assembler가 literal pool 생성
```

![Constants 및 예제|full](/img/01HMCXW400DHE6SYQH0HP0A9XD/f842fb56c54e.webp)

## Encoding Format

Thumb 16-bit 명령어는 opcode + register fields + immediate로 구성.

예: `ADD R0, R1, R2` encoding:
- opcode[15:9]: `0001100`
- Rm[8:6]: R2
- Rn[5:3]: R1
- Rd[2:0]: R0

32-bit Thumb-2 명령어는 2개의 16-bit word로 구성, 첫 word의 상위 5-bit가 `11101`, `11110`, 또는 `11111`이면 32-bit임을 나타냄.

## 정리

| 카테고리 | 예 |
|---------|---|
| **Data processing** | ADD, SUB, MUL, AND, ORR, LSL, LSR |
| **Load/Store** | LDR, STR, PUSH, POP, LDMIA, STMIA |
| **Branch** | B, BL, BX, B{cond} |
| **System** | SVC, WFI, DMB, MRS, MSR |
| **Misc** | NOP, BKPT, REV |

ARM Cortex-M0의 Thumb 명령어는 **단순하고 고정 길이 (16-bit)** 라서 decode가 쉽고 코드 효율이 좋다. 대부분의 임베디드 코드가 이 명령어 집합만으로 구현 가능하다.

![Encoding 예|full](/img/01HMCXW400DHE6SYQH0HP0A9XD/a3885086ca4d.webp)
