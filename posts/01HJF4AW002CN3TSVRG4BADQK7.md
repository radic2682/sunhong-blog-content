---
id: "01HJF4AW002CN3TSVRG4BADQK7"
slug: "arm-3-stage-pipeline"
title: "ARM 프로세서 내부 구조: 3단 파이프라인과 명령어 실행 과정"
category: "01M651PC00PPZ9M79TYD9S0526"
tags: []
summary: "ARM Operation, Organization, Stages"
thumbnail: "/img/01HJF4AW002CN3TSVRG4BADQK7/4524c9b9278a.webp"
note: true
created: "2023-12-25T09:00:00+09:00"
updated: "2026-04-19T05:31:22+09:00"
---
## 2장 Organization & Implementation 개요

ARM 프로세서의 **Organization (구성)** 과 **Implementation (구현)** 을 학습한다. 3-stage pipeline ARM부터 5-stage ARM9까지의 진화, Principal Components, Data Forwarding, Branch, Memory Bottleneck을 다룬다.

## ARM의 Principal Components

- **Register file**: 32-bit 16개의 범용 레지스터. 일부는 PC(R15), LR(R14) 등으로 용도 고정
- **Barrel shifter**: 레지스터 값을 cycle 내에서 shift/rotate (ARM의 특징)
- **ALU**: Arithmetic & Logic Unit
- **Address register/incrementer**: 다음 fetch를 위해 PC 증가
- **Data register**: 메모리와 주고받을 데이터 보관
- **Instruction register**: 현재 실행 중인 instruction

## 3-Stage Pipeline ARM (ARM7 이전)

```
Fetch → Decode → Execute
```

각 stage가 한 cycle씩 걸리고 CPI는 1에 가깝다. Program counter는 항상 **현재 실행 중인 instruction + 8** (pipelining 때문).

### 3 Stages 설명

- **Fetch**: Instruction을 instruction register로 로드 → fetch 완료
- **Decode**: Instruction을 해석, register 읽기, control signal 생성
- **Execute**: ALU 연산, register에 결과 저장

![3-stage Pipeline ARM Organization|full](/img/01HJF4AW002CN3TSVRG4BADQK7/4524c9b9278a.webp)

## ARM Single-Cycle Instruction Operations

ARM의 single-cycle instruction 종류:
- **Data operation**: ADD, SUB, MUL 등 ALU 연산 → register에 저장
- **Addressing register R shifter**: barrel shifter를 통해 연산 (ARM 특징)
- **Store instruction**: register 값을 메모리로 (1 cycle)

### STR (Store Instruction) — Address Mode 4, 상대 주소로 쓰기

예: `STR r0, [r1, #4]` → r1 + 4 주소에 r0 저장

### How to Improve the Performance

성능 향상의 기본 공식:

$$T_{prog} = N_{inst} \times CPI \times T_{clk}$$

- $N_{inst}$: instruction 수 (컴파일러, 알고리즘에 의존)
- $CPI$: cycles per instruction (pipeline 설계)
- $T_{clk}$: clock 주기

→ Pipeline stage 수를 늘려 $T_{clk}$을 줄이면, pipeline 효율에 따라 성능 향상 가능. 그러나 hazard 증가.

ARM은 3-stage → 5-stage pipeline으로 확장. 더 빠른 clock과 더 많은 pipeline 단계로 최종적으로 RISC 기반의 효율적 pipeline을 구성.

![Single-Cycle 및 성능 향상|full](/img/01HJF4AW002CN3TSVRG4BADQK7/b03223585c6a.webp)

## Memory Bottleneck (Von Neumann Bottleneck)

- Memory bottleneck에 의한 **Structural Hazard** 빈번히 발생
- Instruction을 읽는 것과 Data를 읽는 것이 동시에 수행될 수 없기에 하나가 멈춰서 수행 → Hazard 유발

### Solution

**Memory bandwidth를 증가시킴**:
- 일반적인 방법: **Instruction과 Data Memory를 분리 (Harvard Architecture)**
- 1 cycle 당 access가 32-bit 이상을 전달해 주면 됨

## 5-Stage ARM Organization (ARM9)

```
Fetch → Decode → Execute → Buffer/Data → Write-Back
```

- **Stage 수의 증가** → $f_{clk}$ 증가
- **Instruction과 data memory 분리** → Memory bandwidth 2배
- **CPI 증가** (Stage의 수가 증가하기에 중첩해 수행할 수 있는 Instruction 증가)
  - Hazard 발생 확률의 증가이지만 나머지 이점이 더 크기에 상쇄 → $f_{clk}$ 증가, Memory bandwidth 증가

### 5 Stages 설명

- **Fetch**: I를 읽어와서 pipeline에 넣음
- **Decode**: I의 종류를 알고 준비 (register, Control Signal 준비)
- **Execute**: Operand shift, ALU 연산, 주소 연산
- **Buffer/Data**: Memory Access 혹은 by-pass됨
- **Write-Back**: 결과가 register에 저장

