---
id: "01KFE63TDRQ3MD6Y5AR0TP968T"
slug: "guessing-game-robot-arm-hri"
title: "인간-로봇 상호작용 기반 Guessing Game 시스템 구현"
category: "01M651PC00AT7C6BNVD4N04PH1"
tags: ["HRI","Robot Arm","YOLO","VLM","LLaMA","InternVL","UNIST"]
summary: "YOLO 객체 탐지와 VLM(Vision-Language Model)을 결합한 인간-로봇 상호작용 기반 Guessing Game 시스템을 개발했다. 로봇 팔이 물체를 가리키고, 사용자가 추측하는 게임을 구현했다."
thumbnail: "/img/01KFE63TDRQ3MD6Y5AR0TP968T/a2990c27813c.webp"
note: false
created: "2026-01-21T02:09:23+09:00"
updated: "2026-03-28T02:23:03+09:00"
---
## 1. 프로젝트 개요

### 프로젝트 소개

UNIST Human-Robot Interaction 수업의 팀 프로젝트로 **"Guessing Game with Robot Arm"** 시스템을 개발하였다. 사용자가 제공하는 힌트를 기반으로 로봇이 목표 물체를 추론하고, 해당 물체를 집어서 사용자에게 전달하는 인터랙티브 시스템이다.

이 프로젝트는 멀티모달 인식(Multimodal Perception)과 정밀한 모션 제어(Motion Control)를 통합하여, 로봇이 사용자의 의도를 파악하고 물리적 행동으로 응답하는 Human-Robot Interaction의 핵심 과제를 다룬다.

