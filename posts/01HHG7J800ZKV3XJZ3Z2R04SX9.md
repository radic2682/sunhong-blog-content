---
id: "01HHG7J800ZKV3XJZ3Z2R04SX9"
slug: "breadboard-computer-74ls"
title: "74LS 로직 칩으로 만든 8비트 브레드보드 컴퓨터: CPU 동작 원리 직접 구현"
category: "01M651PC00H3J4W7JXTZ32PR45"
tags: ["Hardware"]
summary: "Ben Eater의 설계를 따라 74LS 시리즈 TTL 칩만으로 브레드보드 위에 8비트 컴퓨터를 만들었다. 클럭, 레지스터, ALU, RAM, 프로그램 카운터, 제어 유닛을 직접 배선하며 CPU의 명령어 실행 사이클을 익힌 기록이다."
thumbnail: "/img/01HHG7J800ZKV3XJZ3Z2R04SX9/8789053c87d6.webp"
note: false
created: "2023-12-13T09:00:00+09:00"
updated: "2026-09-15T02:10:29+09:00"
---
![_bit Breadboard Computer|large](/img/01HHG7J800ZKV3XJZ3Z2R04SX9/8789053c87d6.webp)

유튜버 Ben Eater의 8-bit Breadboard Computer 프로젝트를 따라 만들어 본 결과물입니다. CPU가 어떻게 동작하는지 가장 낮은 레벨부터 직접 만들어보면서 학습하는 것이 목표였습니다.

<iframe width="560" height="315" src="https://www.youtube.com/embed/HyznrdDSSGM?si=iP3sNAIj2fq0FuJj" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## 프로젝트 개요

이 8-bit 컴퓨터는 74LS 시리즈 TTL 로직 칩만을 사용해서 완전한 범용 컴퓨터를 구현합니다. 마이크로컨트롤러나 FPGA 없이, 수십 개의 개별 논리 칩들을 브레드보드에 배선해서 모든 것을 직접 만듭니다. 완성된 컴퓨터는 8비트 데이터와 4비트 명령어 + 4비트 주소를 처리할 수 있습니다.

현대 CPU는 수십억 개의 트랜지스터로 구성된 블랙박스지만, 이 프로젝트를 통해 다음 질문들에 답할 수 있게 됩니다: 어떻게 전기 신호가 덧셈을 수행하는가? 명령어는 어떻게 해석되어 실제 동작으로 이어지는가? 클럭은 왜 필요한가? 레지스터와 메모리는 물리적으로 무엇이 다른가?

## 하드웨어 구성

전체 시스템은 공용 8비트 버스를 중심으로 각 모듈이 연결되는 구조입니다. 각 모듈은 버스에 값을 올리거나(out) 버스에서 값을 읽어오는(in) 제어 신호로 동작합니다.

### 1. 클럭 (Clock)

555 타이머 IC를 이용해 클럭 신호를 생성합니다. 가변 저항으로 주파수를 조절할 수 있고, 수동 모드(버튼으로 한 사이클씩 진행)와 자동 모드를 전환할 수 있게 설계되어 있습니다. 디버깅 시 수동 모드로 각 제어 신호와 버스 상태를 눈으로 확인할 수 있습니다.

### 2. 레지스터 (A, B, Instruction Register)

74LS173 4비트 D-플립플롭 칩 2개를 병렬로 연결해 8비트 레지스터를 만듭니다. 각 레지스터는 다음 신호로 제어됩니다:
- **Load**: 클럭 상승 에지에서 버스 값을 레지스터에 저장
- **Enable**: 레지스터 값을 버스에 출력 (3-state 버퍼)

IR(Instruction Register)는 명령어를 저장하며, 상위 4비트는 opcode, 하위 4비트는 피연산자 주소로 쓰입니다.

### 3. ALU (Arithmetic Logic Unit)

74LS283 4비트 이진 가산기 2개로 8비트 덧셈을 구현합니다. 뺄셈은 B 레지스터의 값을 XOR 게이트로 보수를 취한 뒤 Carry-in을 1로 만들어 2의 보수 연산으로 수행합니다(A - B = A + (~B) + 1).

ALU는 A와 B 레지스터를 피연산자로 받아 결과를 버스에 출력합니다. Zero 플래그와 Carry 플래그도 생성해서 조건 분기에 사용합니다.

### 4. RAM (Random Access Memory)

74LS189 16×4 SRAM 2개로 16바이트 RAM을 구현합니다. 프로그램 실행 전에 딥스위치로 주소와 값을 수동 입력할 수 있는 프로그래밍 모드도 설계되어 있어, 매번 프로그램을 직접 로드할 수 있습니다.

