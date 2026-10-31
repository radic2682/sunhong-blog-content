---
id: "01HJF4AW00GCX1KCN8GGCN1R2A"
slug: "arm-assembly-basics"
title: "ARM 어셈블리 프로그래밍 기초: 명령어 형식과 산술·논리 연산"
category: "01M651PC00PPZ9M79TYD9S0526"
tags: []
summary: "ARM Assembly, indexing"
thumbnail: "/img/01HJF4AW00GCX1KCN8GGCN1R2A/ed9a9440f0f4.webp"
note: true
created: "2023-12-25T09:00:00+09:00"
updated: "2026-04-19T05:36:48+09:00"
---
## 4장 ARM Assembly Programming 개요

ARM Assembly의 기본 문법과 실제 프로그래밍에 필요한 명령어, Immediate operation, Shifted Register Operands, Addressing Mode, Branch, Subroutine 등을 다룬다.

## ARM Assembly 기본 구조

```
[label] instruction [operands] [; comment]
```

- **Label**: 코드 라인에 이름을 붙여 jump target 지정
- **Instruction**: ARM mnemonic
- **Operands**: 주로 register, immediate, label
- **Comment**: `;`로 시작

## General Features

ARM Assembly의 일반적인 특징.

- **Arithmetic, Logical Operations**: ADD, SUB, AND, ORR 등
- ALU 연산은 register-to-register, 최대 3 operand

### Suffixed Format

`<instruction>{S}{cond}` 형태. `S`는 flag update, `cond`는 conditional execution.

예: `ADDEQS r0, r1, r2` → EQ 조건이 참일 때만 실행, flag 갱신

## Simple Register Operands

- **ADD r0, r1, r2**: `r0 = r1 + r2`
- 3 operand 또는 2 operand form

### Arithmetic Operation

- ADD, SUB, RSB (reverse subtract), ADC (add with carry), SBC, RSC
- 왜 RSB가 필요한지: ARM은 `r0 = r1 - r2`와 `r0 = r2 - r1`을 immediate 활용 시 모두 유용

### Bitwise Logical Operation

- AND, ORR, EOR (XOR), BIC (bit clear = AND NOT)
- 예: `BIC r0, r1, #0xFF` → r1의 하위 8bit를 모두 0으로

### Bit Shift Operation

- MOV, MVN (move NOT), shift 포함 가능

### Comparison Operation (No destination)

- CMP (compare = SUB, flag만), CMN (compare negative = ADD, flag만)
- TST (test = AND, flag만), TEQ (test equal = EOR, flag만)

![ARM Assembly 개요 및 Register/Arithmetic Operations|full](/img/01HJF4AW00GCX1KCN8GGCN1R2A/ed9a9440f0f4.webp)

## Immediate Operation

하나의 operand (뒤의 operand)는 **상수**로 올 수 있음 (`#`을 이용).

예: `ADD r3, r3, #1`, `ADD r8, r7, #&ff`

### 상수 설정 조건

ARM instruction은 32-bit 고정이라 immediate에는 **12 bit** 공간. `Cond | opcode | 12bit(rn) | 4bit(r1) | 4bit | 4bit 12bit`.

- 상수 범위: $0 \sim 2^{12} - 1$까지만 표현 가능
- **12-bit을 더 잘 사용하기 위해** (많이 사용하는 bit만) **shift시켜 사용**
- $(0 \sim 2^8 - 1) \times 2^{2n} = (0 \sim 2^8 - 1) \ll 2n$ — 8bit × non 4bit shift
- 4bit을 사용해서 $0 \leq n \leq 12$까지 표현
- Encode할 수 없는 값이면 Assembler가 경고

## Shifted Register Operands

**3 or 4 address instruction**: 2번째 reg operand를 shift할 수 있기에 4 address I.

예: `ADD r3, r2, r1, LSL #3` → `r3 = r2 + 8*r1`
- `<<3`은 8을 곱해준 것과 같음
- 곱셈기는 1 clock 안에 수행되지 않는 경우가 많음 → **shift 연산을 수행하면 곱셈도 1 clock에 수행 가능** (ARM의 특징, 다른 프로세서는 별도 자원)

