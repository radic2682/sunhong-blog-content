---
id: "01HMCXW4002M0NR0WHG16JXDQ1"
slug: "amba-apb-protocol"
title: "AMBA APB 프로토콜 분석: 신호 구성과 Read/Write 전송 타이밍"
category: "01M651PC008HMQK12XE50G198W"
tags: []
summary: "AMBA3 APB Protocol"
thumbnail: "/img/01HMCXW4002M0NR0WHG16JXDQ1/9095b7398912.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-19T22:05:50+09:00"
---
## 5장 AMBA APB 개요

**APB (Advanced Peripheral Bus)** 는 ARM의 AMBA(Advanced Microcontroller Bus Architecture) 표준 중 **저속, 저전력 주변장치(peripheral)** 를 위한 버스 프로토콜이다. AHB(고속 bus)와 AHB2APB bridge를 통해 연결된다.

## AMBA APB Protocol

### AMBA Family

- **AHB** (Advanced High-performance Bus): CPU, Memory 등 고속 시스템 버스
- **APB** (Advanced Peripheral Bus): UART, Timer, GPIO 같은 저속 주변장치
- **AXI** (Advanced eXtensible Interface): Out-of-order, multi-outstanding transaction 지원 고성능

### APB 특징

- **비파이프라인(non-pipelined)**: 간단한 setup + access phase만
- **Low-power**, **Low-performance**
- APB slave는 경량으로 구현 → peripheral에 적합

### Signals

- **PCLK**: bus clock
- **PRESETn**: active-low reset
- **PSEL**: slave 선택 (master가 활성화)
- **PENABLE**: 두번째 cycle (access phase)에서 high
- **PADDR**: address (32-bit)
- **PWRITE**: 1=write, 0=read
- **PWDATA**: master → slave 데이터 (write)
- **PRDATA**: slave → master 데이터 (read)
- **PREADY**: slave ready signal (wait state 조절)
- **PSLVERR**: error 응답

![AMBA APB Signals|full](/img/01HMCXW4002M0NR0WHG16JXDQ1/9095b7398912.webp)

## APB Read Transfer

### ① No Wait State

```
T0  T1  T2  T3  T4
PCLK   __/\__/\__/\__/\__
PADDR  __[Addr 1]________
PWRITE __[_______ 0 ]____
PSEL   __/‾‾‾‾‾‾\____
PENABLE ____/‾‾‾\____     ← Master가 가져감
PRDATA ______[Data 1]____  ← 데이터가 의미 있음
PREADY ______/‾‾‾‾\____
```

- T1: Setup (PSEL=1, PENABLE=0)
- T2: Access (PENABLE=1) → slave에서 data 준비
- T3: Master가 PRDATA를 sample → transfer 완료

### ② With Wait State

```
T0  T1  T2  T3  T4  T5  T6
PCLK   __/\__/\__/\__/\__/\__/\__
PADDR  __[____ Addr 1 _______]____
PSEL   __/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\____
PENABLE ____/‾‾‾‾‾‾‾‾‾‾‾‾\____  ← Master가 받을 준비
PRDATA ________[___ Data 1 __]__  ← 데이터 유효 X → 데이터를 줌
PREADY ____________/‾‾‾‾‾\____    ← 데이터를 받아감
```

- Slave가 준비 안되면 PREADY=0 유지
- PREADY=1일 때 data valid → master가 sample

## Operating States (APB Slave FSM)

APB slave는 **IDLE → SETUP → ACCESS** 3개 상태의 FSM:

```
       [IDLE]  (PSELx=0, PENABLE=0)
         │  ↑
         │  │ No transfer
         │  │
    Transfer (전송할게 있을 때)
         │
         ▼
      [SETUP]  (PSELx=1, PENABLE=0)
         │
         ▼
      [ACCESS]  (PSELx=1, PENABLE=1)
         │  ↑
         │  │ PREADY=0 (slave 준비 X)
         │  │
     PREADY=1 and transfer (계속 전송할게 있을 때)
         │
         ▼
      [SETUP] (or [IDLE] if no transfer)
```

- **IDLE**: 전송 없음
- **SETUP**: Master가 transaction 시작 (PSEL 활성화)
- **ACCESS**: 실제 data 전송 (PENABLE=1), PREADY를 기다림

## AHB to APB Bridge

고속 AHB와 저속 APB 사이 **Bridge**:
- AHB master → Bridge → APB slave
- Bridge가 주소 decode → 해당 APB slave 선택
- Wait state 삽입으로 clock domain/속도 차이 보상

![APB Read Transfer 및 Operating States|full](/img/01HMCXW4002M0NR0WHG16JXDQ1/a21b2bb13783.webp)

## Write Transfer

Read와 대칭. `PWRITE=1`, `PWDATA`에 write data 제공.

```
T0  T1  T2  T3
PCLK     _/\_/\_/\_/\_
PADDR    _[Addr]______
PWRITE   _[__1_]______
PSEL     _/‾‾‾\______
PENABLE  ___/‾‾\______
PWDATA   _[__Data__]__
PREADY   ___/‾‾‾\____
```

## APB vs AHB

| 특성 | AHB | APB |
|------|-----|-----|
| **Pipeline** | Address + Data phase 파이프라인 | Non-pipelined (2 cycle) |
| **Burst** | 지원 (INCR, WRAP) | 미지원 |
| **Multi-master** | 지원 (arbiter) | 단일 master (bridge) |
| **성능** | 고성능 | 저성능 |
| **용도** | CPU, DMA, Memory | UART, Timer, GPIO |

## 정리

| 개념 | 핵심 |
|------|------|
| **APB Setup phase** | PSEL=1, PENABLE=0 |
| **APB Access phase** | PSEL=1, PENABLE=1, PREADY로 완료 |
| **Wait state** | PREADY=0 유지로 transaction 지연 |
| **AHB2APB Bridge** | Master = AHB, Slave = APB peripherals |
| **FSM**: IDLE → SETUP → ACCESS | 2-cycle 기본 transaction |

APB는 단순하고 저전력이라 ARM Cortex-M 계열 SoC의 모든 peripheral (UART, SPI, I2C, Timer, GPIO 등)이 APB slave로 구현된다.
