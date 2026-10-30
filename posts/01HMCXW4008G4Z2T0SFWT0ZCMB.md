---
id: "01HMCXW4008G4Z2T0SFWT0ZCMB"
slug: "soc-design-productivity-gap"
title: "SoC 설계 프레임워크와 Design Productivity Gap"
category: "01M651PC003HYJYHCQPHT4KAWF"
tags: []
summary: "SoC Design Framework, Flow"
thumbnail: "/img/01HMCXW4008G4Z2T0SFWT0ZCMB/7d309fd9fb01.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-19T23:16:17+09:00"
---
## 3장 SoC Design 개요

**SoC (System-on-Chip)** 설계는 **IP 재사용, Design framework, Methodology 선택**에 의존한다. 본 장에서는 SoC 설계 흐름, 요구 특성, MCU vs CPU vs MPU 비교, Apple SoC 계열을 정리한다.

## SoC Design Framework

### The Design Productivity Gap

반도체 공정의 복잡도 성장은 설계자의 생산성을 앞지른다. **Moore's Law**로 트랜지스터 수는 매년 폭발적으로 증가하지만, 설계자의 생산성은 그만큼 따라잡지 못함 → **Productivity gap**.

### Productivity 향상 방법

- **① Design Abstraction**: 상위 추상화로 작업 (behavioral → RTL → gate level)
- **② Design Automation**: EDA tool로 자동화
- **③ IP Packaging (Best Strategy)**: 이미 검증된 IP를 여러 곳에서 재사용
- **④ Design Reuse**: Open-source IP, 표준 IP 활용
- **⑤ Standard Tools and Platforms**: Standard interface (AMBA 등)

![SoC Design Framework|full](/img/01HMCXW4008G4Z2T0SFWT0ZCMB/7d309fd9fb01.webp)

## SoC 예시 — ARM-Based SoC

### Bluetooth Platform SoC 예

- ARM7 processor + OS and Support logic
- ARBITER, PLL (Lock Loop), shared memory → 기본 구성 요소

### SoC 구성

- **System Master**: CPU, DMA 등 transaction 시작자
- **System Slave**: Memory, Peripheral 등 수동적 응답자

## SoC 요구 특성

- **Low cost**
- **High performance**
- **Lighter footprint** (가벼움)
- **Lower efficiency** (전력 효율 ↑)
- **Less flexibility** (single-use하기 때문에, complexity 감소로 cost 감소)
- **Application specific** (대개 chip은 응용 분야에 특화)

## SoC vs MCU vs CPU

| 항목 | SoC | MCU | CPU |
|------|-----|-----|-----|
| **CPU** | 있음 | 있음 | 있음 |
| **Memory** | 별도 board (external) | 내장 ROM/RAM | 별도 |
| **Peripheral** | 내장 다수 | 내장 (Timer, UART 등) | 별도 |
| **용도** | 모바일/embedded system | 소규모 embedded | 범용 desktop/server |
| **예시** | Apple A-series | STM32, PIC | Intel x86, ARM Cortex-A |

## SoC Design Flow

SoC 설계는 다음 흐름으로 진행:

1. **Design specs**: 요구사항 정의
2. **Partitioning**: HW/SW 구분 결정
3. **HW co-verification**: Virtual platform에서 HW-SW 통합 검증
4. **Software dev**: Compiler, OS, app 개발 병행
5. **Packaging**: GDSII → Fab → Package → Test

![SoC 요구 특성 및 Design Flow|full](/img/01HMCXW4008G4Z2T0SFWT0ZCMB/4bac483e0d06.webp)

## ARM 계열 구분

- **A (Application)**: 고성능 (모바일/태블릿 AP)
- **R (Real-time)**: 실시간 시스템 (자동차, HDD)
- **M (MCU)**: 저전력 마이크로컨트롤러

## Apple SoC Families 예

| SoC | Model No. | CPU | CPU ISA | Technology | Die size | Date | Devices |
|-----|-----------|-----|---------|-----------|----------|------|---------|
| N/A | APL0098 | ARM11 | ARMv6 | 90 nm | N/A | 6/2007 | iPhone, iPod Touch (1st gen) |
| A4 | APL0398 | ARM Cortex-A8 | ARMv7 | 45 nm | 53.29 mm² | 3/2010 | iPad, iPhone 4, Apple TV (2nd gen) |
| A5 | APL0498 | ARM Cortex-A9 | ARMv7 | 45 nm | 122.6 mm² | 3/2011 | iPad 2, iPhone 4S |
| A5X | APL5498 | ARM Cortex-A9 | ARMv7 | 45 nm | 162.94 mm² | 3/2012 | iPad (3rd gen) |
| A6 | APL0598 | Swift | ARMv7s | 32 nm | 96.71 mm² | 9/2012 | iPhone 5 |
| A6X | APL5598 | Swift | ARMv7s | 32 nm | 123 mm² | 10/2012 | iPad (4th gen) |
| A7 (64-bit) | APL0698 | Cyclone | ARMv8-A | 28 nm | 102 mm² | 9/2013 | iPhone 5S, iPad mini (2nd gen) |
| A7 | APL5698 | Cyclone | ARMv8-A | 28 nm | 102 mm² | 10/2013 | iPad Air |

### Die Size — 반도체 칩 사이즈 (면적)

→ 공정이 미세화될수록 기능을 많이 넣으려고 하다보니 사이즈가 커짐.

A7에서 Apple은 **64-bit ARMv8-A**를 업계 최초로 스마트폰에 적용. Cyclone 코어는 수직통합된 Apple 실리콘의 시작점.

![Apple SoC Families|full](/img/01HMCXW4008G4Z2T0SFWT0ZCMB/236fd5ee5aa1.webp)

## 정리

| 개념 | 핵심 |
|------|------|
| **Productivity Gap** | 공정 복잡도 > 설계 생산성 → IP 재사용으로 극복 |
| **SoC** | CPU + Memory + Peripheral을 한 칩에 |
| **Low cost, High perf, Low power** | SoC 기본 요구사항 |
| **ARM A/R/M** | 용도별 Cortex 시리즈 구분 |
| **Apple A-series** | iPhone/iPad용 Cortex 기반 SoC, A7부터 64-bit |

SoC 설계의 성공은 **IP 재사용**과 **HW-SW 공동 검증**에 달려있다.
