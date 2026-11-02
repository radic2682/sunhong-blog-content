---
id: "01HMCXW400XENPTAGHV63S5B4X"
slug: "full-custom-standard-cell-pld"
title: "반도체 설계 방법론 비교: Full-Custom, Standard-Cell, PLD"
category: "01M651PC003HYJYHCQPHT4KAWF"
tags: []
summary: "반도체 회로 설계 공정과 SoC 설계 방법론을 정리한 노트. 사양부터 레이아웃, 검증, 제조까지의 단계와 Full-Custom·Standard-Cell·PLD 방식 비교, 내부·외부 IP와 Hard·Soft IP 통합을 다룬다."
thumbnail: "/img/01HMCXW400XENPTAGHV63S5B4X/930d9a263a37.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-19T23:15:11+09:00"
---
## 2장 Design Methodology 개요

**SoC (System-on-Chip)** 설계는 기존 ASIC과 달리 **IP(Intellectual Property) 재사용**, **Design Methodology 선택**, **Full-custom부터 Programmable logic까지의 스펙트럼**에 따라 달라진다. 이 장에서는 반도체 회로 설계 공정과 SoC 설계 방법론을 정리한다.

## Semiconductor Circuit Design Process

반도체 회로 설계는 다음 단계로 진행:

1. **System Specification**: 요구사항 정의 (목표 무엇을 할지)
2. **Architectural Design**: 아키텍처/블록 수준 결정
3. **Functional Design and Logic Design**: HDL(Verilog/VHDL) 코드 작성
4. **Circuit Design**: 트랜지스터 레벨 설계
5. **Physical Design**: Layout 결정 (면적, 배치)
6. **Physical Verification and Signoff**: DRC, LVS, STA 검증
7. **Layout Post Processing**: GDSII 변환
8. **Fabrication**: Fab에서 웨이퍼 제조
9. **Packaging and Testing**: 패키지, 테스트

## Design Methodology

### ① Full-Custom Design

- 모든 셀의 대로(배치), 소자 크기를 일일이 조절할 수 있음 (원하는 대로 만들 수 있음)
- 숙련된 노력이 더 개시 만들 수 있음
- **면적/성능 최적** but 시간과 비용 매우 ↑

### ② Standard-Cell Design

- 이미 디자인된 cell을 **블럭**으로 이용
- EDA tool은 standardized cell library를 지원해줌
- 가장 일반적인 ASIC 설계 방식

### ③ Programmable Logic

- FPGA가 대표적
- **장점**: 프로그래밍 가능, 빠른 시작점 얻음
- **단점**: 비싼, 대용량 저장X, 성공 졸업X (면적/성능 ASIC보다 낮음)
- Fab이 필요 X
- Programmable logic이 일어 해프 쓰는 단점

## Pitfall — 밑에서 출발하는 IP가 많은 분야에 필요하다

### ④ IPs (Intellectual Properties) Integration

- Third-party component을 이미 만들어진 걸 쓴 것
- 보통 SoC 설계는 licensing하여 processor core를 사용한다

## IP (Intellectual Property)

- 재사용을 가능한 HW description (logic, cell, chip layout design 등)
- 상품에 대한 IP로도 추가할 수 있음

### Internal IP

- 내부에서 개발

### External IP

- 3rd-party component로서 제공받음

### Hard IP

- 반도체 공정에 의존적 (layout도 만들어 놓아서 수정이 불가능)

### Soft IP

- HDL 코드로 주어지 flexible함 (원하는 대로 수정 가능)

## SoC Design

공급업체 ARM에서 pre-verified design of processor core 지원 + S/W chains and application 제공 (Compiler, debugger, OS, ...)

## 정리

| 항목 | 설명 |
|------|------|
| **Full-custom** | 모든 소자 직접 설계. 면적/성능 최고. 비용 ↑ |
| **Standard-cell** | Cell library 조합. 가장 일반적 |
| **Programmable (FPGA)** | 빠른 검증. 면적/성능 낮음 |
| **IP 기반 SoC** | Third-party core (ARM Cortex 등) + 자체 IP 조합 |
| **Hard IP vs Soft IP** | Layout 고정 vs HDL 수정 가능 |

SoC 설계의 본질은 **"IP를 어떻게 재사용할 것인가"**. ARM의 Cortex 시리즈가 SoC 시장의 표준이 된 이유는 **잘 검증된 soft IP + Compiler/OS 스택**을 한 번에 제공하기 때문이다.

![SoC Design Methodology|full](/img/01HMCXW400XENPTAGHV63S5B4X/930d9a263a37.webp)
