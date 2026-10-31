---
id: "01HMCXW4001E6CHQ8AA7GC0ES1"
slug: "crossbar-bus"
title: "크로스바 버스 구조: 공유 버스에서 Full Matrix까지 버스 아키텍처 진화"
category: "01M651PC008HMQK12XE50G198W"
tags: []
summary: "PL301 Structure, Arbitration Scheme"
thumbnail: "/img/01HMCXW4001E6CHQ8AA7GC0ES1/98ecc750a0f8.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-19T22:02:34+09:00"
---
## 4장 2절 PL Crossbar Bus 개요

**Crossbar Bus**는 여러 master와 여러 slave를 연결하는 고성능 interconnect 구조. Point-to-Point 연결을 동적으로 스위칭해 **다중 동시 전송**을 가능하게 한다.

## Bus의 진화

### Shared Bus

- 모든 device가 하나의 bus를 공유
- 한 번에 하나의 통신만 가능 → bottleneck
- 구현 간단, 낮은 대역폭

### Hierarchical Bus

- System bus → Local bus로 계층화
- AMBA AHB + APB 구조

### Crossbar (Full Matrix)

- M master × N slave → **M×N 스위치 매트릭스**
- 모든 master가 서로 다른 slave와 **동시에** 통신 가능
- 대역폭 ↑, 면적·복잡도 ↑

![Bus 진화|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/98ecc750a0f8.webp)

## PL Crossbar 구조

**PL (Programmable Logic) Crossbar**: 스위치 배열이 programmable하게 configured.

- **Master ports**: CPU, DMA, GPU 등
- **Slave ports**: Memory controller, Peripheral, I/O
- 각 교차점에 switch → arbitration + routing

![PL Crossbar 구조|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/f5320a7b2172.webp)

## Arbitration (중재)

다수의 master가 같은 slave로 요청 → arbiter가 선택:

- **Fixed priority**: 우선순위 고정
- **Round-robin**: 공평성
- **Weighted**: 대역폭 비율에 따라
- **QoS (Quality of Service)**: 실시간 요구

![Arbitration|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/0aa7f27680a6.webp)

## Pipelined Transaction

Bus transaction을 **Address phase**와 **Data phase**로 분리, pipeline화:
- Address와 Data가 중첩 → throughput 2배
- AMBA AHB, AXI의 기반

![Pipelined Transaction|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/cc37756c2bfa.webp)

## Burst Transfer

연속된 여러 word를 하나의 address phase로 전송:
- INCR4, INCR8, INCR16 (증가)
- WRAP4, WRAP8 (랩 어라운드, cache line fill용)

Cache line fill에 필수.

![Burst Transfer|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/fbb9ad97f74a.webp)

## QoS (Quality of Service)

실시간 시스템(영상, 오디오)에서 **Latency guarantee**:
- Deadline-based scheduling
- Bandwidth reservation
- Priority queues

AMBA AXI에서는 **AxQOS** signal로 구현.

![QoS|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/f8a1851b5c67.webp)

## Crossbar 확장 — NoC (Network-on-Chip)

Master/Slave 수가 많아지면 full crossbar 불가능 → **Network-on-Chip**:
- Mesh, Torus, Ring, Tree 등 토폴로지
- Router + Link 구조
- Packet-based routing

![NoC 확장|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/0bee37276607.webp)

## Clock Domain Crossing

Master와 slave가 다른 clock domain이면 **Asynchronous bridge** 또는 **FIFO** 필요:
- Metastability 방지
- Handshake protocol (req/ack)

![CDC|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/26b82bb454ed.webp)

## 정리

| 개념 | 설명 |
|------|------|
| **Shared Bus** | 하나의 공용 bus, 단순 |
| **Crossbar** | M×N switch matrix, 동시 다중 전송 |
| **Arbitration** | 충돌 시 master 선택 |
| **Pipelined/Burst** | 대역폭 향상 기법 |
| **QoS** | 실시간 보장 |
| **NoC** | 대규모 확장 interconnect |

Crossbar는 현대 SoC의 핵심. AMBA AXI, ACE, CHI 프로토콜 모두 crossbar/NoC 기반이다.

![Crossbar 종합|full](/img/01HMCXW4001E6CHQ8AA7GC0ES1/bea51ec818ce.webp)
