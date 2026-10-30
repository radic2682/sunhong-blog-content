---
id: "01HMCXW400TQKY39WXM2TAGQ3T"
slug: "storage-io-bus-mttf"
title: "스토리지와 I/O 시스템 구조: I/O 버스와 신뢰성(MTTF)"
category: "01M651PC0078NBC62J3EZDXHDQ"
tags: []
summary: "Dependability, Register Map, Polling, Interrupt, DMA, RAID"
thumbnail: "/img/01HMCXW400TQKY39WXM2TAGQ3T/f796d43c8aa0.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-19T05:06:47+09:00"
---
## 5장 Storage and Other I/O Topics 개요

컴퓨터 시스템에서 **I/O**는 디스크, 네트워크, 그래픽 등 외부 세계와 데이터를 주고받는 모든 장치를 의미한다. 이 장에서는 I/O의 기본 개념, Dependability(의존성), Interrupt/Polling/DMA의 전송 방식, RAID 구성까지 다룬다.

## Introduction

I/O 장치는 다음 세 가지 축으로 특성화된다.
- **Behavior**: input, output, input/output, or storage
- **Partner**: human, machine
- **Data rate**: byte/sec, transfer/sec

### I/O Bus Connections

```
[Processor]───[TLB / PA / Cache]
      │
[Memory-I/O Interconnect]
      │
┌─────┼─────┬──────┬─────────┐
Main  I/O    I/O    I/O       
memory controller  controller controller
      │     │      │
     Disk  Graphics  Network
          output
```

Interrupt는 event 발생 시 Processor로 전달된다.

### Diverse I/O Devices (Data rate 비교)

| Device | Behavior | Partner | Data rate (Mbit/sec) |
|--------|----------|---------|---------------------|
| Keyboard | input | human | 0.0001 |
| Mouse | input | human | 0.0038 |
| Voice input | input | human | 0.2640 |
| Sound input | input | machine | 3.0000 |
| Graphics display | output | human | 800~8000 |
| Network/LAN | input/output | machine | 100~10000 |
| Optical disk | storage | machine | 80~220 |
| Flash memory | storage | machine | 32~200 |
| Magnetic disk | storage | machine | 800~3000 |

## Dependability (의존성)

시스템이 **지정된 서비스**를 신뢰성 있게 제공하는 능력. 다음 두 상태 사이를 왕복한다.

```
Service accomplishment ↔ (Failure / Restoration) ↔ Service interruption
  (specified service)                              (deviation from specified)
```

- **Fault**: Component가 failure를 일으키는 것 → System failure로 이어질 수도/이어지지 않을수도 있음. System이 down될 수 있음.
- Noise, 공정상 문제 → Error. 대부분은 SW error가 많음.

### Dependability Measures

- **Reliability (신뢰도)**: failure까지의 시간 → **MTTF** (Mean Time To Failure)
- **Service Interruption**: failure 발생 후 고치는 시간 → **MTTR** (Mean Time To Repair)
- **MTBF** (Mean Time Between Failures) = MTTF + MTTR (failure 사이의 시간)
- **Availability (이용률)**:

$$\text{Availability} = \frac{\text{MTTF}}{\text{MTBF}}$$

### MTTF 향상 방법 — 예측하여 회피

- **Fault avoidance**: 피하기
- **Fault tolerance**: Fault가 나도 failure로 가지 않게 (여러 개 만들어 놓고 사용, redundancy)
- **Fault forecasting**: 예측

### MTTR 줄이는 방법

Failure를 진단하고 수리하는 툴이나 프로세서를 향상시키기.

![I/O Topics 및 Dependability|full](/img/01HMCXW400TQKY39WXM2TAGQ3T/f796d43c8aa0.webp)

## I/O Management

### I/O Commands — I/O Controller Registers

각 I/O 장치는 **command register, status register, data register**를 가진다. CPU가 이 레지스터에 read/write하여 장치를 제어한다.

- **Command register**: I/O controller에게 어떤 일을 할지 지시
- **Status register**: I/O 완료 여부, error 여부 등
- **Data register**: 주고받을 데이터 (read/write)

### I/O Register Mapping (메모리 입출력 매핑)

