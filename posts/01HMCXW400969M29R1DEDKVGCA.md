---
id: "01HMCXW400969M29R1DEDKVGCA"
slug: "memory-hierarchy-cache-design"
title: "메모리 계층 구조와 캐시 동작 원리: Memory Wall과 캐시 설계 4대 질문"
category: "01M651PC0078NBC62J3EZDXHDQ"
tags: []
summary: "메모리 계층과 캐시를 정리한 노트. Memory Wall과 시간·공간 지역성, 계층 용어, 캐시 설계의 네 가지 질문(배치·식별·교체·쓰기), 직접 사상 캐시 예제, 블록 크기와 미스율 관계를 다룬다."
thumbnail: "/img/01HMCXW400969M29R1DEDKVGCA/0767d231af20.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-19T04:32:01+09:00"
---
## 3장 Memory Hierarchy 개요

**Memory Hierarchy**는 "가장 빠른 메모리의 속도 + 가장 큰 메모리의 용량"이라는 가상 메모리 환상(illusion)을 만들기 위한 핵심 최적화. Program locality(Temporal & Spatial)를 활용해 작은 고속 메모리와 큰 저속 메모리를 조합한다.

## Memory Hierarchy: Solution to Memory Wall

### 메모리 시스템의 Big Picture

- **Memory**: Stores programs and data
- **Problem**: Memory is too slow and too small compared to CPU

```
       (fast, small, expensive)
            [ Register ]
              [ L1 $ ]
             [   L2 $   ]
           [  Main memory  ]
         [      Disk       ]
       [       Tape (slow, huge, cheap)     ]
```

- Speed: sub-ns → ms
- Size: 1 KB → TB

### Memory Hierarchy의 정의

> An optimization resulting from a perfect match between memory technology and two types of program locality.

- **Temporal locality**: 시간적 지역성. 한 번 참조된 데이터는 곧 다시 참조될 가능성이 높음
- **Spatial locality**: 공간적 지역성. 한 데이터에 접근하면 그 근처의 데이터도 곧 참조됨

**Goal**: Provide a "virtual" memory technology (illusion) that has an **access time of the highest-level memory** with the **size and cost of the lowest-level memory**.

![Memory Hierarchy 피라미드 및 locality 개념|full](/img/01HMCXW400969M29R1DEDKVGCA/0767d231af20.webp)

## Memory Hierarchy Terminology

- **Hit**: Accessed data is found in upper level (상위 레벨에 있음 — 성공)
- **Miss**: 상위 레벨에 없음, 아래 레벨에서 가져와야 함
- **Hit time**: 상위 레벨 access 시간
- **Miss penalty**: Miss 발생 시 아래 레벨에서 가져오는 추가 시간
- **Hit ratio (Hit rate)**: $\frac{\text{Hits}}{\text{Accesses}}$, Miss ratio = 1 - Hit ratio
- **Average memory access time (AMAT)**: 

$$\text{AMAT} = \text{Hit time} + \text{Miss rate} \times \text{Miss penalty}$$

## L0/L1 Cache 개요

- **Upper level**: SRAM (fast, expensive)
- **Lower level**: DRAM (large, slow, cheap)
- Goal: To provide a "virtual" memory technology
- Additional benefits:
  - Reduction of memory bandwidth consumed by processor
  - More memory bandwidth to I/O
  - No need to change the ISA
- **CPU 입장에서는 하나의 큰 메모리로 보인다 (Illusion)**

![Locality와 Cache 개념|full](/img/01HMCXW400969M29R1DEDKVGCA/1c4ab0391a66.webp)

## Four Big Cache Questions

- **Q1**: Where can block be placed in cache? → **Block placement**
- **Q2**: How can block be found in cache? → **Block identification**
- **Q3**: Which block should be replaced on a miss? → **Block replacement**
- **Q4**: What happens on a write? → **Write strategy**

### Block Placement — 3가지 방식

- **Direct-mapped**: 각 block이 캐시의 **한 자리**에만 올 수 있음. $$\text{Cache index} = \text{Block address} \bmod \text{Cache block 수}$$
- **Fully associative**: 어느 자리든 가능 (모든 slot 검색 필요)
- **Set associative**: 중간. 여러 set로 나뉘고 set 내에서는 fully associative. N-way set associative.