### Multiple Load/Store

여러 개의 I를 하나로 대체할 수 있는 I 사용 가능 → MUX 부분이 있는 이유

```
LDR r1, 0(r0)    연속적인 증가
LDR r1, 4(r0)    → 하나로 대체 가능
LDR r1, 8(r0)
LDR r1, 12(r0)
```

![Memory Bottleneck 및 5-Stage ARM|full](/img/01HJF4AW002CN3TSVRG4BADQK7/aaa077ce74dd.webp)

## Data Forwarding

5-stage pipeline에서는 throughput ↑, I가 동시에 수행.

- Stall이 없다면 **data dependency** 관계가 있음 (data Hazard)
- **Data Forwarding**으로 해결

```
reg bank → Mux → ALU → D-cache
              ↑     │
              └─────┘   internal forwarding
```

Mux를 통해 바로 나온 데이터를 사용할 수 있음.

### Load I의 경우

그러나 Load I은 internal forwarding을 이용해도 **1 stall이 필요**.
→ **Instruction reordering으로 해결 가능**.

예:
```
LDR r0, 0(r1)          LDR r0, 0(r1)
ADD r3, r0, r2    →    ADD r4, r5, r1
ADD r4, r5, r1          ADD r3, r0, r2
   (1 stall 필요)           (stall 필요 X)
```

## Data Processing Instruction

덧셈, 뺄셈, 곱셈과 같은 I의 처리 과정.

- **2개의 operand 필요**
- 하나는 항상 **register**, 다른 하나는 **register나 immediate value 가능** (8-bit로 표현 가능)
- **barrel shifter를 통과함** (ARM 특징)

```
ADD [reg]  [reg/imm]   (shift 옵션 포함)
```

- 2개의 operand는 ALU에서 연산해서 **register에 저장** (ARM 특징)
- Hazard가 없다면 1 cycle 동안 수행

### 예시

```
ADD r0, r1, r2 << 1   ← r2 LSL 1 (shift 이용)
```

- Data 연산: `data out`으로 나옴
- Decoder 생략

![Data Forwarding 및 Data Processing|full](/img/01HJF4AW002CN3TSVRG4BADQK7/9488b60e5d61.webp)

## Data Transfer Instruction

메모리와 Register 사이 데이터 이동 (Load/Store).

- **Level 1 (Load I) = Single mem cycle: Stage 1** (1 cycle register access)
- 데이터 읽는 경우: register ← memory[address]
- 데이터 쓰는 경우: memory[address] ← register

### Auto-Index (Pre/Post Index)

- **Pre-index**: `LDR r1, [r2, #4]` → address = r2 + 4, 메모리에서 읽음 (r2는 그대로)
- **Pre-index with writeback**: `LDR r1, [r2, #4]!` → address = r2 + 4, r2 = r2 + 4 (갱신)
- **Post-index**: `LDR r1, [r2], #4` → address = r2, 읽은 후 r2 = r2 + 4

이는 ARM의 특징인 **load/store with auto-indexing**을 지원.

### Single I → Auto-index Calculation

ARM은 load/store 시 **auto-indexing**으로 주소 계산을 효율적으로 처리. 다른 ISA에서는 load + add가 필요하지만 ARM은 하나의 instruction에서 처리.

## Branch Instruction

PC relative addressing, target address 계산.

- PC를 기준으로 **offset (signed)** 계산 → target address
- Target address = PC + offset

**단순 Branch** (B label):
- 1st cycle: target 계산 및 PC 갱신

**Branch with Link** (BL label):
- 1st cycle: branch target 계산
- 2nd cycle: return address를 **R14 (LR)** 에 저장
- 3rd cycle: LR 수정 (현재 PC의 다음번으로 돌아와야 하기에) → R14 = R14 + 4

![Data Transfer 및 Branch Instruction|full](/img/01HJF4AW002CN3TSVRG4BADQK7/dd416a15aa62.webp)
![Branch 1st/2nd Cycle 상세|full](/img/01HJF4AW002CN3TSVRG4BADQK7/e316bb572760.webp)

## 정리

- **Principal Components**: Register file, Barrel shifter, ALU, Address/Data register, Instruction register
- **3-stage (ARM7)** → **5-stage (ARM9)** 로 진화, $f_{clk}$ 증가 + Harvard architecture로 memory bandwidth 2배
- **Data Forwarding**: Mux를 통한 internal forwarding으로 대부분 data hazard 해결, Load-use는 1 stall 또는 reordering
- **Data Processing I**: Barrel shifter 통과, 2 operand(하나는 immediate 가능), ALU 연산 후 register 저장
- **Data Transfer I**: Auto-indexing (pre/post)으로 주소 계산 효율화
- **Branch**: PC-relative, BL은 3 cycle 필요 (target + LR 저장 + LR 수정)

Pipelining의 기본 원칙과 ARM 특유의 최적화 (barrel shifter, auto-indexing, multiple load/store)를 결합해 RISC의 정수를 구현한다.
