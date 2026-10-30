---
id: "01HJM94A00G4VPKA2KHMFBKTYY"
slug: "radix-8-booth-multiplier"
title: "Radix-8 Booth 곱셈기 설계: 인코딩 테이블과 TDM 트레이드오프 (Verilog)"
category: "01M651PC00H3J4W7JXTZ32PR45"
tags: ["verilog","vivado"]
summary: "Partial Product를 줄인 고성능 곱셈기 설계"
thumbnail: "/img/01HJM94A00G4VPKA2KHMFBKTYY/af70d3cf8de5.webp"
note: false
created: "2023-12-27T09:00:00+09:00"
updated: "2026-09-15T02:10:29+09:00"
---
![Radix-8 Booth Encoding Multiplier using TDM|large](/img/01HJM94A00G4VPKA2KHMFBKTYY/af70d3cf8de5.webp)

Radix-8 Booth Encoding을 적용한 곱셈기에 TDM(Time Division Multiplexing)을 결합한 설계 프로젝트입니다. 기본 Booth 알고리즘의 확장판으로, 곱셈 연산의 하드웨어 효율을 극대화하는 것이 목표입니다.

## Booth 곱셈 알고리즘 개요

전통적인 이진 곱셈은 곱하는 수의 각 비트마다 자리이동(shift)과 조건부 덧셈을 반복합니다. 32비트 곱셈이라면 최대 32번의 덧셈이 필요하므로 느립니다. Booth 알고리즘은 연속된 1 비트열을 뺄셈과 덧셈의 조합으로 치환해 덧셈 횟수를 줄이는 방법입니다.

예를 들어 이진수 `0111`(10진수 7)을 곱할 때, 일반적으로 3번 더해야 하지만, Booth는 `1000 - 0001`(= 8 - 1 = 7)로 해석해서 한 번의 덧셈과 한 번의 뺄셈으로 처리할 수 있습니다. 이 원리를 확장한 것이 Modified Booth 알고리즘이며, Radix-2, Radix-4, Radix-8 등으로 발전해 왔습니다.

## Radix의 의미와 트레이드오프

Radix-N은 한 번에 몇 비트를 인코딩하느냐를 뜻합니다:

- **Radix-2**: 2비트씩 인코딩, 부분곱(partial product)이 N/2개
- **Radix-4**: 3비트씩 인코딩(2비트 + 이전 1비트), 부분곱 N/2개 (이전 대비 절반)
- **Radix-8**: 4비트씩 인코딩(3비트 + 이전 1비트), 부분곱 N/3개

Radix가 커질수록 부분곱의 수가 줄어 파이프라인 단계가 짧아지지만, 각 부분곱을 생성하기 위한 로직이 복잡해집니다. 특히 Radix-8부터는 3배(3×multiplicand) 같은 비정규 배수가 필요해 사전 계산이 요구됩니다.

## Radix-8 Booth 인코딩 테이블

3비트 윈도우 + 이전 1비트 = 4비트로 다음 8가지 중 하나의 연산을 선택합니다:

| 비트 패턴 | 연산 | 설명 |
|----------|------|------|
| 0000, 1111 | 0 | 아무것도 더하지 않음 |
| 0001, 0010 | +1× | multiplicand 덧셈 |
| 0011, 0100 | +2× | multiplicand × 2 (shift) |
| 0101, 0110 | +3× | multiplicand × 3 (사전 계산) |
| 0111 | +4× | multiplicand × 4 (shift) |
| 1000 | -4× | -multiplicand × 4 |
| 1001, 1010 | -3× | -multiplicand × 3 |
| 1011, 1100 | -2× | -multiplicand × 2 |
| 1101, 1110 | -1× | -multiplicand |

핵심은 **3× multiplicand** 생성입니다. 이는 `1× + 2×`로 계산되는데, 곱셈 시작 전에 한 번만 계산해두고 계속 재사용합니다.

## TDM (Time Division Multiplexing) 적용

TDM은 하나의 하드웨어 유닛을 여러 시간대에 다른 데이터로 공유하는 기법입니다. 이 프로젝트에서는 가산기(adder)를 TDM으로 공유해서 전체 하드웨어 면적을 크게 줄였습니다.

일반적으로 Radix-8 Booth 곱셈기는 N/3개의 부분곱을 Wallace Tree 같은 CSA(Carry-Save Adder) 트리로 합산합니다. 이는 빠르지만 면적이 큽니다. TDM을 적용하면 하나의 CSA 라인을 시간적으로 반복 사용해 면적은 줄이면서 처리량은 타임 슬롯 수로 나눈 만큼 감소합니다.

응용 분야에서 처리량이 극도로 중요하지 않거나, FPGA/ASIC 면적이 제한되는 경우에 유용한 설계입니다.

## 설계 방법

자세한 설명은 Radix-4에 있습니다. 설계 방법이 크게 다르지 않습니다.

[Radix-4 설명 보러가기](https://radic2682.github.io/project/hardware%20project/2023/12/27/Booth_multiplier_r4.html)

Radix-4와 비교한 주요 차이점:
- 인코더가 3비트 + 1 = 4비트 윈도우로 확장
- 3× 사전 계산 로직 추가
- 부분곱 수 N/2 → N/3으로 감소 (32비트 기준 16개 → 11개)

## 구현 및 검증

Verilog로 RTL 코드를 작성했고, 다양한 입력(양수/음수/극단값)으로 테스트벤치를 구성해 곱셈 결과를 검증했습니다. 특히 signed 곱셈에서 음수 입력 시 올바른 sign extension이 이루어지는지, overflow가 발생하는 경우 결과가 어떻게 처리되는지를 중점적으로 확인했습니다.

코드는 깃허브에서 보실 수 있습니다.

[깃허브에서 보기](https://github.com/radic2682/radix-8_Booth_Multiplier.git)

## 배운 점

- **면적과 속도의 트레이드오프**: Radix를 올리면 부분곱 수는 줄지만 인코더와 부분곱 생성기의 복잡도가 지수적으로 증가. 실제 프로젝트에선 Radix-4가 가장 널리 쓰이는 이유.
- **TDM의 효용**: 클럭 주파수가 충분히 높으면 TDM으로 면적을 크게 절약 가능. 저전력 IoT 칩에 적합.
- **Sign Extension의 중요성**: Booth 알고리즘에서 음수 부분곱 처리가 까다로움. 부호 확장 버그는 결과의 상위 비트에만 영향을 주기 때문에 쉽게 놓칠 수 있음.
- **사전 계산의 가치**: 3× 같은 값을 매번 계산하면 critical path가 늘어남. 한 번 계산해서 레지스터에 저장해두는 게 핵심.
