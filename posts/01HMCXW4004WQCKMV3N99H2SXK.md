---
id: "01HMCXW4004WQCKMV3N99H2SXK"
slug: "cortex-m0-architecture"
title: "ARM Cortex-M0 구조: 레지스터 세트, 동작 모드, 메모리 맵"
category: "01M651PC003HYJYHCQPHT4KAWF"
tags: []
summary: "ARM Processor, Registers, Memory Map"
thumbnail: "/img/01HMCXW4004WQCKMV3N99H2SXK/9ee72b537977.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-20T00:39:40+09:00"
---
## 5장 Cortex-M0 개요

**ARM Cortex-M0**는 초저전력, 초소형 MCU를 위한 **ARMv6-M 아키텍처** 기반 프로세서 코어. 3-stage pipeline, Thumb-only 명령어 지원, 최소한의 NVIC으로 구성되어 IoT/센서 노드 등에 최적화되어 있다.

## Cortex-M0 특징

- **ARMv6-M architecture**
- **3-stage pipeline**: Fetch → Decode → Execute
- **Thumb instruction set** (16-bit) + 일부 Thumb-2 (32-bit)
- **~11,000 gates** (Cortex-M3보다 훨씬 작음)
- **~12 μW/MHz** 수준의 초저전력
- **Single-cycle multiplier** (option)
- **NVIC**: Nested Vectored Interrupt Controller (up to 32 interrupts)

## Cortex-M0 구조

```
┌────────────────────────────────────┐
│   Cortex-M0 Processor Core          │
│  ┌──────────┐   ┌──────────────┐   │
│  │ Register │   │  NVIC        │   │
│  │ File     │   │ (up to 32)   │   │
│  └──────────┘   └──────────────┘   │
│  ┌──────────┐   ┌──────────────┐   │
│  │ ALU      │   │  Debug (SWD) │   │
│  └──────────┘   └──────────────┘   │
└────────────────────────────────────┘
   AHB-lite bus → Memory, Peripherals
```

## Register Set

- **R0~R12**: 범용 register
  - R0~R7: Low registers (모든 Thumb I에서 직접 사용)
  - R8~R12: High registers (제한적 사용, MOV/ADD에서만)
- **R13**: Stack Pointer (MSP / PSP bank)
- **R14**: Link Register
- **R15**: Program Counter
- **xPSR**: Program Status Register

## Operating Modes

- **Thread mode**: 일반 프로그램 실행
- **Handler mode**: Exception handler 실행

Privilege:
- **Privileged**: 모든 기능 접근 가능
- **Unprivileged**: 일부 제한 (Cortex-M0는 항상 Privileged)

## Memory Map

$2^{32}$ B = 4 GB 가상 주소 공간:
- 0x00000000 ~ 0x1FFFFFFF: Code (0.5 GB)
- 0x20000000 ~ 0x3FFFFFFF: SRAM
- 0x40000000 ~ 0x5FFFFFFF: Peripheral
- 0x60000000 ~ 0xDFFFFFFF: External RAM/Device
- 0xE0000000 ~ 0xE00FFFFF: Private Peripheral Bus (NVIC, SysTick, SCB)

## Exception / Interrupt

- **Vector table**: 0x00000000에 위치
- 15 system exceptions + 최대 32 external IRQs
- Priority: Reset > NMI > HardFault > configurable

## Thumb Instruction Set

Cortex-M0는 **Thumb 명령어만** 지원. 일부 32-bit Thumb-2 (BL, DSB, DMB, ISB, MRS, MSR 등).

- 16-bit 명령어 → 코드 크기 절감
- Branch, data processing, load/store, control flow

## 3-Stage Pipeline

```
[Fetch] → [Decode] → [Execute]
```

- Branch prediction 없음 (M3 이상에서 도입)
- Branch taken → pipeline flush (1 cycle penalty)

## SysTick Timer

24-bit down counter로 간단한 time-base 제공. RTOS 스케줄러 tick에 자주 사용.

## Cortex-M0 vs M3/M4

| 특성 | M0 | M3 | M4 |
|------|----|----|----|
| ISA | ARMv6-M | ARMv7-M | ARMv7E-M |
| Thumb-2 | 제한적 | 전체 | 전체 + DSP |
| FPU | X | X | Option |
| Bit-band | X | O | O |
| MPU | X | Option | Option |
| Pipeline | 3-stage | 3-stage | 3-stage |
| Gates | ~11k | ~30k | ~50k |
| Use case | 센서, IoT | 범용 MCU | DSP, 모터 제어 |

## 사용 예

- STM32F0 series
- NXP LPC1100 series
- Nordic nRF51 series
- Atmel SAM D20

주로 **배터리 구동 IoT 기기**, **센서 노드**, **간단한 제어기**에 사용.

## 정리

| 항목 | 값 |
|------|---|
| **Arch** | ARMv6-M |
| **Pipeline** | 3-stage |
| **ISA** | Thumb + subset of Thumb-2 |
| **Register** | R0~R15, xPSR, MSP/PSP |
| **NVIC** | 최대 32 IRQ |
| **Memory** | 4 GB 선형 주소 공간 |
| **용도** | 저전력 IoT/sensor MCU |

Cortex-M0는 **가장 작고 저렴한 ARM core**로, 8-bit MCU 시장을 32-bit로 대체한 주역이다.

![Cortex-M0 구조|full](/img/01HMCXW4004WQCKMV3N99H2SXK/9ee72b537977.webp)
![Register Set|full](/img/01HMCXW4004WQCKMV3N99H2SXK/55007f150048.webp)
![Memory Map|full](/img/01HMCXW4004WQCKMV3N99H2SXK/894f445fb1b9.webp)
![Pipeline 및 NVIC|full](/img/01HMCXW4004WQCKMV3N99H2SXK/6cad65902737.webp)