## Direct-Mapped Cache 상세

- Each memory block is mapped to a single block in cache
- Mapped cache block is determined by memory block address $\bmod$ number of blocks in cache
- Cache index: Block # (bit address)
- 결과 비교: tag와 block number로 hit/miss 판단 + valid bit

### Block Identification

- Every cache block has an address tag that identifies its location in memory
- Hit when tag and valid bits match + Miss otherwise
- What happens when a cache block is empty? → Mark this condition with a valid bit (0 = empty)

![Direct-mapped cache 구조|full](/img/01HMCXW400969M29R1DEDKVGCA/aa3222362135.webp)

### Direct-Mapped Cache 예제

16B block, 8 cache blocks, 5-bit index, 3-bit offset. Memory에서 Cache block index로 접근 → tag 비교 → Hit/Miss 판정.

Step by step:
- Address decoding: `[tag | index | offset]`
- Block address = Memory address / Block size
- Cache index = Block address mod Cache blocks
- Miss ratio는 temporal/spatial locality에 따라 달라짐

![Direct-mapped Cache 예제|full](/img/01HMCXW400969M29R1DEDKVGCA/37b6464cb02b.webp)

## Block Size Considerations

- **Larger blocks**: spatial locality 활용 감소, but latency 증가
- But in a fixed-size cache:
  - Larger blocks → fewer of them → more competition → miss rate 증가
  - Larger blocks → pollution (cache에 안 쓰이는 데이터가 들어옴)
- Larger miss penalty → can override benefit of reduced miss rate
- Early restart and critical-word-first can help

### Miss rate vs Block size

- 너무 작으면 spatial locality 활용 부족
- 너무 크면 cache 오염 + miss penalty 증가
- 중간 값이 optimal (보통 32~128B)

![Block size 및 Miss rate 관계|full](/img/01HMCXW400969M29R1DEDKVGCA/1516e743a350.webp)

## Cache Organization Spectrum

### Direct-mapped vs Fully associative vs Set-associative

- **Set-associative cache**: 미리 정해진 set 안에서는 fully associative
- **Four-way set associative**: 4개 set × N blocks
- For fixed cache capacity, higher associativity tends to **higher hit rate**

### Set-Associative Cache 구성

- Index → Set 선택 → Set 내 tag 비교 (fully associative)
- 4-way의 경우 동일 index로 4개 tag와 병렬 비교

![Set associative cache 구조|full](/img/01HMCXW400969M29R1DEDKVGCA/512d07df730a.webp)

## Memory Reference Sequence 예제

Cache initially empty, 1 hit for the third memory reference with the same capacity.

Step by step access pattern을 추적하며 각 cache의 **hit/miss** 카운트를 비교. Set-associative가 direct-mapped보다 hit 많음.

![Cache reference 예제|full](/img/01HMCXW400969M29R1DEDKVGCA/981a879ffacd.webp)

## Cache Read 동작

### On cache hit
CPU proceeds normally — 정상 작동.

### On cache miss (handled completely by hardware)
- Stall the CPU pipeline
- Fetch the missed block from the next level of hierarchy
- **Instruction cache miss** → Restart instruction fetch
- **Data cache miss** → Complete data access

### Instruction Fetch Flow (Instruction Cache Miss)

Clock Cycle 1에서 Compare, Cycle 2에서 Miss detected → Stall → Fetch completes → Pipeline restart. DRAM에서 읽어올 때까지 Stall.

### Load Instruction Flow (Data Cache Miss)

Data를 읽다가 miss가 나는 것은 Load instruction일 때만. Cycle 4에서 compare, Cycle 5에서 miss detected → Stall → Load completes → Pipeline restart.

![Cache read 및 miss handling|full](/img/01HMCXW400969M29R1DEDKVGCA/e03a1fe5b76c.webp)

## Cache Write 전략

### Write-Through

- Always write the data into both the cache and main memory
- Simple but slow — **increases memory traffic**
- Needs a **write buffer** (주메모리 쓰기를 비동기로)

