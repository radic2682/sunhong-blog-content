---
id: "01HMCXW4006TP0B7RZQA6WH3VF"
slug: "virtual-memory-paging"
title: "가상 메모리 동작 원리: 페이징과 가상주소-물리주소 변환"
category: "01M651PC0078NBC62J3EZDXHDQ"
tags: []
summary: "Paging, Page Table, TLB"
thumbnail: "/img/01HMCXW4006TP0B7RZQA6WH3VF/d807c1f2f996.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-19T04:35:32+09:00"
---
## 4장 Virtual Memory 개요

**Virtual Memory**는 실제 이용 가능한 것보다 더 큰 Physical memory를 사용할 수 있게 해주는 메모리 계층화 기법이다. 각 프로세스에 고유한 virtual address space를 제공해 **프로세스 간 격리**와 **protection/sharing**을 동시에 달성한다.

## 구현 방법

- **① Virtual Address 공간을 Page 단위로 쪼갠다**. 예: 32-bit 프로세서면 $2^{32}$-1의 가상 주소 공간을 page로 분할.
- **② 각 프로그램은 Page 단위로 access** (전체가 필요하지 않음)
- **③ Page 저장**: Physical memory가 허용하는 만큼 physical memory에 올리고, 나머지는 disk (HDD, SSD)에 저장
- **④ CPU는 virtual address를 사용하여 page access**

### 장점

- 프로그램이 실제 physical memory보다 **더 큰 메모리에서 돌아갈 수 있음**
- 여러 프로그램이 같은 메모리에 동시에 수행할 수 있게 함
- Virtual Address 공간은 일정(프로세서 아키텍처에 따름)하고 Physical Memory는 실제 내가 PC에 설치한 메모리에 따라 달라짐
- 프로그램 1개 당 1개의 Virtual Address space를 각각 가지게 됨

![Virtual Memory 개요|full](/img/01HMCXW4006TP0B7RZQA6WH3VF/d807c1f2f996.webp)

## Paging

### ① Virtual Memory

프로세서의 virtual memory 공간을 **같은 사이즈의 page로 나누는 것**.

- $p$: page number ($p_{\max}$ page)
- $o$: page offset ($o_{\max}$ B/page)
- Virtual address: $o_{\max} \times p + o$

예: Size of VA = 100B, Size of Page = 10B. 주소 29 = $10 \times 2 + 9$ → page number = 2, page offset = 9.

예: 8-bit processor, Size of Page = 16B. VA = `0010 0001` → page number = 2, page offset = 1 → $16 \times 2 + 1 = 33$번지 access.

### ② Physical Memory

동일한 frame 사이즈로 나눔.

- $f$: frame number ($f_{\max}$ frame)
- $o$: frame offset ($o_{\max}$ B/frame)
- Physical address: $o_{\max} \times f + o$

> Size of page = Size of frame → **page offset = frame offset**, **page number ≠ frame number**

Virtual memory에서 physical memory로 갈 수 있는 만큼 가고 나머지는 Disk에 있음.

### Virtual Address → Physical Address 변환

| VA (31-0) |
|-----------|
| Virtual page number (bits 31-12) + Page offset (bits 11-0) |

→ Translation → 

| PA |
|----|
| Physical page number (29-12) + Page offset (11-0) |

**page offset = frame offset** (같은 크기로 나뉘기 때문)

![Physical Memory 및 Address Translation|full](/img/01HMCXW4006TP0B7RZQA6WH3VF/6ef5a5a1f755.webp)

## Virtual Address Translation

> A **page table** maps virtual pages to physical frames.

CPU는 Virtual Address ($p, o$)를 생성 → Page Table로 $p \to f$ 변환 → Physical Address ($f, o$)로 메모리 접근.

### Page Table 구성

각 프로세서마다 하나의 테이블을 가짐.

**Flags** (1bit로 이뤄여 다음을 알려줌):
- Valid/invalid (= resident) bit: Page가 메인메모리에 있는지 없는지 (없으면 디스크에 존재)
- Dirty bit: 업데이트 여부
- Reference (= clock or used) bit