**프로젝트 링크**: [GitHub Repository](https://github.com/eldor-fozilov/guessing-game-with-robot)

### 팀 구성

이 프로젝트는 UNIST의 6명이 함께 진행하였다:

- Eldor Fozilov (Department of Computer Science and Engineering)
- Gahyeon Shim (Graduate School of Artificial Intelligence)
- Minji Kim (Graduate School of Artificial Intelligence)
- Seongjae Lee (Department of Electrical Engineering)
- Sunhong An (Department of Electrical Engineering)
- Youngbin Ki (Graduate School of Artificial Intelligence)

### 프로젝트 배경

로봇은 일상생활에서 점점 다양한 역할을 수행하고 있다. Boston Dynamics의 Spot은 위험 지역에서 점검 작업을 수행하고, LG의 Smart Home AI Agent는 가정 내 기기들을 관리한다. 이러한 로봇들은 특정 작업 요구사항을 해결하여 인간의 업무 부담을 줄이고 효율성을 높인다.

AI 기반 Human-Robot Interaction 알고리즘의 발전으로, 로봇이 인간의 의도를 추론하고 그에 맞게 반응할 것이라는 기대가 높아지고 있다. 이러한 기술적 진보는 로봇이 기본적인 자동화를 넘어 인간 중심의 상호작용을 가능하게 한다.

### 프로젝트 목표

이 프로젝트에서 개발한 "Guessing Game"은 로봇이 사용자의 힌트, 피드백, 그리고 환경적 맥락을 활용하여 사용자의 의도를 추론하는 시스템이다. 로봇은 세 가지 주요 동작을 수행한다:

1. **추론(Inference)**: 사용자 입력으로부터 목표 물체를 추론
2. **검증(Validation)**: 사용자 피드백을 통해 선택을 확인
3. **전달(Delivery)**: 물체를 사용자에게 전달

이를 위해 시스템은 시각적, 텍스트적 단서를 처리하는 인식 모듈과 로봇 동작을 제어하는 제어 모듈을 통합하였다.

---

## 2. 시스템 아키텍처

### 전체 워크플로우

![시스템 워크플로우 - 사용자 인터페이스부터 로봇 제어까지의 전체 파이프라인|large](/img/01KFE63TDRQ3MD6Y5AR0TP968T/a2990c27813c.webp)

시스템의 전체 워크플로우는 다음과 같다:

1. **사용자 입력**: 사용자가 테이블 위에 물체들을 배치하고 목표 물체에 대한 힌트를 제공
2. **음성 처리**: 음성 입력의 경우 Whisper 모델을 사용하여 텍스트로 변환
3. **인식 모듈**: 테이블의 탑다운 뷰 이미지와 사용자 힌트를 처리하여 물체 탐지 및 추론 수행
4. **로봇 제어**: 추론된 물체의 위치로 로봇 암을 이동시켜 물체를 집음
5. **피드백 루프**: 사용자에게 추론 결과를 보여주고 피드백을 받음
   - 정답인 경우: 물체를 사용자에게 전달
   - 오답인 경우: 물체를 지정된 영역에 놓고, Tacotron2 TTS로 결정 이유를 음성으로 설명

오답으로 판정된 물체는 이후 탐색에서 제외되어 반복적인 오류를 방지한다.

### 사용자 인터페이스

![사용자 인터페이스와 실험 환경|large](/img/01KFE63TDRQ3MD6Y5AR0TP968T/91ed0eebde58.webp)

Flask 기반의 로컬 서버에서 호스팅되는 웹 UI를 구현하였다. 사용자 입력은 JavaScript 파일에서 트리거되는 Python 함수를 통해 처리된다. 이 시스템은 "HRI TEST System"이라고 명명하였다.

UI의 주요 기능:
- **파이프라인 선택**: 두 가지 인식 파이프라인 중 선택
- **카메라 연결**: 테이블 위 물체를 촬영하는 카메라 연결
- **힌트 입력**: 텍스트 또는 음성으로 힌트 제공
- **물체 탐지 결과**: 탐지된 물체와 선택된 물체 표시
- **피드백 제공**: Accept/Reject 버튼으로 결과 평가

---

## 3. 인식 모듈 (Perception Module)

인식 모듈은 물체 탐지와 추론을 담당하며, 미리 정의된 물체와 새로운 물체 모두에 대한 일반화 능력을 탐구하기 위해 두 가지 파이프라인을 구현하였다.

### Pipeline 1: Closed-Vocabulary 접근법

#### 구성 요소
- **YOLO11**: 사전 정의된 클래스 집합을 사용하는 효율적인 물체 탐지 프레임워크
- **LLaMA-3.2-1B**: 강력한 추론 능력을 가진 언어 모델

#### 동작 과정

1. **물체 탐지**: 캡처된 이미지를 YOLO11이 처리하여 사전 정의된 클래스에 속하는 물체들의 바운딩 박스 좌표와 레이블을 제공
2. **필터링**: 사용자가 이전에 거부한 물체들을 필터링
3. **추론**: 필터링된 물체 레이블과 사용자 힌트를 LLaMA에 전달하여 가장 적합한 물체를 선택

#### 프롬프트 설계

LLaMA 모델의 출력을 가이드하기 위해 다음과 같은 프롬프트를 사용하였다:

```
You are an assistant whose task is to identify the correct object from this list
{detected objects} that aligns best with the following clue: '{clue}'.
Take a moment to think about each object's properties, considering how the
clue relates to them, directly or indirectly.
Provide your response in the following format:
Selected Object: <name of the correct object>
Explanation: <brief explanation of how the object's properties align with the clue>
```

이 파이프라인의 출력은 추론된 물체 이름과 설명이며, 이후 시스템 단계로 전달된다.

### Pipeline 2: Open-Vocabulary 접근법

#### 구성 요소
- **InternVL-2.5-2B**: 멀티모달 추론 작업에 뛰어난 비전-언어 모델
- **YOLO-World**: 효율적인 Open-Vocabulary 물체 탐지 프레임워크

#### 동작 과정

1. **멀티모달 추론**: 캡처된 이미지와 사용자 힌트를 InternVL-2.5-2B에 입력
2. **물체 식별 및 선택**: 모델이 이미지의 모든 물체를 탐지/분류하고, 힌트에 가장 적합한 물체를 선택
3. **위치 추출**: 선택된 물체 이름과 이미지를 YOLO-World에 전달하여 바운딩 박스 좌표 획득

#### 프롬프트 설계

```
You are an assistant whose task is to analyze the given image and the following
clue: '{clue}'.
1. Identify and list all the objects present in the image.
2. From the list of identified objects, select the one that best aligns with the
clue, while ignoring these excluded objects if present: {excluded objects}.
3. Provide a brief explanation of why the selected object matches the clue.
Format your response as follows:
All Identified Objects: <comma-separated list of objects>
Selected Object: <name of the correct object>
Explanation: <brief explanation>
```

### 두 파이프라인의 비교

| 특성 | Pipeline 1 | Pipeline 2 |
|------|------------|------------|
| 물체 탐지 | YOLO11 (Closed) | YOLO-World (Open) |
| 추론 모델 | LLaMA-3.2-1B (Text) | InternVL-2.5-2B (Vision+Text) |
| 새로운 물체 처리 | 불가능 | 가능 |
| 처리 속도 | 빠름 | 상대적으로 느림 |

Pipeline 1은 사전 정의된 클래스에 대해 빠르고 정확한 탐지가 가능하지만, 새로운 물체에 대응하지 못한다. Pipeline 2는 Open-Vocabulary 접근법으로 새로운 물체도 처리할 수 있지만, Vision-Language 모델의 추론 시간이 더 소요된다.

---

## 4. 제어 모듈 (Control Module)

### 좌표 변환 (Coordinate Transformation)

Guessing Game에서 로봇은 로봇 베이스 중심점을 원점으로 하는 좌표계에서 동작한다. 그러나 인식 모듈에서 제공하는 물체 위치는 2D 이미지 좌표계로 정의되어 있다. 따라서 2D 좌표를 로봇 좌표계의 3D 좌표로 변환하는 과정이 필요하다.

#### 카메라 캘리브레이션

**내부 행렬(Intrinsic Matrix)** $M_{int}$는 카메라 제조 과정에서 고정되며, ChArUco 보드를 사용하여 한 번 계산된다.

**외부 행렬(Extrinsic Matrix)** $M_{ext}$는 카메라와 월드 좌표계 간의 변환 $T_{cb}$를 나타내며, 카메라 위치가 변경될 때마다 재계산해야 한다. 각 게임 시작 시 카메라가 현재 뷰를 캡처하고, OpenCV 라이브러리를 사용하여 체커보드 패턴을 감지하여 $T_{cb}$를 계산한다.

#### 좌표 변환 수식

캘리브레이션 보드의 로봇 원점에 대한 위치와 방향 $T_{rb}$가 알려져 있으므로, 2D 이미지 포인트 $P_{img}$를 로봇 좌표계의 3D 월드 포인트 $P_{robot}$으로 변환할 수 있다:

$$P_{robot} = T_{rb} \cdot T_{cb}^{-1} \cdot M_{int}^{-1} \cdot P_{img} \cdot Z_{cam}$$

RGB 카메라로는 깊이 정보를 얻을 수 없으므로, 이미지가 항상 탑다운 시점에서 캡처되는 점을 이용하여 카메라에서 작업 공간까지의 고정 높이 $Z_{cam}$을 사용한다.

### 로봇 제어 (Robot Control)

#### 역기구학 (Inverse Kinematics)

로봇 엔드 이펙터가 목표 위치로 이동하도록 하기 위해 역기구학(IK) 솔버를 적용하여 6-DoF 로봇 암의 관절 값을 계산한다. 가볍고 빠른 오픈소스 솔루션인 **IKPy** 라이브러리를 활용하였다.

현재 위치와 목표 위치 사이를 선형 보간하여 궤적을 생성하고, 균등한 간격의 스텝으로 나눈다. 이 궤적을 사용하여 Dynamixel 모터의 위치를 제어한다.

#### PID 제어

목표 위치에 정확하게 도달하기 위해 **PID(Proportional-Integral-Derivative) 제어**를 적용하였다. 오차는 각 관절의 목표 PWM 값과 현재 PWM 값의 차이로 정의된다.

$$u(t) = K_p \cdot e(t) + K_i \cdot \int e(t) dt + K_d \cdot \frac{de(t)}{dt}$$

여기서:
- $e(t)$: 시간 t에서의 위치 오차
- $u(t)$: 모터 위치를 조정하는 제어 신호 출력
- $K_p = 0.5$, $K_i = 0.1$, $K_d = 0.01$

위치 오차가 정의된 임계값 이하로 떨어지면 로봇은 현재 동작을 완료하고 다음 동작으로 이동한다.

#### 충돌 방지

로봇이 다른 물체나 의도하지 않은 영역과 충돌하는 것을 방지하기 위해, 그리핑 과정에서 엔드 이펙터의 방향을 아래 방향으로 고정하였다. 이를 통해 주변 물체와의 충돌을 피하면서 안정적이고 정확한 Pick-and-Place 동작을 보장한다.

---

## 5. 실험 설계 및 환경

### 실험 가정

![실험 셋업 - 작업 공간 정의와 좌표계|large](/img/01KFE63TDRQ3MD6Y5AR0TP968T/e98618285c71.webp)

#### 물체 설계

제공된 그리퍼가 실제 물체를 다루기에 적합하지 않아, **2D 프린트된 이미지**를 사용하였다. 이미지는 인식 모듈이 탐지할 수 있을 만큼 충분히 크게 인쇄하고, 로봇이 잡을 수 있도록 3D 프린트된 기둥을 부착하였다.

#### 물체 배치

모든 물체는 로봇 암의 작동 범위와 카메라의 가시 범위를 고려하여 **사전 정의된 작업 공간(파란색 박스)** 내에 배치한다.

#### 좌표계 정의

실험 환경에는 세 가지 좌표계가 정의되어 있다:
- **{R}**: 로봇 베이스 좌표계 (원점)
- **{B}**: 체커보드 좌표계 (캘리브레이션용)
- **{C}**: 카메라 좌표계

### 실험 환경

- **카메라**: FHD 웹캠
- **로봇**: 위치 제어 방식의 Koch v1.1 (저비용 로봇 암)
- **그리퍼 개조**: 원래 그리퍼는 뾰족한 디자인으로 불안정한 그리핑을 유발하여, 접촉 면적을 늘려 더 안정적인 그리핑이 가능하도록 개조. 추가로 그리퍼 끝에 고무 밴드를 감아 마찰력을 높이고 그립 안정성을 향상

### 실험 시나리오

![실험에 사용된 12개 물체 클래스|large](/img/01KFE63TDRQ3MD6Y5AR0TP968T/73789cf19b75.webp)

총 **20개의 실험 시나리오**를 설계하였다. 이 시나리오들은 12개의 사전 정의된 물체 클래스를 포함하며, YOLO 학습에 사용된 클래스와 사용되지 않은 클래스를 모두 포함한다.

각 시나리오는 **3개의 물체 세트**로 구성된다. 시스템 성능을 평가하기 위해, GPT-4o를 사용하여 각 시나리오에 대해 **두 가지 난이도의 힌트**를 생성하였다.

**물체 클래스 (12종)**:
- YOLO 클래스: 사과, 오렌지, 컵, 가위, 테디베어, 브로콜리, 파인애플, 시계
- Non-YOLO 클래스: 선인장, 고양이, 기린, 얼룩말

---

## 6. 실험 결과

### 인식 모듈 평가

#### 평가 지표

**분류 정확도(Classification Accuracy)**:
$$Acc = \frac{N_{classify}}{N_{total}}$$

- YOLO: 이미지의 모든 물체가 정확히 분류되어야 성공
- YOLO-World: 물체의 클래스 레이블이 Vision-Language 모델이 제공한 클래스와 일치해야 성공

**추론 정확도**:
- $Acc_{c1}$: 어려운 힌트 하나로 성공한 비율
- $Acc_{c2}$: 첫 번째 힌트 실패 후 쉬운 힌트로 성공한 비율

#### 결과

| 파이프라인 | 탐지 정확도 | $Acc_{c1}$ | $Acc_{c2}$ |
|-----------|-----------|-----------|-----------|
| YOLO + LLM (Pipeline 1) | 80.0% | 75.0% | 20.0% |
| VLM + YOLO-World (Pipeline 2) | 100.0% | 75.0% | 100.0% |

**분류 결과 분석**:
- Pipeline 1은 80% 정확도를 달성한 반면, Pipeline 2는 모든 테스트에서 물체를 성공적으로 분류
- YOLO는 사전 정의된 클래스 집합 외의 물체에서 어려움을 겪지만, YOLO-World는 이러한 물체들도 성공적으로 분류
- 이는 YOLO-World가 더 넓은 분류 능력을 가지고 있어 다양한 물체를 처리할 수 있음을 나타냄

**추론 결과 분석**:
- 두 파이프라인 모두 $Acc_{c1}$에서 75%를 달성
- $Acc_{c2}$에서 Pipeline 1은 20%, Pipeline 2는 100%를 달성
- 이 큰 차이는 YOLO-World가 사전 정의된 클래스 집합에 포함되지 않은 새로운 물체도 탐지할 수 있기 때문
- Pipeline 1은 목표 물체가 사전 정의된 클래스 집합에 속하지 않으면 실패

### 로봇 모션 제어 평가

#### 평가 지표

**작업 성공률(Task Success Rate)**: n번의 테스트 실행에서 성공적인 작업 완료의 평균 비율. 로봇이 목표 물체를 집어서 지정된 위치에 놓으면 성공으로 간주.

**위치 오차(Position Error)**: 목표 지점과 로봇이 실제로 이동한 지점 사이의 평균 유클리드 거리.

#### 통합 시스템 테스트

인식과 제어가 통합된 전체 시스템을 테스트하였다. 각 파이프라인에 대해 3개의 물체를 작업 공간에 배치하고, 인식 모듈 평가에 사용된 시나리오를 따랐다. 로봇이 힌트를 기반으로 3개 물체 중 올바른 물체를 추론하고 Pick-and-Place 동작을 수행하면 성공으로 간주한다. 각 파이프라인에 대해 총 n=20번의 실행을 수행하였다.

#### 결과

| 방법 | 성공률 | 위치 오차 |
|------|--------|----------|
| Pipeline 1 + Control | 75.0% | - |
| Pipeline 2 + Control | 75.0% | - |
| Control only | 86.7% | 0.016mm |

**통합 시스템 결과**:
- Pipeline 1과 Pipeline 2 모두 75%의 작업 성공률을 달성, 20번 중 16번 성공
- 각 실행은 1분 이내에 완료되어 시스템의 실시간 운영 능력을 입증
- Pipeline 2와 제어 시스템 통합 시, 대부분의 실패 사례는 실험 진행에 따른 로봇 오작동에서 발생
- Pipeline 2가 새로운 물체에 대해 우수한 추론 성능을 보임에도 불구하고, 모터 오작동으로 인해 로봇이 목표 지점에 도달하지 못하거나 원치 않는 자세를 보이는 경우가 발생

#### 제어 모듈 단독 테스트

![제어 모듈 평가를 위한 추가 실험 셋업|large](/img/01KFE63TDRQ3MD6Y5AR0TP968T/6869d55fd228.webp)

제어 모듈의 유효성을 추가로 평가하기 위해 다른 그룹의 로봇을 사용하여 추가 실험을 수행하였다. 3개의 물체를 작업 공간에 배치하고, 로봇이 목표 물체를 집어서 지정된 위치에 놓으면 성공으로 간주한다. 총 n=60번의 실행을 수행하였다.

**제어 모듈 결과**:
- 모든 실험에서 평균 위치 오차가 2cm 이내로, PID 제어와 IKPy 라이브러리가 제공하는 IK 솔버를 활용한 제어 방법이 정확함을 입증
- 작업 완료 측면에서 로봇은 모든 실행에서 목표 물체를 성공적으로 집음
- 그러나 물체를 지정된 위치에 놓는 데 실패하는 경우가 일부 발생

---

## 7. 결론 및 향후 연구

### 프로젝트 성과

이 프로젝트에서는 직관적인 상호작용을 위해 멀티모달 인식과 제어를 효과적으로 결합한 Human-Robot Interaction 시스템을 개발하였다. 실험 결과는 물체 인식, 추론, 조작에서 견고한 성능을 보여주며, 실제 세계 작업에 대한 적용 가능성을 강조한다.

**주요 성과**:
- 두 가지 인식 파이프라인 구현 및 비교 분석
- Open-Vocabulary 접근법(Pipeline 2)의 새로운 물체에 대한 우수한 일반화 능력 확인
- 좌표 변환과 PID 제어를 통한 정밀한 로봇 모션 제어 구현
- 실시간 운영 가능한 통합 시스템 구축 (각 실행 1분 이내)

### 시스템의 의의

사용자 힌트를 해석하고 사용자 피드백에 응답하는 시스템의 능력은 다음과 같은 응용 분야에서의 잠재력을 보여준다:
- **개인화된 지원(Personalized Assistance)**: 사용자 의도를 파악하여 맞춤형 서비스 제공
- **스마트 홈(Smart Homes)**: 음성/텍스트 명령으로 가정 내 물건 조작
- **협업 작업 공간(Collaborative Workspaces)**: 작업자와 로봇 간의 직관적인 협업

### 향후 연구 방향

향후 연구에서는 다음과 같은 방향으로 시스템의 능력을 확장할 수 있다:

1. **복잡한 환경으로 확장**: 더 다양하고 복잡한 실제 환경에서의 테스트
2. **이동 로봇 통합**: 고정된 로봇 암에서 모바일 로봇으로 확장
3. **실제 물체 처리**: 2D 프린트 이미지가 아닌 실제 3D 물체 조작
4. **다중 턴 대화**: 더 복잡한 힌트와 대화를 통한 추론 능력 향상
5. **학습 기반 제어**: PID 제어에서 강화학습 기반 제어로 발전

### 참고 문헌

- [Whisper](https://github.com/openai/whisper) - OpenAI's Automatic Speech Recognition System
- [Coqui TTS](https://github.com/coqui-ai/TTS) - Deep learning toolkit for Text-to-Speech
- [YOLO11](https://arxiv.org/abs/2410.17725) - Object Detection Framework
- [LLaMA 3.2-1B](https://github.com/meta-llama/llama3) - Multilingual Large Language Model
- [InternVL2.5-2B](https://github.com/OpenGVLab/InternVL) - Multimodal Large Language Model
- [YOLO-World](https://arxiv.org/abs/2401.17270) - Real-Time Open-Vocabulary Object Detection
- [IKPy](https://github.com/Phylliade/ikpy) - Inverse Kinematics Library