reg로 사용도 가능. 예: `ADD r5, r5, r3, LSL r2` → `r5 = r5 + r3 * 2^r2`

### Shift Mnemonics

- **LSL(R)**: Logical shift left (Right)
- **ASL(R)**: Arithmetic shift left (Right) → ASR의 경우 Sign-extension을 해줘 LSR과 결과가 다름
- **ROR**: Rotate right
- **RRX**: Carry도 합쳐서 Rotate

![Immediate Operation 및 Shifted Register|full](/img/01HJF4AW00GCX1KCN8GGCN1R2A/7e10e2a68643.webp)

## Setting Condition Code

각 Data processing I에 condition code suffix를 쓰면:
- S suffix condition code 결과가 CPSR에 반영

### Multiplication

- **MUL r0, r1, r2** → `r0 = (r1 * r2)[31:0]` (하위 32bit만)
- **MLA r0, r1, r2, r3** → `r0 = (r1 * r2 + r3)[31:0]` (multiply accumulate)
- Multiply and accumulate: MLA r0, r1, r2, r3; MLAS (condition code 반영)

### Data-Transfer Instruction (Load/Store)

Register를 메모리에 load/store.

- **Single register load/store** (LDR, STR): byte/halfword/word
- **Multiple register load/store** (LDM, STM): block transfer

### Register Indirect Addressing

예: `LDR r0, [r1]` → `r0 = mem[r1]`, `STR r0, [r1]` → `mem[r1] = r0`

### Initializing an Address Pointer

Load from: **Load from label**; 현재 PC 기준.

![Condition Code, Multiplication, Data Transfer|full](/img/01HJF4AW00GCX1KCN8GGCN1R2A/c729491c455a.webp)

## Pointing label 1 and label 2

메모리 상의 labeled data에 pointer 설정 예제.

### Single Register Load/Store I

Base with offset, pre/post indexed auto-increment 예제를 통해 구조 이해.

## Base + Offset Addressing

- **Pre-indexed addressing**: `LDR r1, [r0, #4]` → `r1 = mem[r0 + 4]`, r0 변경 X
- **Pre-indexed with writeback**: `LDR r1, [r0, #4]!` → `r1 = mem[r0 + 4]`, `r0 = r0 + 4`
- **Post-indexed**: `LDR r1, [r0], #4` → `r1 = mem[r0]`, `r0 = r0 + 4`

![Base+Offset Addressing 예제|full](/img/01HJF4AW00GCX1KCN8GGCN1R2A/53cf922c7444.webp)

## Multiple Register Data Transfer

여러 register를 한 번에 load/store.

- **LDM (Load Multiple)**: `LDMIA r0!, {r1-r3}` → `r1 = mem[r0]`, `r2 = mem[r0+4]`, `r3 = mem[r0+8]`, `r0 = r0 + 12`
- **STM (Store Multiple)**: 동일하게 저장

### Addressing Mode

- **IA** (Increment After), **IB** (Increment Before), **DA** (Decrement After), **DB** (Decrement Before)
- Stack에서 쓰이는 경우: **FD** (Full Descending), **FA**, **ED**, **EA**

### Stack Addressing

- **Full stack** (F): SP가 마지막 저장된 값을 가리킴
- **Descending stack** (D): stack이 낮은 주소로 성장

ARM은 표준 **Full Descending (FD) stack**을 사용.

## Block Copy Addressing

`LDMIA/STMIA` 조합으로 메모리 block을 효율적으로 복사.

![Multiple Register Transfer 및 Stack|full](/img/01HJF4AW00GCX1KCN8GGCN1R2A/506f01bba97a.webp)

## Block Copy Addressing Usage

- **LDMIA r8!, {r0-r3}**: Load Multiple Increment After (4개 레지스터에 load, r8 update)
- **STMIA r9!, {r0-r3}**: Store Multiple Increment After
- → Block copy loop 구조의 핵심

비슷하게:
- **Full descending stack** (STMFD/LDMFD): Stack push/pop
- **Descending stack, full-empty check**

## Branch Instruction

- **Unconditional jump**: `B label`, `BL label`
- **Conditional jump**: `BEQ label`, `BNE label` 등

PC relative addressing, offset 24-bit → ±32MB 범위 jump 가능.