**Page frame number**: 변환 결과

### PTBR (Page Table Base Register)

- Page Table이 메모리에 어디에 있는지를 나타냄
- Page Table이 시작하는 부분을 가리킴
- **PTBR + p** → 해당 PTE(Page Table Entry) 위치

**Page의 개수**: 32-bit processor, size of page = 4KB → $2^{32} / 4\text{KB} = 2^{20}$개

### 중요 포인트

- **각 프로그램(프로세스)마다 하나의 Page Table을 갖는다**
- **Page Table은 DRAM에 저장된다**

![Page Table 구조 및 예제|full](/img/01HMCXW4006TP0B7RZQA6WH3VF/601b30498814.webp)

## Demand Paging

CPU가 해당 page를 읽을 때 비로소 main memory에 올림 (필요할 때).

### 장점

- 모든 page를 미리 물리 메모리에 올리지 않아도 됨 → 큰 프로그램 실행 가능
- 메모리 효율성

### 단점: External Fragmentation

Page의 연속된 영역이 필요할 때 외부 파편화 문제가 발생할 수 있음 (단 paging 자체는 fixed size로 나누기 때문에 external fragmentation은 거의 없지만 **internal fragmentation**은 발생).

## Translation Look-aside Buffer (TLB)

Address translation을 위해 매번 Page Table (in DRAM) access → Cache access → DRAM access가 필요 → **너무 느림**.

**Solution**: TLB (Translation Look-aside Buffer) = **page table의 cache**.

- Fully-associative cache
- Recent page 변환을 캐싱 → 대부분의 경우 TLB hit으로 DRAM page table access를 건너뜀
- TLB hit: Page table access가 필요 없이 바로 physical address 계산

![Demand Paging 및 TLB|full](/img/01HMCXW4006TP0B7RZQA6WH3VF/5a631ac23141.webp)

## A Big Picture (TLB + Cache + Memory)

전체 가상 → 물리 주소 변환 + 데이터 접근 흐름:

1. CPU가 VA 생성
2. TLB access → TLB Hit/Miss
3. TLB Miss → Page Table access → PT Hit/Miss
4. PT Miss → Page fault → Disk I/O로 page 가져옴
5. 물리 주소로 Cache access
6. Cache miss → Main memory access

### On TLB Miss

- TLB가 page table entry(PTE)를 찾기 위해 DRAM 접근
- Page table entry가 valid bit=1 → TLB 갱신, 주소 변환 성공
- Page table entry valid bit=0 → Page fault → OS에서 처리 (page fault handler)

### TLB Miss Handler

Page table을 메모리에서 가져와서 TLB 갱신.

![TLB 및 TLB Miss Handler|full](/img/01HMCXW4006TP0B7RZQA6WH3VF/8db6b607c05b.webp)

## Page Fault Handler

- **① 디스크에서 해당되는 page를 찾음**
- **② Main memory에서 대체해야 할 page 선택** (LRU 사용)
  - 만약 dirty bit이 1이라면 Main memory의 데이터를 먼저 Disk에 업데이트
- **③ Main memory로 page를 옮겨주고 page table을 업데이트** (PTE의 valid bit을 1로 만듦)
- **④ Process 다시 시작**

## Page Size 결정

### ① Page size를 작게 했을 때

- **장점**: Internal fragmentation problem이 적게 발생 → 메모리 효율 ↑
- **단점**: Page table이 커짐, page fault handling overhead가 더 자주 발생

### ② Page size를 크게 했을 때

- **장점**: Page table이 작아짐, page fault handling overhead가 덜 발생
- **단점**: Internal fragmentation problem이 자주 발생 → 메모리 효율 ↓

Instruction을 죽 읽어나가다가 경계에 마주하면 page fault 발생 → 발생빈도 ↑

## Paging — Protection and Sharing

### ① Protection

- Page 단위로 가능 (효율적인 Protection)
- 각 processor는 자신만의 address space를 갖기에 다른 processor로부터 보호

### ② Sharing

- 다른 프로세서가 같은 physical memory를 access할 수 있음 (같은 frame을 공유)