I/O controller register를 메모리 주소 공간에 매핑해서 **일반 load/store 명령어로 I/O 제어**. 이를 **memory-mapped I/O**라고 함.

- 메모리와 I/O register가 같은 address space 공유
- CPU 인터페이스 복잡도 감소

### Polling

CPU가 status register를 주기적으로 **check**해 I/O 장치의 상태를 확인.

- 장점: 구현 간단
- 단점: CPU resource 낭비 (다른 일 못함)

![I/O Management 및 Polling|full](/img/01HMCXW400TQKY39WXM2TAGQ3T/51aa95f7c93e.webp)

## Interrupts

Device의 ready나 error 발생 시 status register를 CPU가 체크 X, **interrupt controller가 체크**. Event 발생 시 CPU에게 알려줌.

### Interrupt vs Exception

Interrupt는 exception과 유사 (처리 방식이 유사하기 때문에). 주로 I/O에 의해 발생.

- Interrupt는 instruction의 수행과 동기화되지 않음 (현재 수행하는 것과 관계 X)
- Instruction 사이에서 handler가 개입 가능 (Instruction 수행 중 끊는 게 아닌, instruction이 끝나고 다음 instruction 수행 시 interrupt 처리)
- Interrupt signal은 한 개만 존재 → CPU가 interrupt를 발생시킨 것을 찾아 처리

### Priority Interrupt

긴급한 처리가 필요한 device들이 더 높은 우선 순위를 가짐. 낮은 우선 순위의 interrupt 수행 중에도 수행 가능.

> I/O 처리는 Interrupt 처리와 같다.

## I/O Data Transfer (Data register와 Memory 사이에서 일어나는 일)

### ① Polling & Interrupt-Driven I/O

CPU가 I/O와 Data register 사이에서 데이터를 읽고 씀 → **CPU가 수행하기에 시간을 많이 소모**.

예: Page fault 시 CPU → Command register → HDD I/O Controller (HDD Data → data R) → CPU (data R → Memory의 한 영역). 이 과정이 CPU 낭비 → **DMA**로 대신하게 됨.

### ② DMA (Direct Memory Access)

일종의 I/O, Command R & Status R 존재. But data R은 보통 존재 X.

HDD I/O Controller가 읽힌 데이터를 **data R → Memory의 한 영역으로 옮기는 역할**.

- OS가 DMA에게 Starting address, Size, 대상 I/O controller 정보를 DMA의 Command register를 통해 알려주면 DMA가 역할 수행
- **Special I/O Controller** (data R가 없기 때문)
- Error 발생 시, interrupt를 통해 CPU에게 알려줌 → interrupt controller가 있으면 그것을 통해

![Interrupts 및 I/O Data Transfer|full](/img/01HMCXW400TQKY39WXM2TAGQ3T/f2dae609c717.webp)

## DMA와 Cache/VM Interaction

### DMA/Cache Interaction

DMA가 cached location을 읽을 때, Memory는 최신이 아닐 수 있음 (cache에 dirty) → **Cache를 flush** 해야 함.

- DMA가 cached location에 write → Cache의 copy는 **stale** (무효화 필요)
- Cached location을 잘못 읽지 않도록 flushing 필요

### 해결

- DMA가 시작되기 전 해당 물리 메모리 영역을 **Flushing** (cache에 dirty data 있으면 memory로 write-back)
- DMA 끝난 후, **invalidate** (cache의 stale copy 무효화)
- Non-cacheable region으로 지정할 수도 있음

### DMA/VM Interaction

OS virtual memory 환경에서 DMA는 **physical address 기준으로** 동작.

- VM의 한 page가 여러 frame에 걸칠 수 있음 → DMA는 page 단위로 분할해서 전송해야 함
- Use multiple handlers / physical page도 여러 번 나눠서 처리

![DMA/Cache 및 DMA/VM 상호작용|full](/img/01HMCXW400TQKY39WXM2TAGQ3T/52bf09299297.webp)

## 전체 시나리오 — 프로그램 실행 중 DMA 동작

다음은 두 프로세스 P0, P1이 동작하고 DMA가 I/O를 수행하는 전체 흐름:

1. P0 실행 → Timer 설정
2. TLB Miss → exception → TLB 업데이트
3. Page fault → DMA에게 명령 전달
4. OS가 DMA command register에 명령 작성 (kernel → 쫓겨나지 X)
5. DMA 작업 중 Context Switching (P0 → P1)
6. P1 실행 (TLB miss 나지만 DRAM에 있기에 계속 사용)
7. DMA 일을 다 함 → Interrupt Controller가 OS에 알림
8. P1의 시간이 남아서 계속 사용
9. P1 시간이 끝나서 P0에 제어권 중 → TLB Hit → PA 변경 → Cache → Data 사용
10. OS가 명령 (SA+VA) → HDD I/O Controller → HDD

이런 식으로 DMA, Timer, TLB, Page table, Cache가 복합적으로 작동.

![전체 I/O 시나리오|full](/img/01HMCXW400TQKY39WXM2TAGQ3T/571cbf1fc459.webp)

## RAID (Redundant Array of Inexpensive Disks)

디스크 여러 개를 묶어 **신뢰성 + 성능**을 동시에 얻는 기법.

### RAID 0 (Striping)

- 데이터를 여러 disk에 나누어 저장 (stripe)
- 장점: 성능 ↑ (병렬 read/write)
- 단점: redundancy 없음 → 하나 고장나면 데이터 손실

### RAID 1 (Mirroring)

- 동일 데이터를 두 disk에 복제
- 장점: Reliability ↑ (하나 고장나도 다른 쪽 사용)
- 단점: 용량 절반, 비용 ↑

### RAID 2, 3 (이전 RAID)

- Bit-level / Byte-level parity (잘 안 씀)

### RAID 4 (Block-Interleaved Parity)

- Block stripe + 별도 parity disk 1개
- 읽기는 병렬 가능, 쓰기는 parity disk가 bottleneck

### RAID 5 (Block-Interleaved Distributed Parity)

- RAID 4의 Parity Disk를 제거 후, **모든 Disk에 나누어 저장**
- 병렬적인 service가 가능해짐
- 일부 경우에서만 가능, 비용은 RAID 4와 동일, Write의 성능 ↑ (RAID 4 < RAID 5)

예:
```
      0    1    2    3    4
Stripe Unit D0  D1  D2  D3  P
           D4  D5  D6  P   D7
           D8  D9  P   D10 D11
           D12 P   D13 D14 D15
           P   D16 D17 D18 D19
```

- Disk 0과 Disk 1 동시에 업데이트 가능 → 다시 Disk 0, 1, 3, 4 사용
- Disk 0과 Disk 4 동시에 업데이트 X (parity 충돌) → Disk 4 둘 사용 X

### RAID 6 (P+Q Redundancy)

- **2개의 parity**를 사용하여 2개의 error에 대해 복구해줌 (Reed-Solomon Codes)
- Parity P & Q → Data 업데이트 시 6번의 I/O가 필요
- 신뢰도가 중요한 환경에서 사용

**Hot swapping**: HDD 교체 시 전원을 끄지 않고 교체.

![RAID 0-4 및 RAID 5/6|full](/img/01HMCXW400TQKY39WXM2TAGQ3T/1f310a708eed.webp)
![RAID 5 및 RAID 6 상세|full](/img/01HMCXW400TQKY39WXM2TAGQ3T/72d11c375b79.webp)

## 정리

| 개념 | 핵심 |
|------|------|
| Dependability | Reliability, Availability — MTTF/MTTR/MTBF |
| Polling | CPU가 status register 주기적 검사 |
| Interrupt | Device 이벤트 시 CPU에 signal |
| DMA | CPU 없이 메모리-I/O 직접 전송 |
| RAID 0 | Striping — 성능 ↑ |
| RAID 1 | Mirroring — 신뢰성 ↑ |
| RAID 5 | Distributed parity — 성능+신뢰성 균형 |
| RAID 6 | P+Q redundancy — 2개 fault 허용 |

I/O는 CPU 관점에서 메모리 접근보다 훨씬 느리기 때문에 **CPU를 최대한 활용하기 위한 비동기 기법(interrupt, DMA)** 과 **다중 디스크를 활용한 신뢰성(RAID)** 이 핵심이다.
