---
id: "01HMCXW400JGWPCWBX3H9F1T34"
slug: "multiprocessor-shared-memory-sync"
title: "멀티프로세서와 공유 메모리 동기화: 임계 구역과 Producer-Consumer"
category: "01M651PC0078NBC62J3EZDXHDQ"
tags: []
summary: "멀티프로세서 구조를 정리한 노트. UMA·NUMA 공유 메모리, 합 축소 예제, 생산자-소비자와 임계 구역 문제, Test-and-Set 상호 배제와 최적화된 동기화, 메시지 패싱과 클러스터, 상호 연결 구조를 다룬다."
thumbnail: "/img/01HMCXW400JGWPCWBX3H9F1T34/4fe26555a34b.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-19T05:14:12+09:00"
---
## 6장 Multiprocessor 개요

단일 프로세서의 clock frequency 한계에 부딪히면서 **여러 프로세서를 병렬로 사용하는 Multiprocessor 아키텍처**가 컴퓨터 구조의 주류가 되었다. 이 장에서는 Shared Memory / Message Passing, Cache Coherence, 동기화, Cluster 및 상호연결 토폴로지를 다룬다.

## Shared Memory Multiprocessor

```
[P1]──[Cache]──┐
[P2]──[Cache]──┼──[Interconnection Network]──[Memory]
[P3]──[Cache]──┘
```

- **UMA (Uniform Memory Access)**: 모든 프로세서가 메모리 접근 시간이 같음
- **NUMA (Non-Uniform Memory Access)**: 프로세서에 가까운 메모리 접근은 빠르고, 먼 메모리 접근은 느림
- Memory access time도 다름

### Example: Sum Reduction

100개의 프로세서가 100,000개 숫자를 합산하는 코드:

```c
sum[Pn] = 0;
for (i = 10000*Pn; i < 10000*(Pn+1); i = i + 1)
    sum[Pn] = sum[Pn] + A[i];
/* now add the sub-sums */
half = 100;
repeat
    synch();
    if (half%2 != 0 && Pn == 0)
        sum[0] = sum[0] + sum[half-1];
    /* Conditional sum needed when half is odd */
    half = half/2;  /* dividing line on who sums */
    if (Pn < half)
        sum[Pn] = sum[Pn] + sum[Pn+half];
until (half == 1);
```

Synchronization이 필요한 이유: 여러 processor가 같은 변수를 동시에 읽고 쓸 때 race condition 발생.

## Synchronization in Shared Memory

### 예: Producer-Consumer 문제

```c
Shared data
int counter = 0;
int buffer[BUFFER_SIZE];
int in = 0; int out = 0;

/* Producer */
while (counter == BUFFER_SIZE) ;  // do nothing
buffer[in] = next_produced;
in = (in + 1) % BUFFER_SIZE;
counter++;

/* Consumer */
while (counter == 0) ;  // do nothing
next_consumed = buffer[out];
out = (out + 1) % BUFFER_SIZE;
counter--;
```

`counter++`와 `counter--`가 atomic하지 않으면 **race condition** 발생.

예: `counter++` = `register1 = counter; register1 = register1 + 1; counter = register1;`

두 processor가 인터리빙되면 값이 손실될 수 있음 → **Critical Section** 문제.

![Shared Memory 및 Synchronization|full](/img/01HMCXW400JGWPCWBX3H9F1T34/4fe26555a34b.webp)

## The Critical-Section Problem

```
repeat
    Entry Section
       critical section  ← 한 번에 한 processor만 진입
    Exit Section
       remainder section
until false;
```

### Correctness Criteria for a Solution

- **Mutual Exclusion**: 한 critical section은 한 processor만 진입
- **Progress**: critical section에 들어가려는 processor가 있으면 결국 들어갈 수 있음
- **Bounded Waiting**: Critical Section에 진입하려는 processor는 유한 시간 내에 진입해야 함 (starvation 방지)