### ③ Basic Mechanism

- **User mode**: user address space를 access하는 권한을 가짐
- **Supervisor (kernel) mode**: OS가 access할 수 있는 영역 (모든 곳 access 가능)
- Page마다 권한을 정할 수 있음 (Read/Write)

### ④ System Call

- Processor가 수행 중 다른 권한이 필요할 때 OS에 요청 (exception의 한 종류)
- User → Supervisor (I/O 사용 시 요청)
- 권한이 필요하지 않아지면 Supervisor → User (과정반복)

![Page Fault Handler, Page Size, Protection|full](/img/01HMCXW4006TP0B7RZQA6WH3VF/9fd5dafda964.webp)

## Virtual Memory Performance

### Example

- Memory access time: 100 ns
- Disk access time: 25 ms (= 25,000,000 ns, page fault cost)
- Effective access time:
  - If $p$ = probability of a page fault
  - $1 - p$: normal access (100 ns)
  - $p$: page fault (100 + 25,000,000 ns)
  - If we want only 10% degradation → $p$가 매우 작아야 함 (미세한 fault ratio)

## LRU Approximation — Second Chance Algorithm

Clock (page replacement) + Reference bit.

- 각 페이지마다 reference bit이 있음
- Page 참조 시 reference bit을 1로 설정
- 교체가 필요할 때: clock pointer가 돌며, reference bit = 0인 page를 제거, reference bit = 1인 page는 0으로 리셋 후 다음으로 이동
- 거의 LRU와 비슷한 효과, 구현이 간단

![Virtual Memory Performance 및 LRU 근사|full](/img/01HMCXW4006TP0B7RZQA6WH3VF/d8baa4b35838.webp)

## I/O Interlock

**DMA (Direct Memory Access) Problem**: Page가 참조는 되지 않지만 I/O가 사용 중일 때 I/O가 작업 중인 데이터가 page fault가 난 경우 문제 발생.

**Solution**: **lock bit** 사용. Second chance algorithm이 돌다가 lock bit = 1이면 통과 (I/O가 사용 중).

## Segmentation with Paging

Paging을 하기 전에 비슷한 역할을 하는 것끼리 나눈다 → **Segmentation**.

```
 Heap         →   Heap
 Run-Time Stack →  Run-Time Stack
 Program Data →   Program Data, Libraries
 Program Text →   Program Text, User Code
```

- Sharing과 Protection을 조금 더 효율적으로 할 수 있음
- Virtual address = $(s \times p_{\max} + p) \times o_{\max} + o$
- $s$: segment number, $p$: page number, $o$: page offset

### Address Translation with Segmentation

- **STBR (Segment Table Base Register)** — Segment table이 DRAM의 어디서부터 시작되는지 알려줌
- Segmentation Table: Page table이 있는 위치를 알려줌 (어디서부터 시작하는지)
- TLB는 그대로 사용, s+p를 Tag로 사용

![I/O Interlock 및 Segmentation with Paging|full](/img/01HMCXW4006TP0B7RZQA6WH3VF/d5bf666972d8.webp)

### 장점

Protection, sharing이 segment 단위로 하는 게 page 단위보다 더 이득. Program text, shared segment 같은 단위로 공유 관리.

## Typical Memory Hierarchy (The Big Picture)

```
VA = [Virtual page number (20) | Page offset (12)]
        ↓
       TLB (Valid, Dirty, Tag, Physical page number)
        ↓ Hit
PA = [Physical page number | Page offset]
        ↓
  Cache (Valid, Tag, Data)
        ↓ Hit
  Data
```

![Shared Segment 및 Memory Hierarchy 전체|full](/img/01HMCXW4006TP0B7RZQA6WH3VF/0bbf725365a4.webp)

## 전체 흐름 — Flowchart

