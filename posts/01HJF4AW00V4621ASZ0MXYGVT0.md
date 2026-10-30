---
id: "01HJF4AW00V4621ASZ0MXYGVT0"
slug: "arm-banked-registers"
title: "ARM 아키텍처와 레지스터 구조: 프로세서 모드별 뱅크드 레지스터"
category: "01M651PC00PPZ9M79TYD9S0526"
tags: []
summary: "ARM Register, CPSR, Instruction features"
thumbnail: "/img/01HJF4AW00V4621ASZ0MXYGVT0/b0034c8c2bdd.webp"
note: true
created: "2023-12-25T09:00:00+09:00"
updated: "2026-04-19T05:33:43+09:00"
---
## 3장 ARM Architecture 개요

ARM 아키텍처의 핵심 구성인 Programmer\'s Model, Visible Register, Mode, Exception 처리, Development Tools를 정리한다.

## Acorn RISC Machine → Advanced RISC Machine

ARM은 Acorn RISC Machine에서 출발해 Advanced RISC Machine으로 발전. **RISC이지만 몇몇 CISC의 특징을 가짐**.
- 예: ALU operation에서 **마지막 operand를 shift**, 곱셈과 덧셈을 한번에 하는 I (MLA) 등

## ARM Relationship Versions

- **Thumb I 지원**: Thumb I는 절반 공간 사용 (Thumb: 16-bit)
- ARM I, Data I → 32-bit processor/architecture 지원
- ARM I, Data I, Thumb I, Thumb decompressor 지원 (ARM v7 계열)
- ARM I, Data I, Thumb I, Thumb II

## Programmer's Model

Programmer에게 보이는 모든 것 — 상태(State) + 연산(Operation)
- **State**: 15 general purpose register, PC, memory
- **Operation**: Instruction
- **User code** ↔ **System code** 영역 구분

![ARM Architecture 개요|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/b0034c8c2bdd.webp)

## ARM의 Visible Register

ARM은 Programmer에게 보이는 register를 **Mode**에 따라 달리 보여준다.

### 주요 Register (User mode 기준)

- **R0 ~ R12**: 범용 register
- **R13 (SP)**: Stack Pointer
- **R14 (LR)**: Link Register (branch 시 return address 저장)
- **R15 (PC)**: Program Counter
- **CPSR (Current Program Status Register)**: 현재 mode, flag, interrupt mask 저장

### Mode별 Banked Register

- User mode: R0~R15, CPSR
- **Privileged modes** (FIQ, IRQ, Supervisor, Abort, Undefined): 해당 mode에서만 쓰는 **banked register**(R8~R14 일부) + **SPSR** (Saved Program Status Register)
- ARMv7이 들어가며 **sticky overflow** 등 추가

## Program Counter

- ARMv6 이전: PC는 **일반 register R15**. 32-bit aligned 주소.
- Pipeline 때문에 `Mov Pc, rn` 과 같이 이용하면 **jump (Branch)와 같이 사용**됨.

## CPSR (Current Program Status Register)

```
31 30 29 28    24   8 7 6 5   4   0
 N  Z  C  V    Q    I F T  M[4:0]
```

- **N**: Negative flag
- **Z**: Zero flag
- **C**: Carry flag
- **V**: Overflow flag
- **Q**: Sticky overflow (ARMv5TE+)
- **I**: IRQ disable
- **F**: FIQ disable
- **T**: Thumb mode flag
- **M[4:0]**: Processor mode

## Mode (Processor Mode)

- **User**: 일반 애플리케이션 실행 모드
- **FIQ (Fast Interrupt)**: High-speed data transfer or channel process. Banked register 많음 → fast
- **IRQ (Interrupt)**: General-purpose interrupt handler
- **Supervisor (SVC)**: OS / kernel mode. Reset, SWI 시 진입
- **Abort**: Memory abort (data/prefetch)
- **Undefined**: Undefined instruction handler
- **System** (ARMv4 이후 추가): User mode와 동일하지만 privileged → OS task 수행용

각 mode는 **CPSR, SPSR**과 일부 banked register를 가짐.

![Register, Mode, CPSR|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/88920d5e03b0.webp)

## Supervisor Mode / Load-Store Architecture

### Supervisor Mode

User mode에서 여행할 때 SWI (supervisor call)을 쓰면, OS 에서 진입. 시스템 자원을 사용해야 할 때 진입.

### Load-Store Architecture

- **RISC**의 특징: ALU 연산은 register-to-register만, 메모리 접근은 Load/Store로만
- Load/Store만 메모리와 register 사이 data 이동
- 대조: **CISC**는 메모리 연산도 ALU I에서 수행 가능

### ARM I: Data Processing I, Load/Store, Control Flow

- **Data processing I**: ALU, shift 등
- **Load/Store (transfer) I**: memory ↔ register
- **Control Flow**: Branch (B), BL, jump, conditional instructions

![SPSR, Memory System, ARM Little-endian|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/dd6d57e4c5ea.webp)

## ARM Instruction Feature

### Instruction Data Processing I

- 덧셈, 뺄셈, 곱셈과 같은 연산을 정리한 I
- 3개의 register를 사용하는 것이 일반적: `ADD r2, r1, r0`
- 2-operand form: `ADD r2, r1` (r2 ← r2 + r1)

### Load and Store I

- Post-indexing / Pre-indexing (auto-indexing)
- Multiple load/store: LDM, STM

### Open Instruction Set Extension

- Coprocessor I와 같이 사용
- Coprocessor를 interrupt로 관리하면 ARM이 대기
- Coprocessor가 ready bit interrupt 발행
- Coprocessor로 I를 수행할 수 없으면 Undefined I trap (Abort 모드 진입)

### Not Thumb I: Thumb 포함

