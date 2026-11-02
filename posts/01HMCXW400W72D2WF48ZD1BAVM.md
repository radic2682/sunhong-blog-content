---
id: "01HMCXW400W72D2WF48ZD1BAVM"
slug: "barrel-shifter-comparator-multiplier"
title: "데이터패스 연산 유닛: 배럴 시프터, 비교기, 곱셈기 설계"
category: "01M651PC008HMQK12XE50G198W"
tags: []
summary: "프로세서 데이터패스의 연산 유닛을 정리한 노트. 배럴 시프터, 동등·크기 비교기, 배열 곱셈기와 Booth 인코딩, Wallace·Dadda 트리, 복원·비복원 나눗셈기의 구조와 면적·지연 특성을 다룬다."
thumbnail: "/img/01HMCXW400W72D2WF48ZD1BAVM/5477fdfa62b6.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-19T21:52:38+09:00"
---
## 3장 Datapath Functional Units 개요

프로세서의 **Datapath**를 구성하는 주요 기능 유닛(Functional Unit)들을 학습한다. Shifter, Comparator, Multiplier, Divider, Counter 등 ALU 내부의 핵심 구성 요소와 그 설계 기법을 다룬다.

## Shifter (시프터)

Shift 연산은 **곱셈·나눗셈의 빠른 근사**와 비트 조작에 필수. 주요 종류:

- **Logical shift** (LSL, LSR): 0으로 채움
- **Arithmetic shift** (ASL, ASR): sign bit 유지 (signed division)
- **Rotate** (ROL, ROR): 순환
- **Rotate through carry**: carry bit 포함 순환

### Barrel Shifter

n-bit 시프터를 **1 cycle 내에** 임의 shift 횟수로 처리. ARM의 특징적 유닛.

- MUX tree 구조: log n 단계
- 8-bit, 16-bit, 32-bit … 각 단계에서 shift 또는 통과
- Area: O(n log n), Delay: O(log n)

![Shifter 종류 및 Barrel|full](/img/01HMCXW400W72D2WF48ZD1BAVM/5477fdfa62b6.webp)

## Comparator (비교기)

두 수 A, B를 비교해 A=B, A>B, A<B를 판정.

### Equality Comparator

$A = B \iff \text{XNOR}(a_i, b_i) = 1 \text{ for all } i$

즉 각 bit XNOR → AND로 결합.

### Magnitude Comparator

- Cascaded: 상위 bit부터 비교, 같으면 다음 bit로
- Parallel: 각 bit 쌍 비교 후 priority encoder

![Comparator|full](/img/01HMCXW400W72D2WF48ZD1BAVM/d7694752df04.webp)

## Multiplier (곱셈기)

### Array Multiplier

각 partial product를 병렬로 생성 후 adder tree로 합산:
```
   A3 A2 A1 A0
×  B3 B2 B1 B0
───────────────
partial products 4개
   →  shift + add
```

지연 O(n), 면적 O(n²).

### Booth Encoding

Signed 곱셈에서 연속된 1 bit를 묶어 partial product 수를 줄임.
- **Booth-2 (radix-4)**: 3-bit씩 묶어 partial product가 절반
- **Booth-4 (radix-8)**: 4-bit씩 묶어 partial product가 1/3

![Booth Multiplier|full](/img/01HMCXW400W72D2WF48ZD1BAVM/d29eb1a3dd7a.webp)

### Wallace Tree Multiplier

Partial product들을 CSA (Carry Save Adder)로 줄이는 **reduction tree**:
- Depth = $O(\log n)$
- 최종 2개의 수를 CPA로 합산
- 현대 CPU의 정수 multiplier 표준

![Wallace Tree|full](/img/01HMCXW400W72D2WF48ZD1BAVM/603de6025bfb.webp)

### Dadda Tree

Wallace와 유사하지만 adder 개수가 약간 적음. 면적 더 효율적.

![Dadda Tree|full](/img/01HMCXW400W72D2WF48ZD1BAVM/0e1a92fc1562.webp)

## Divider (나눗셈기)

나눗셈은 곱셈보다 훨씬 복잡.

### Restoring Division

- 뺄셈 후 결과가 음수면 **원상 복구(restore)**
- n-bit division: n cycle

### Non-Restoring Division

- 원상 복구 없이 부호에 따라 다음 연산 결정
- 더 빠름, 현대 일반적 구현

### SRT Division

- Redundant digit set (−2, −1, 0, +1, +2)
- Radix-4, Radix-8 SRT divider → cycle 수 감소
- Intel Pentium의 유명한 FDIV bug가 SRT lookup table 오류

![Divider|full](/img/01HMCXW400W72D2WF48ZD1BAVM/35d15be823e2.webp)

## Counter

- **Binary counter**: n-bit ripple counter
- **Synchronous counter**: 모든 flip-flop에 같은 clock → skew 적음, high freq 가능
- **Johnson counter, Ring counter**: 시프트 레지스터 기반

![Counter|full](/img/01HMCXW400W72D2WF48ZD1BAVM/b8c70016e76e.webp)

## Multiplexer / Demultiplexer

- 2:1 MUX = `y = s·b + s̄·a` (AOI gate)
- n:1 MUX는 log n 선택 신호
- 대규모 MUX는 tree 구조로 분해

![MUX/DEMUX|full](/img/01HMCXW400W72D2WF48ZD1BAVM/cbf6fb2bca7e.webp)

## Register File

- 여러 포트(multi-port)를 가진 SRAM 기반 저장소
- 일반적인 RISC: 2 read port + 1 write port
- Out-of-order 프로세서: 8+ read, 4+ write port 등
- Area가 port 수의 제곱에 비례

![Register File|full](/img/01HMCXW400W72D2WF48ZD1BAVM/3b005730d194.webp)

## Memory Arrays (SRAM/DRAM)

- **SRAM**: 6T cell, 빠름 (0.5~1 ns), 비쌈, 휘발성
- **DRAM**: 1T1C cell, 느림 (수십 ns), 싸고 조밀
- Cache에는 SRAM, Main memory에는 DRAM

![Memory Arrays|full](/img/01HMCXW400W72D2WF48ZD1BAVM/e044769e65ae.webp)

## 정리

| Unit | 용도 | 주요 기법 |
|------|------|-----------|
| **Shifter** | Shift/Rotate | Barrel shifter (log n) |
| **Comparator** | 비교 | Parallel prefix |
| **Multiplier** | 곱셈 | Booth + Wallace tree |
| **Divider** | 나눗셈 | SRT (Radix-4/8) |
| **Counter** | 계수 | Synchronous |
| **MUX** | 선택 | Tree of 2:1 |
| **RegFile** | 저장 | Multi-port SRAM |

ALU는 이 functional unit들이 결합된 복합 회로. Datapath 성능은 이들의 **latency와 throughput의 조합**으로 결정된다.