```
Virtual address → TLB access
  ├─ TLB hit? 
  │   ├─ No → TLB miss exception
  │   └─ Yes → Physical address
  │             ↓ Write?
  │             ├─ No → Try to read from cache
  │             │        ├─ Cache hit → Deliver data to CPU
  │             │        └─ Cache miss stall, read block
  │             └─ Yes → Write access bit?
  │                      ├─ Off → Write protection exception
  │                      └─ On → Try to write to cache
  │                               ├─ Cache hit → Write + update dirty bit
  │                               └─ Cache miss stall, read block
```

### TLB / Page Table / Cache 조합 가능성

| TLB | Page table | Cache | Possible? |
|-----|-----------|-------|-----------|
| Hit | Hit | Miss | Possible (TLB 있으면 PT 안 확인) |
| Miss | Hit | Hit | TLB miss, PT 있음, 캐시에 있음 |
| Miss | Hit | Miss | TLB miss, PT 있음, 캐시 miss |
| Miss | Miss | Miss | Page fault → 재시도 후에 캐시 miss |
| Hit | Miss | Miss | **Impossible**: TLB에 있는데 메모리에 없음은 불가능 |
| Hit | Miss | Hit | **Impossible** |
| Miss | Miss | Hit | **Impossible**: 메모리에 없는데 캐시에 있는 건 불가능 |
| Hit | Hit | Hit | **가장 자주 발생 (당연한 결과)** |

![Flowchart 및 조합 가능성 표|full](/img/01HMCXW4006TP0B7RZQA6WH3VF/639cfc625f4e.webp)

## 전체 요약 — Virtual → Physical Access 단계

- CPU → Virtual Address $(s, p, o)$
- TLB의 valid bit = 1인 것의 Tag 비교
  - **TLB Hit**: Physical page number + $o$ → Cache access (Tag + Index + Offset)
  - **TLB Miss**: Page table 접근 ((STBR + $s$ → Segment table) + $p$ → page table)
    - PT valid bit = 1 → PT Hit, TLB로 값을 옮겨주고 TLB 재접근
    - PT valid bit = 0 → PT Miss (Page fault), LRU 알고리즘을 통해 PT의 대체될 페이지 선택 (I/O 수행)
      - Dirty bit = 0 → Disk로부터 PT에 데이터를 옮겨주고 TLB로 값을 가져오고 TLB 재접근
      - Dirty bit = 1 → PT → Disk 후, Disk로부터 PT에 데이터를 옮겨주고 TLB로 값 가져오고 TLB 재접근

Cache access: Index로 찾아가 valid bit = 1인 것의 Tag 비교
- **Hit**: Data 사용
- **Miss**: Main memory로부터 Data를 옮겨줌 → Cache 다시 Access (dirty bit / LRU 알고리즘 이용)

![전체 VM → PA 흐름|full](/img/01HMCXW4006TP0B7RZQA6WH3VF/1e28ee58c298.webp)

## 시나리오 예제 — Context Switching

- P0가 CPU 사용 시 Timer를 맞춤 (예: 10 ms)
- TLB Miss시 exception 발생
- Fault나면 I/O Controller가 가져올 때까지 다른 프로그램에게 제어권을 넘김 (P1)
- Context Switching (valid = 0으로 다 설정): TLB Miss가 나지만 DRAM에 있기에 계속 사용
- I/O Controller가 시킨 일을 다 해서 (exception 발생)
- P1의 사용시간이 낮아지게 계속 사용, 후 P1의 사용시간을 다 사용 (Timer가 울림) → P0가 사용 Context Switching이 일어남
- Disk에서 옮겨갔기에 DRAM에서 옮겨서 사용

![Context Switching 시나리오|full](/img/01HMCXW4006TP0B7RZQA6WH3VF/2bae0280121d.webp)

## 정리

- **Virtual memory** extends the physical memory illusion, enables multi-programming, provides protection/sharing
- **Paging** splits memory into fixed-size pages / frames
- **Page table** maps virtual pages to physical frames (stored in DRAM, indexed by PTBR)
- **TLB** caches page table entries to avoid DRAM access on every memory reference
- **Page fault handler** swaps pages between disk and memory (LRU with clock reference bit)
- **Segmentation with paging** gives coarse-grained sharing/protection on top of paging
- Overall memory access: **TLB → Cache → Main Memory → Disk**