![Critical Section Problem|full](/img/01HMCXW400JGWPCWBX3H9F1T34/ae36348ceada.webp)

## Mutual Exclusion with Test-and-Set

```
repeat
    while Test-and-Set(lock) do no-op;   // Entry Section
        critical section
    lock := false;                       // Exit Section
       remainder section
until false;
```

- Lock 값이 1이면 no-op 수행 (loop, busy-wait)
- SW 명령어로 lock을 0으로 바꿈

### P0와 P1이 동시에 lock bit를 읽으면 문제 발생

동시에 lock bit가 0으로 읽히므로 문제. **Test-and-Set**은 atomic 연산으로 이를 방지 (사용하려고 하면 바로 lock bit를 1로 바꿈, 읽자마자).

→ **Mutual Section과 Progress는 만족하지만 Bounded Waiting은 만족하지 않음** (확률이 낮기에 구현을 안할 수도 있음).

## Naive Synchronization vs Optimized Synchronization

### Naive (Test-and-Set in a loop)

```
Try to lock variable using swap:
    read lock variable and then set
    variable to locked value (1)

Succeed? (= 0?)
    Yes → Begin update, Finish update, Unlock (set lock variable to 0)
    No  → loop
```

기능적인 문제는 없지만, **성능적인 문제** 발생:
- P0 사용 → lock bit: 1
- P1 Access 불가 → lock bit: 1 (나머지 Cache의 lock bit invalidate)
- P2 Access 불가 → lock bit: 1 (나머지 Cache의 lock bit invalidate)

→ **계속해서 Memory Access 문제 발생** (Interconnect 지연, 전력 소모).

### Optimized Synchronization (Load-first)

- Memory 사용 X → 전력에 좋음
- Test-and-Set 전에 **lock 값을 load**
- `lock: 1` → 계속 Load
- `lock: 0` → Test-and-Set

만약 Load할 때 0이었는데 Test-and-Set할 때 1이면 누군가가 먼저 빼앗아간 것.

- P0 사용 → lock bit: 1
- P1, P2 Access → Load (Cache의 lock bit: 1) → Cache에서 계속 Hit (메모리 접근 X)
- P0 사용 종료 → Lock bit: 0되고 invalidate protocol에 의해 다른 Processor Cache의 lock bit: 0
- P1, P2 Access → Load (Cache의 lock bit: 0) → Test-and-Set해서 P1과 P2 중 하나가 낚아챔

![Test-and-Set 및 Optimized Sync|full](/img/01HMCXW400JGWPCWBX3H9F1T34/3f779401e80d.webp)

## Message Passing

- 각각의 Processor가 각각의 Physical Address 공간을 가짐
- HW는 Processor 사이에 Message를 주고 받음

```
[Processor]─[Cache]─[Memory]  ...  [Processor]─[Cache]─[Memory]
          └──────── Interconnection Network ────────┘
```

### Loosely Coupled Clusters

- **독립적인 컴퓨터들을 잇는 Network** 필요 → I/O System 이용 (Ethernet/Switch/Internet)
- 각자 다른 Memory와 CPU
- 독립적인 task를 가지는 application에 적합 (Web server, databases, simulations …)
- 높은 **Availability, Scalable, Affordable**

→ **Shared Memory는 연결시킬 수 있는 Processor의 수 한계**가 있지만, Message Passing은 더 많은 Processor 이용 가능.

[문제점]: 관리 비용이 비싸고, Interconnect의 성능이 낮으면 전체 성능이 떨어짐 → **SMP (Symmetric Multiprocessor) 이용 → 고성능, 고대역 구현**.

### Sum Reduction in Message Passing