### Write-Back

- Write the data into the cache only
- Update the main memory when a dirty block is replaced
- Requires a **dirty bit** (변경 여부 표시)
- Fast but complex to implement and causes a consistency problem

### Write Miss Handling

- **Write-allocate**: Miss 시 block을 캐시에 올리고 write
- **No-write-allocate**: Miss 시 바로 메모리에만 write (cache 건드리지 X)

일반적으로 Write-back은 Write-allocate, Write-through는 No-write-allocate와 조합.

![Cache Write 전략|full](/img/01HMCXW400969M29R1DEDKVGCA/3e7ae68e5ac9.webp)

## Cache Performance

### AMAT

$$\text{AMAT} = \text{Hit time} + \text{Miss rate} \times \text{Miss penalty}$$

$$\text{Average memory access time} = \frac{1}{N} \sum (\text{hit time} + \text{miss rate} \times \text{miss penalty})$$

### Improving Cache Performance

- **Decrease hit time**: Small & direct-mapped cache
- **Decrease miss rate**: Bigger cache, more associativity
- **Decrease miss penalty**: Multi-level cache, critical-word first, prefetching

### Current Cache Organizations

| Level | Cache size | Access time | Associativity |
|-------|-----------|-------------|---------------|
| L1 D-cache | 32 KB | 1 cycle | 4-way |
| L1 I-cache | 32 KB | 1 cycle | 4-way |
| L2 | 256 KB | 10 cycles | 8-way |
| L3 | 8 MB | 30 cycles | 16-way |
| Main memory | GB | 100+ cycles | - |

## Cache Coherence Problem

**Suppose two CPU cores share a physical address space**.

Write-through caches 예시:

| Time step | Event | CPU A''s cache | CPU B''s cache | Memory |
|-----------|-------|----------------|----------------|--------|
| 0 | - | - | - | 0 |
| 1 | CPU A reads X | 0 | - | 0 |
| 2 | CPU A writes 1 to X | 1 | - | 1 |
| 3 | CPU B reads X | 1 | 0 | 1 |

→ CPU B의 캐시가 stale(0) → **coherence problem**.

### Snoopy Protocol

- **Write-Broadcast**: Write to shared data → an invalidate is sent to all other caches
- **Write-Invalidate**: Write to shared data → broadcast on bus, processors snoop and update copies

![Multi-level cache 및 Cache coherence|full](/img/01HMCXW400969M29R1DEDKVGCA/151dfb4d289a.webp)

## Write Invalidate Protocol 상세

**Cache gets exclusive access to a block when it is to be written**:

- Broadcasts an invalidate message on the bus
- Subsequent read in another cache misses → Owning cache supplies updated value

예시 시나리오 (Write-back 사용):

| CPU activity | Bus activity | CPU A''s cache | CPU B''s cache | Memory |
|--------------|--------------|----------------|----------------|--------|
| - | - | - | - | 0 |
| CPU A reads X | Cache miss for X | 0 | - | 0 |
| CPU B reads X | Cache miss for X | 0 | 0 | 0 |
| CPU A writes 1 to X | Invalidate for X | 1 | X (invalid) | 0 |
| CPU B reads X | Cache miss for X | 1 | 1 | 1 |

→ Owning cache (A)가 최신 값을 supply.

## 정리

- **Memory hierarchies** are an optimization resulting from a perfect match between memory technology and two types of program locality
  - Temporal locality / Spatial locality
- The goal is to provide a **"virtual" memory technology** (illusion) that has an access time of the highest-level memory with the size and cost of the lowest-level memory
- **Cache memory** is an instance of a memory hierarchy
  - Exploits both temporal and spatial localities
  - **Direct-mapped** caches are simple and fast but have higher miss rates
  - **Set-associative** caches have lower miss rates but are complex and slow
  - **Multilevel** caches are becoming increasingly popular
  - **Cache coherence protocols** ensure consistency among multiple caches

![Write invalidate protocol 및 Summary|full](/img/01HMCXW400969M29R1DEDKVGCA/053c42f1460f.webp)