![Block Copy 및 Branch|full](/img/01HJF4AW00GCX1KCN8GGCN1R2A/7c1317385cae.webp)

## Branch Condition

| Branch | Interpretation | Flag state |
|--------|---------------|-----------|
| BEQ | Equal | Z=1 |
| BNE | Not equal | Z=0 |
| BCS/BHS | Carry set/unsigned high or same | C=1 |
| BCC/BLO | Carry clear/unsigned low | C=0 |
| BMI | Minus | N=1 |
| BPL | Plus | N=0 |
| BVS | Overflow set | V=1 |
| BVC | Overflow clear | V=0 |
| BHI | Higher (unsigned) | C=1 AND Z=0 |
| BLS | Lower or same (unsigned) | C=0 OR Z=1 |
| BGE | Greater or equal (signed) | N=V |
| BLT | Less than (signed) | N≠V |
| BGT | Greater than (signed) | Z=0 AND N=V |
| BLE | Less or equal (signed) | Z=1 OR N≠V |

## Conditional Execution

일반적인 방법으로 branch 할 때 conditional execution을 사용한다.

```
if (r1 > r0) { r0 = r0 - 2; } else { r0 = r0 + 2; }
```

일반적 branch 사용 시:
```
CMP r0, r1
BGT ELSE
SUB r0, r0, #2
B END
ELSE:
ADD r0, r0, #2
END:
```

Conditional execution 사용 시:
```
CMP r0, r1
SUBLE r0, r0, #2   ; condition이 LE이면 실행
ADDGT r0, r0, #2   ; condition이 GT이면 실행
```

→ **Branch를 없애 pipeline이 효율적**. ARM 특유의 강력한 기능.

![Branch Condition 및 Conditional Execution|full](/img/01HJF4AW00GCX1KCN8GGCN1R2A/f48f26eea0de.webp)

## Branch and Link Instructions

Subroutine 수행 후 원래 코드로 돌아올 때 많이 사용.

- **BL SUBR**: Branch to SUBR with Link
  - `r14 = r15 - 4` (return address 저장) → PC에 LR을 복원
- **B SUBR**: Branch only (no link)

### Nested Call일 경우

위와 같은 방법의 문제 발생 → **r14를 stack에 push해줄 필요가 있음** (subroutine에서 subroutine을 call할 경우).

예:
```
BL SUB1         ; branch to SUB1, r14 ← △ (△는 현재 PC + #4)
                ; return to here
SUB1 STMFD r13!, {r0-r2, r14}  ; save work & link regs
     BL SUB2    ; r14 ← □
     ...
SUB2 ...
```

→ △을 다 덮어쓰기에 △로 돌아갈 방법이 없음!

**해결 방법으로 STMFD I을 수행** → r14를 stack에 저장하고 덮어씀. **Full Descending** 방식. 나중에 stack에서 다시 꺼내서 돌아올 주소 가져옴.

## Subroutine Return Instructions

```
SUB2 ...
     MOV pc, r14    ; copy r14 into r15 to return

BL SUB1
SUB1 STMFD r13!, {r0-r2, r14}   ; save work regs/link → stack에 r14 저장
     BL SUB2
     ...
     LDMFD r13!, {r0-r2, pc}    ; restore work regs/link → 나중에 r14에서 가져옴
     ; MOV PC, r14를 생략하기 위해 r14 대신 PC를 씀
```

## Supervisor Calls (Privileged level에서 동작)

**Supervisor**: 특권 level에서 동작하는 프로그램, user level보다 높은 우선순위, system level I로 구현.

### SWI (Software Interrupt)

특별한 subroutine을 call하는 I (call 한 device를 수행해야 하는지 확인하는 코드).

예: r0의 bottom Byte를 display로 보냄:
```
SWI SWI_WriteC    ; output r0[7:0] → r0의 하위 첫 Byte 출력 (User level program이
                  ; 모니터에 대한 제어권을 가짐)
```

- `SWI SWI_Exit` — return to monitor (제어권한을 Monitor로 돌려줌)

![Branch and Link 및 Supervisor Calls|full](/img/01HJF4AW00GCX1KCN8GGCN1R2A/1418057b1f36.webp)