- ① 누군가가 더해야 하는 수를 1000개씩 나누어 주어야 함 (Shared Memory 방식은 나누어 줄 필요 없음)
- ② 각각의 Processor가 1000개의 수를 더함
- ③ 100개의 프로세서가 더한 결과를 합쳐줌 (절반은 주고 절반은 받아 더함) — 반복
- **동기화 과정이 필요 X** → Message를 주거나 받는 과정이 Synchronization 제공 (주거나 받는 Processor가 준비가 되지 않는다면 응답 X → 자연스럽게 동기화)

```c
limit = 100; half = 100;  /* 100 processors */
repeat
    half = (half+1)/2;  /* send vs. receive dividing line */
    if (Pn >= half && Pn < limit)
        send(Pn - half, sum);
    if (Pn < (limit/2))
        sum = sum + receive();
    limit = half;  /* upper limit of senders */
until (half == 1);  /* exit with final sum */
```

## Network Topology

### NUMA Topology 예

```
[Switch]─[Switch]─[Switch]─[Switch]
   │        │        │        │
[Switch]─[Switch]─[Switch]─[Switch]
   │        │        │        │
  ...
```

**2-D grid or mesh**, **3-D n-cube (hypercube)** 등.

- **Processing Elements (PEs)** 여러 개, Switch로 연결
- 가장 멀리 있는 노드까지 갈 수 있는 수 = $n$ (n-cube)

![Message Passing 및 Network Topology|full](/img/01HMCXW400JGWPCWBX3H9F1T34/352d6e88ea0f.webp)

## Crossbar & Multi-Stage Interconnection Network

### Crossbar

동시에 여러 쌍 연결 가능. 예: P0와 P3 연결 가능.

### Omega Network (Multi-stage interconnection network)

- 2×2 Switch 사용: If P가 16개라면 $\log_2(16) = 4$ → **4 stage 필요**
- 4×4 Switch 사용: If P가 16개라면 $\log_4(16) = 2$ → **2 stage**

```
Switch (한 쌍만 연결)
Crossbar (여러 쌍도 연결 가능)
```

## The Evolution-Revolution Spectrum of Computer Architecture

```
Evolutionary ────────────────────────────→ Revolutionary
Pipelining → Cache → Timeshared → Virtual → RISC → CC-UMA → CC-NUMA → Not-CC-NUMA → Message-passing → Massive SIMD → Parallel processing
```

- **가장 큰 발전**: Pipelining, Cache
- **CC (Cache Coherency)**: Cache coherency를 맞춰줌
- **SIMD**: Single Instruction Multiple Data — Instruction은 같지만 Data가 다름 (가장 예측에 많이 사용)

### RISC vs CISC

- **RISC (Reduced Instruction Set Computer)**: 수행 시간 동일, **현재 대부분 사용**. 장점: 효율적인 Pipelining.
- **CISC (Complex Instruction Set Computer)**: Instruction의 길이가 다름 (수행하는 시간이 다름). 예전에는 Memory가 비싸서 Instruction의 공간을 줄이기 위해서.

→ Intel Processor의 경우 HW는 RISC, SW는 CISC 타입 → 변환하여 사용 → **Application의 호환성을 위해**. 대부분은 RISC 타입.

![Crossbar, Omega, RISC vs CISC|full](/img/01HMCXW400JGWPCWBX3H9F1T34/7165d41b37e6.webp)

## 정리

| 개념 | 설명 |
|------|------|
| **Shared Memory MP** | 모든 processor가 같은 memory space 공유 (UMA/NUMA) |
| **Message Passing MP** | 각자의 memory, message로 통신 (cluster, HPC) |
| **Cache Coherence** | Multi-level cache 간 값 일치 유지 |
| **Synchronization** | Test-and-Set / Load-first로 atomic 연산 구현 |
| **Network Topology** | Crossbar, Omega, 2D-mesh, n-cube |
| **SIMD** | 한 명령어로 여러 데이터 동시 처리 (GPU의 기반) |

Multiprocessor는 이후 **Multi-core CPU + GPU + Cluster**로 이어지는 모든 병렬 컴퓨팅의 이론적 기반이 된다.
