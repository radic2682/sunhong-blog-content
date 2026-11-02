---
id: "01HMCXW400BBRYRW0ZQKV7NN9E"
slug: "cmos-transistor-basics"
title: "CMOS 트랜지스터 동작 원리: PN 접합부터 nMOS/pMOS까지"
category: "01M651PC008HMQK12XE50G198W"
tags: []
summary: "CMOS 기초를 정리한 노트. IC와 VLSI, 실리콘과 PN 접합, nMOS·pMOS 트랜지스터 동작과 스위치 모델, CMOS 인버터와 NAND·NOR 같은 복합 게이트, CMOS가 전력을 아끼는 원리와 공정 단면을 다룬다."
thumbnail: "/img/01HMCXW400BBRYRW0ZQKV7NN9E/0a1508be335b.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-19T16:15:59+09:00"
---
## 1장 CMOS 개요 — Complementary Metal Oxide Semiconductor

**CMOS**(Complementary Metal Oxide Semiconductor)는 현재 마이크로프로세서와 반도체 회로의 표준 기술로, **Fast, Cheap, Low power**라는 세 가지 핵심 특성을 갖는다. 본 노트에서는 CMOS 공정부터 nMOS/pMOS 트랜지스터 동작, 스위치 모델, 그리고 CMOS 게이트 설계까지 다룬다.

## IC와 VLSI

- **IC (Integrated Circuit)**: 하나의 실리콘 칩 위에 다수의 트랜지스터, 저항, 커패시터를 집적
- **VLSI (Very Large Scale Integration)**: 초대규모 집적 — 수백만 개 이상의 트랜지스터를 하나의 칩에 집적

### 기술 변화

- **1970년대**: nMOS만 사용 → 사용하지 않을 때도 power 소모 (정적 전력 소비)
- **1980년대 이후**: **CMOS 채택** → 사용하지 않을 때 power 거의 소모 X (CMOS의 가장 큰 장점)

## Silicon과 P-N Junction

Silicon은 4가 원소. 순수 Si 결정에 **5가 원소(인, 비소 등)** 를 도핑하면 자유 전자가 남아 **n-type**이 되고, **3가 원소(붕소 등)** 를 도핑하면 정공이 남아 **p-type**이 된다.

### P-N Junction

p-type과 n-type을 접합하면 **다이오드**. 한 방향(anode → cathode)으로만 전류가 흐른다.

## nMOS Transistor

- **Gate-oxide-body는 Capacitor와 동일** (4개의 terminal: G, S, D, B)
- Gate와 Body는 도체, SiO₂는 좋은 절연체

### nMOS 동작

**① OFF (Gate Low)**:
- `Source, Gate, Drain — Polysilicon, SiO₂ layer`
- Body는 0V → S-B, D-B 사이의 다이오드 OFF → 전류 안 흐름

**② ON (Gate High)**:
- Gate에 high 전압 인가 → Body는 (-)로 반전 → **channel 형성** → S와 D 사이 전류 흐름

## pMOS Transistor

nMOS와 반대 동작.
- Body는 **V_DD**
- **Gate Low → Transistor ON**
- **Gate High → Transistor OFF**

## Power Supply Voltage

- GND = 0V, V_DD = 5V (전통적)
- → 최근에는 **낮은 V_DD 이용** (transistor에 데미지 줄이고 power 절약): 예) 3.3, 2.5, 1.8V 등

## Transistors as Switches

CMOS 설계에서 트랜지스터는 **스위치**로 추상화:

| g=0 | nMOS | pMOS |
|-----|------|------|
| OFF | ON | - |

| g=1 | nMOS | pMOS |
|-----|------|------|
| ON | OFF | - |

nMOS는 **g=1일 때 ON (0 전달 잘함)**, pMOS는 **g=0일 때 ON (1 전달 잘함)**. 이 특성이 CMOS 상보성의 핵심.

![CMOS, Silicon, nMOS/pMOS Transistor|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/0a1508be335b.webp)

## CMOS Inverter와 기본 게이트

CMOS Inverter (NOT 게이트):
```
       V_DD
        │
       [pMOS]
        │
   A ───┼──── Y = NOT A
        │
       [nMOS]
        │
       GND
```
- A=0 → pMOS ON, nMOS OFF → Y=V_DD (1)
- A=1 → pMOS OFF, nMOS ON → Y=GND (0)

### CMOS의 절전 원리

- 정적 상태(A가 바뀌지 않을 때): pMOS, nMOS 중 하나만 ON → 전류 경로가 **끊김** → **static power = 0**
- 트랜지스터 switching 시에만 short current 발생

이것이 nMOS-only 대비 CMOS가 압도적으로 전력이 낮은 이유.

![CMOS 기본 게이트|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/89b1eba19dbf.webp)

## NAND, NOR, XOR 등 복합 게이트

CMOS는 **pull-up network (pMOS)**, **pull-down network (nMOS)** 로 구성:

- Pull-up: `Y`를 V_DD에 연결 (pMOS)
- Pull-down: `Y`를 GND에 연결 (nMOS)

두 네트워크는 **서로 상보(complement)** 관계.

- NAND: pull-down = A·B (직렬), pull-up = A+B (병렬)
- NOR: pull-down = A+B (병렬), pull-up = A·B (직렬)

![Pull-up/Pull-down 및 NAND 게이트|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/96d46fb7b525.webp)

## CMOS 공정 단면

- P-substrate 위에 N-well을 만들고, 그 안에 pMOS의 P+ source/drain
- N-substrate 바깥쪽에 nMOS의 N+ source/drain
- 각 영역은 polysilicon gate, SiO₂ insulator, metal interconnect로 연결

![CMOS 공정 단면|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/cdc35af5d507.webp)

