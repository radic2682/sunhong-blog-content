---
id: "01KEFG49P82ANGB8RSC673PD8A"
slug: "arm-thumb-instruction-set"
title: "ARM 임베디드 시스템 입문: ARM과 Thumb 명령어 집합 비교"
category: "01M651PC00PPZ9M79TYD9S0526"
tags: []
summary: "ARM 임베디드 시스템 입문 노트. 임베디드 시스템의 정의와 ARM이 쓰이는 이유, ARMv4부터 도입된 16비트 Thumb 명령어와 32비트 ARM 명령어의 데이터 크기·특성 비교를 정리했다."
thumbnail: "/img/01KEFG49P82ANGB8RSC673PD8A/3e4386efee48.webp"
note: true
created: "2026-01-09T04:07:57+09:00"
updated: "2026-04-19T05:29:00+09:00"
---
## 1장 Introduction — ARM과 임베디드 시스템 개요

**임베디드 시스템(Embedded System)** 은 특정한 목적과 어플리케이션을 위해서 **하드웨어나 소프트웨어를 최적화시킨 시스템**이다. ARM은 임베디드 시스템에서 가장 유명한 프로세서로, 낮은 전력 소비, 작은 다이 사이즈, 높은 성능을 특징으로 한다.

## Data Sizes and Instruction Sets

ARM은 여러 세대를 거치며 다양한 데이터 크기와 명령어 집합을 지원한다.

- **ARMv4 이전**: 32-bit instruction 위주
- **ARMv4 이후**: **Thumb instruction set** 도입
  - 기존 32-bit ARM instruction + 16-bit Thumb instruction 동시 지원
  - Short physical registers, virtual address, physical address 등 설계 요소 정의

### ARM과 Thumb Instruction

- **ARM instruction**: 32-bit, full instruction set. Short physical registers, virtual address, physical address 등 사용
- **Thumb instruction**: 16-bit, ARM의 축약판. 코드 크기 절감을 위해 도입되어 메모리가 작은 임베디드 환경에서 유리

## Instruction Sets

### ARM Instruction Set

- 32-bit fixed-length instruction
- 각각의 Instruction은 같은 시간에 수행 가능 (RISC 특성)
- 파이프라인이 효율적으로 동작

### Thumb Instruction Set

Thumb instruction set은 ARM의 32-bit instruction을 **16-bit로 압축**한 것.

- **Variable-length instruction**: 16-bit이기 때문에 ARM과 혼용 가능
- 코드 크기가 작아지지만, 일부 instruction은 복잡한 연산 수행 시 여러 cycle 필요
- ARM은 Thumb 2로 넘어가면서 16/32-bit Thumb instruction을 지원하며 개선됨

### 기본 특성

- Thumb instruction이 추가된 이유: 임베디드 환경에서 **메모리 용량이 작기 때문에** 코드 공간 절감 필요
- 속도는 ARM instruction보다 조금 낮지만, 메모리 access가 적어져 전체 성능은 향상될 수 있음

![ARM/Thumb Instruction Set|full](/img/01KEFG49P82ANGB8RSC673PD8A/3e4386efee48.webp)

## Summary

- **ARM은 임베디드 시스템에서 가장 유명한 프로세서** (특정한 목적과 어플리케이션을 위해 하드웨어나 소프트웨어를 최적화시킨 시스템)
- **ARM은 RISC의 표현이 되어가고 있음** (작은 사이즈, 고성능) vs. 인텔 **CISC** (메모리를 작게 Access, Cache 이후 증가, 파이프라인 ↓, 명령어도 복잡)
  - **RISC (Reduced Instruction Set Computer)**: 단순하고 빠른 명령어
  - **CISC (Complex Instruction Set Computer)**: 복잡한 명령어, 더 많은 클록 cycle
- **Pipelining이 필수적** (RISC의 기본적인 테크닉)
  - Instruction을 중첩해서 동시에 수행 가능, 해저드의 위험이 있음
- **저전력은 필수적** (파워 vs. 성능) → 공정 개발로 leakage power 감소 중요

![ARM의 특징 요약|full](/img/01KEFG49P82ANGB8RSC673PD8A/4c99cba13219.webp)