ARM I 중 일부가 아닌 Thumb 가장 흔한 basic 포함.

## I/O System

Interrupt가 memory mapped registers를 모두 가짐.

- LDR/STR로 I/O controller 조작
- 대부분의 I/O device는 **Interrupt로 completion**을 알림
- Fast interrupt를 위해 **FIQ mode**
- **Local-sensitive and maskable**
- Interrupt를 통해 I/O 보고

## Exception

Memory abort, Undefined instruction, Reset 등 정상 흐름과 다른 상황.

### Exception의 종류

- **FIQ (Fast Interrupt Request)**: 가장 급히 처리해야 하는 interrupt → nFIQ를 low로 하여 생성
- **IRQ (Interrupt Request)**: 급하지 않은 interrupt → nIRQ를 low로 하여 생성
- **ABORT**: 메모리 access가 실패했을 때
- **Software Interrupt (SWI)**: User program이 좀 더 많은 권한이 필요해서 SWI (software interrupt instruction)에 의해 생성 → supervisor 권한으로
- **Undefined Instruction Trap**: ARM이 I을 인식할 때, 모든 I이랑 매칭 되는지는 coprocessor도 있음 → 먼저 coprocessor가 I을 수행하고 실패하면 coprocessor에서 처리 (coprocessor가 ready이나 interrupt 발생하면 ARM이 대기). Coprocessor가 I를 수행하지 못하면 Undefined I trap 발생 (Abort 모드 검사).

### Exception 우선순위

- Reset (highest priority) — 에러나서 가장 높음
- Data abort
- FIQ
- IRQ
- Prefetch abort (이 뒤의 I 알고리즘 하셔서, 데모리에 관여 있을시)
- Undefined Instruction, Software Interrupt

Exception 처리가 길면, 현재 processor state를 복원해야 함. 즉, 현재 돌고 있는 program의 exception과 복원 후 각 분야도 수 있어야 함.

### Exception Vector

| Address | Exception | Mode on entry |
|---------|-----------|--------------|
| 0x00000000 | Reset | Supervisor |
| 0x00000004 | Undefined instruction | Undefined |
| 0x00000008 | Software interrupt | Supervisor |
| 0x0000000C | Abort (prefetch) | Abort |
| 0x00000010 | Abort (data) | Abort |
| 0x00000014 | Reserved | Reserved |
| 0x00000018 | IRQ | IRQ |
| 0x0000001C | FIQ | FIQ |

→ Exception 발생 시 해당 주소로 PC가 강제로 분기 (점프 있는 branch, jump로 수행되어 처리)

### Entering an Exception 절차

1. User나 general mode 상태에서 작동 (다른 reg를 대신 reg 공유해서 user mode에만 영향력 수 있음 → 이에 regs가 여러 군데 있는 상태에서 어떤 활용변도 수 있음)
2. 돌아오는 위치의 현재 PC를 R14 (LR)에 저장
3. CPSR을 SPSR에 저장
4. Interrupt disable하여 연속 IPS set에 의해 Interrupt 처리 중 또 interrupt 방생하면 문제 생겨서
5. CPSR mode bit를 exception에 해당되는 값으로 변경
6. PC를 해당 Exception Vector로 이동 → Exception vector에는 branch I이 있고 jump로 해당 Exception Handler로 진입

![ARM Instruction Feature, I/O, Exception|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/d4f3ca547301.webp)
![Exception 종류 및 Vector|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/43d4e2f3649e.webp)

## Development Tools

### Example: User Mode vs FIQ Mode

- User: Standard R0~R15, CPSR
- FIQ: Banked R8~R14, SPSR → 빠른 switching
- Mode 전환 시 R8~R14가 banked register로 자동 교체

### ARM Development Tools

- **Debugger** (ARMsd): 소프트웨어 디버깅
- **ARMulator**: ARM Processor core를 SW적으로 모방하여 만든 시뮬레이터
- **ARM C Compiler**: ARM에 맞춰 C code를 compile
- **ARM Assembler and Linker**: object files 생성, link

### ARM C Compiler

- **ANSI C** standard 준수
- **Optimizations**: common sub-expression elimination, loop optimization 등

### ARM Assembler and Linker

- Object files들을 결합해 실행 가능한 final executable 생성
- Symbol table에서 instruction 주소를 resolve

![Example 및 Development Tools|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/aa356d4e32e9.webp)

## ARMsd (ARM Symbolic Debugger)

Debugging을 도와주는 **front-end interface**.

### 결과를 보기 위한

- **Under emulation** → ARMulator (잘 사용 X)
- **통신 interface** → A serial line (UART), JTAG interface (HW 문제까지 알 수 있음)

### 기능

- C debugger와 유사하게 동작: **breakpoint 설정 가능, 변수, reg, MEM을 볼 수 있음**
- Graphical Interface

## ARMulator

ARM Processor core를 SW적으로 모방하여 만든 **시뮬레이터**. 용어 정도만 양면됨.

### 정확도 단계

- **Instruction 정확도**: 일반적인 SW engineer에게 충분
- **Cycle 정확도**: 반도체 설계자에게 필요 (setup time, hold time이 길하잔지)
- **Timing 정확도**: 1 operation 단위 정확도, layout 설계자에게 필요

## Summary

- **ARM: Acorn RISC Machine → Advanced RISC Machine**
- ARM은 RISC이지만 몇몇 CISC의 특징을 가짐. 예: ALU operation에서 마지막 operand를 shift, 곱셈과 덧셈을 한번에 하는 I
- **Programmer\'s model**:
  - State: 15 general purpose reg, PC, memory
  - Operation: Instruction
- **ARM development tools**: Debugger, ARMulator

![ARMsd 및 ARMulator|full](/img/01HJF4AW00V4621ASZ0MXYGVT0/a78e55746098.webp)