## Noise Margin과 전압 전달 특성 (VTC)

CMOS 논리회로의 **Voltage Transfer Characteristic**:
- V_OL: Output low 레벨
- V_OH: Output high 레벨  
- V_IL: Input low threshold
- V_IH: Input high threshold
- **Noise Margin high** = V_OH - V_IH
- **Noise Margin low** = V_IL - V_OL

CMOS는 swing이 rail-to-rail (0~V_DD)이라 noise margin이 매우 좋음.

![VTC 및 Noise Margin|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/e309956b0562.webp)

## Propagation Delay (전파 지연)

- **t_pHL**: 출력이 H→L로 바뀔 때까지 걸리는 시간
- **t_pLH**: 출력이 L→H로 바뀔 때까지
- **t_p** = (t_pHL + t_pLH) / 2

지연 시간은 **부하 커패시턴스 C_L**과 **충전 전류 I**에 의해 결정:

$$t_p \approx \frac{C_L \cdot V_{DD}}{2 I_{avg}}$$

![Propagation Delay|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/ea187bd372f1.webp)

## Power Dissipation

CMOS의 전력 소모:

$$P = P_{switching} + P_{short-circuit} + P_{leakage}$$

- **Switching (dynamic)**: $P = \alpha \cdot C_L \cdot V_{DD}^2 \cdot f$
- **Short-circuit**: 스위칭 중 PMOS, NMOS가 동시 ON인 순간
- **Leakage (static)**: 공정 미세화 시 증가 (중요)

### Power 줄이기

- V_DD 감소 (가장 효과적 — 제곱)
- Clock frequency 감소
- Capacitance 감소 (layout 최적화)
- Leakage 제어 (threshold voltage 조절)

![Power 소모|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/1ce26baf0a95.webp)

## Sizing (크기 조절)

트랜지스터의 W/L 비율이 **driving strength**를 결정:
- W(채널 폭) 증가 → current 증가 → faster
- L(채널 길이) 증가 → current 감소 → slower, lower leakage

### Balancing pMOS / nMOS

- pMOS의 hole mobility < nMOS의 electron mobility (약 2~3배)
- 같은 driving strength를 위해 **pMOS의 W를 2~3배 크게**

![Transistor Sizing|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/d85cfc0295f4.webp)

## Layout Design

- Metal1, Metal2 등 다층 배선
- Contact, Via로 layer 간 연결
- Design rules (minimum spacing, overlap, etc.) 준수

![Layout 및 설계 규칙|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/18fa960347d9.webp)

## Interconnect Delay

미세 공정에서는 wire delay가 gate delay를 초과. RC delay 모델:

$$t_{wire} = R_{wire} \cdot C_{wire}$$

- Wire length ↑ → delay 제곱으로 ↑
- Buffer insertion, wire sizing으로 최적화

![Interconnect Delay|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/806c05f76c7c.webp)

## Clock Distribution

Clock은 모든 flip-flop에 **동시에** 도달해야 함. H-tree, clock grid 등으로 skew를 최소화.

![Clock Distribution|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/d4afd48b7325.webp)

## Latch와 Flip-Flop

- **Latch**: level-triggered. Clock이 active level에 있을 때 transparent
- **Flip-Flop**: edge-triggered. Clock의 rising/falling edge에서만 데이터 캡처

CMOS 표준 flip-flop은 master-slave latch 2단 구성.

![Latch, Flip-Flop|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/db7945db6b5a.webp)

## Setup Time, Hold Time

- **Setup time (t_su)**: clock edge 이전에 D가 안정되어야 하는 최소 시간
- **Hold time (t_h)**: clock edge 이후에 D가 유지되어야 하는 최소 시간

Timing 분석:
- **Max path**: t_cq + t_comb + t_su < T_clk → setup 제약
- **Min path**: t_cq + t_comb > t_h → hold 제약

![Setup/Hold Time|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/e6cfb1980889.webp)

## Scaling (공정 미세화)

Moore's law: 트랜지스터 수가 2년마다 2배. 이를 위해 feature size (L)를 **매 세대 0.7배**씩 줄이는 **constant-field scaling** 또는 **constant-voltage scaling** 사용.

- Faster transistors
- Lower power per transistor
- But leakage 증가, wire delay 증가 → 새로운 설계 기법 필요 (FinFET, etc.)

![Scaling 및 공정 미세화|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/9d86057a65a1.webp)

## CMOS의 미래

- **FinFET / GAAFET (Gate-All-Around)**: 미세 공정에서 3D 구조로 leakage 제어
- **Low-power design techniques**: clock gating, power gating, DVFS (Dynamic Voltage Frequency Scaling)
- **Multi-V_T**: 속도 vs power trade-off에 따라 다른 V_T 사용

![CMOS 미래 기술|full](/img/01HMCXW400BBRYRW0ZQKV7NN9E/df67759fe212.webp)

## 정리

| 개념 | 핵심 |
|------|------|
| **CMOS = Complementary MOS** | nMOS + pMOS 상보 구조 |
| **nMOS**: g=1 ON, 0 전달 우수 | **pMOS**: g=0 ON, 1 전달 우수 |
| **Static power ≈ 0** | 정적 상태에서 전류 경로 없음 |
| **Dynamic power**: $\alpha C V^2 f$ | V_DD 감소가 가장 효과적 |
| **Sizing** | W/L로 driving strength 조절, pMOS 2~3배 W |
| **Timing**: Setup/Hold, Clock skew | 매 공정에서 가장 중요한 제약 |

CMOS는 마이크로프로세서 설계의 **물리적 기반**이며, 이후의 모든 digital logic, RTL 설계, computer architecture는 이 위에 쌓인다.
