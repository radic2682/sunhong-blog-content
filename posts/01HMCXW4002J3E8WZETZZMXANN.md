---
id: "01HMCXW4002J3E8WZETZZMXANN"
slug: "computer-architecture-isa-performance"
title: "컴퓨터 구조 기초: ISA와 추상화 계층, 성능 평가 방법론"
category: "01M651PC0078NBC62J3EZDXHDQ"
tags: []
summary: "컴퓨터 구조 수업 도입부를 정리한 노트. 추상 자료형(ADT)과 스택 예시, 명령어 집합 구조(ISA), Hennessy와 Patterson의 엔지니어링 방법론과 두 가지 황금 규칙, 성능 평가 방법과 설계 기법을 다룬다."
thumbnail: "/img/01HMCXW4002J3E8WZETZZMXANN/76626e6d4927.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-19T04:27:12+09:00"
---
## 1장 Introduction — 컴퓨터 구조 개요

컴퓨터 구조(Computer Architecture) 수업의 도입 파트로, 추상 자료형(ADT), 명령어 집합 구조(ISA), 엔지니어링 방법론, 성능 평가 지표를 정리했다.

## Abstract Data Type (ADT)

**ADT**는 데이터 타입을 **독립적(independent)** 으로 정의하는 방식이다. 구현과 분리해서, 값의 집합(state)과 값에 대해 정의된 연산(operation)만 명시한다.

> A set of data values (state) and associated operations that are precisely specified **independent of any particular implementation**.

### 예시: Stack

Stack은 다음과 같이 정의된다.
- State: `push, pop, top`이 가능한 선형 자료구조
- Operation: `push(x)`, `x = pop()`, `x = top()`

ADT를 정의해 두면 구체 구현(배열, 연결 리스트 등)과 무관하게 논리적인 인터페이스로 사용할 수 있다.

![ADT와 ISA 개요|full](/img/01HMCXW4002J3E8WZETZZMXANN/76626e6d4927.webp)

## Instruction Set Architecture (ISA)

**ISA**는 소프트웨어와 하드웨어가 만나는 **계약(contract)** 이다. 프로세서가 이해할 수 있는 명령어의 집합, 레지스터 구성, 주소 지정 방식 등을 정의한다.

### 주요 가정

- **Registers** (범용 레지스터, PC 등), **Memory**: 프로그램과 데이터가 저장되는 공간
- 모든 명령어는 Before / After의 상태 변화를 야기한다.

### 예시 명령어 동작

메모리에 `(j 15)`, `beq r0, r1, 20`, `sw r2, 0(r0)`, `lw r2, 1(r0)`, `add r0, r1, r2` 같은 명령어가 저장되어 있을 때:

- **beq r0, r1, 2**: branch if equal. `r0 == r1`이면 PC + 2(offset), 아니면 다음 명령(`24`번지 instruction) 실행.
- **Jump j 15**: 15번지로 **무조건 점프**, PC가 15로 변경됨.

이처럼 각 명령어 실행 전후 레지스터와 메모리 상태를 정확하게 기술하는 것이 ISA의 역할이다.

## Engineering Methodology (Hennessy & Patterson)

> The discipline of a [computing] system as seen by the programmer, i.e. the conceptual structure and functional behavior, as distinct from the organization of the data flow and controls, the logical design, and the physical implementation.
> — Amdahl, Blaauw, and Brooks, 1964

### 두 가지 황금 규칙

- **Rule 1 — Identify and optimize the common case**: 흔히 발생하는 경우에 대해서 **빠르게 만들어야 함**
- **Rule 2 — Make the rare case correct and reasonably fast**: 자주 발생하지 않는 경우에 대해서는 **작동할 수 있기만 하면 됨**

즉 자주 쓰는 경로(common path)에 최적화 예산을 쏟고, 예외 처리는 정확성만 보장하자는 철학.

## Correctness Criteria

설계된 시스템이 **올바르게 구현되었는지 확인**하는 기준. 성능 최적화 이전에 반드시 만족시켜야 할 조건.

## Performance Evaluation Methods

### Performance Types

- **Time 관점**
  - Response time — 요청 후 응답까지 시간
  - Execution time — 프로그램 실행에 걸린 시간
  - Latency — 개별 연산 지연
- **Rate 관점** (시간당 하는 일의 양)
  - Throughput: MIPS, MFLOPS
  - Bandwidth: Mbps
- **Ratio 관점**
  - Relative performance — 시스템 간 상대 성능 비교 (time, rate 모두 기준 가능)

## Design Techniques

컴퓨터 시스템의 전체 설계는 아래 네 개의 큰 블록으로 나뉜다.

- **Processor** (I-cache, D-cache, Unified cache 포함)
- **Memory Hierarchy** (계층별 메모리)
- **Interconnection Network**
- **Input/Output and Storage**

이 블록들을 최적화하기 위한 대표 기법:
- **Sequential execution / Pipelined execution** — 순차 / 파이프라인 실행
- **Out-of-order execution / Speculative execution** — 비순차, 투기적 실행 (instruction을 축하는 것)
- **Cache Design** — I/D-cache, Unified cache
- **Cache Coherence, Synchronization, Interconnection network** — 멀티 프로세서 환경 최적화

→ 앞으로의 장에서 **Pipelining (2장)**, **Memory Hierarchy (3장)**, **Virtual Memory (4장)**, **Storage/IO (5장)**, **Multiprocessor (6장)** 를 하나씩 깊이 있게 다룬다.

![Engineering 방법론 및 설계 기법|full](/img/01HMCXW4002J3E8WZETZZMXANN/eb8b28702570.webp)