MAR(Memory Address Register)가 주소를 지정하고, RAM은 그 주소의 값을 버스에 출력하거나 버스 값을 그 주소에 저장합니다.

### 5. 프로그램 카운터 (PC)

74LS161 4비트 카운터 칩으로 4비트 PC를 구성합니다. 클럭마다 자동 증가(CE 신호)하며, 점프 명령에서는 버스 값을 PC에 로드(J)합니다.

### 6. 출력 (Display)

4자리 7-세그먼트 디스플레이에 8비트 값을 10진수로 표시합니다. EEPROM(28C16)을 이진-BCD 변환 룩업 테이블로 사용해, 8비트 값을 주소로 주면 해당 자릿수의 세그먼트 패턴이 출력됩니다. 4개 자릿수를 다중화(multiplexing)하기 위해 타이머로 빠르게 스위칭합니다.

### 7. 제어 유닛 (Control Logic)

이 프로젝트의 가장 복잡한 부분입니다. 28C256 EEPROM 2개를 제어 신호 생성기로 사용합니다. 입력은 opcode(4비트) + flags(2비트) + microstep counter(3비트)이고, 출력은 16개 제어 신호입니다.

즉 각 명령어는 여러 마이크로스텝으로 쪼개지고, 각 스텝마다 어떤 제어 신호를 활성화할지 EEPROM에 미리 구워둡니다. EEPROM 프로그래밍에는 Arduino Nano를 이용한 별도 라이터를 제작했습니다.

## 명령어 집합

16개 opcode를 지원합니다 (4비트):

| Opcode | 명령 | 동작 |
|--------|------|------|
| 0000 | NOP | 아무것도 안 함 |
| 0001 | LDA addr | RAM[addr] → A |
| 0010 | ADD addr | A + RAM[addr] → A |
| 0011 | SUB addr | A - RAM[addr] → A |
| 0100 | STA addr | A → RAM[addr] |
| 0101 | LDI imm | imm → A |
| 0110 | JMP addr | PC ← addr |
| 0111 | JC addr | Carry면 PC ← addr |
| 1000 | JZ addr | Zero면 PC ← addr |
| 1110 | OUT | A → Display |
| 1111 | HLT | 정지 |

## 명령어 실행 사이클

각 명령어는 공통 Fetch 2단계 + 명령별 Execute 단계로 구성됩니다.

**Fetch**:
1. T0: PC → MAR (다음 명령어 주소를 MAR로)
2. T1: RAM[MAR] → IR, PC++ (명령어 읽고 PC 증가)

**Execute (LDA의 경우)**:
3. T2: IR[3:0] → MAR (피연산자 주소)
4. T3: RAM[MAR] → A (데이터를 A로)
5. T4: (종료)

EEPROM 제어 신호 테이블에 이 마이크로코드가 전부 저장되어 있어, 하드웨어적으로 자동 실행됩니다.

## 구현 과정

<iframe width="560" height="315" src="https://www.youtube.com/embed/GaiMkFP2ASs?si=QIWrYL3DEOLzNcFR" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

브레드보드 6개에 수백 개의 점퍼 와이어로 배선했습니다. 각 모듈을 독립적으로 테스트한 뒤 차근차근 통합했습니다. 디버깅의 대부분은 배선 실수와 칩 불량 찾기였고, 오실로스코프와 논리 분석기가 필수였습니다.

## 테스트 프로그램

완성 후 피보나치 수열을 계산하는 프로그램을 작성해 실행했습니다. 메모리에 피연산자를 저장하고 ADD/STA/JMP 조합으로 반복문을 구현합니다. 초기값 0과 1을 번갈아 더하며 OUT 명령으로 7-세그먼트에 표시합니다.

## 배운 점

- **CPU의 본질**: CPU는 결국 공유 버스 + 레지스터 + 제어 신호의 조합이고, 제어 신호 타이밍만 정확하면 어떤 연산이든 가능함
- **마이크로코드**: 명령어를 마이크로스텝으로 쪼개는 설계가 현대 CISC CPU(x86 등)에서도 사용됨
- **클럭 동기화**: 모든 신호가 같은 클럭 에지에 동작해야 함. 비동기로 동작하면 race condition 발생
- **디버깅의 가치**: 논리 설계는 종이에 잘 그려도, 실물 배선에서 대부분의 버그가 나옴

이 프로젝트로 컴퓨터 구조 책에서 읽던 Fetch-Decode-Execute 사이클, 마이크로아키텍처, 제어 유닛 설계가 실제로 어떻게 회로로 구현되는지 체감할 수 있었습니다.
