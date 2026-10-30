---
id: "01KFE63RF8YES2EV8AT799BVFN"
slug: "rectilinear-steiner-tree-routing"
title: "VLSI 배선 최적화를 위한 Rectilinear Steiner Tree 알고리즘 설계"
category: "01M651PC00AT7C6BNVD4N04PH1"
tags: ["VLSI","RSMT","Steiner Tree","Dijkstra","CAD","Routing"]
summary: "VLSI 설계의 배선 최적화를 위한 Rectilinear Steiner Minimum Tree(RSMT) 알고리즘을 Dijkstra 기반으로 구현했다. Hanan Grid를 활용한 Steiner 후보점 선정과 반복적 최적화를 통해 배선 길이를 최소화했다."
thumbnail: "/img/01KFE63RF8YES2EV8AT799BVFN/299d2cd5906e.webp"
note: false
created: "2026-01-21T02:09:21+09:00"
updated: "2026-03-28T02:23:03+09:00"
---
## 서론

이 프로젝트에서는 디지털 시스템 CAD 알고리즘 수업(EE58401)의 일환으로 **Rectilinear Steiner Minimum Tree(RSMT)** 문제를 해결하는 알고리즘을 직접 설계하고 구현했다. RSMT는 VLSI(Very Large Scale Integration) 설계에서 핵심적인 역할을 하는 문제로, 주어진 점들을 최소 비용으로 연결하는 배선 경로를 찾는 것이 목표다.

VLSI 칩 설계에서 수백만 개의 게이트와 트랜지스터를 효율적으로 연결하는 것은 칩의 성능, 전력 소비, 면적에 직접적인 영향을 미친다. 배선이 길어질수록 신호 지연이 증가하고, 전력 소비가 늘어나며, 칩의 물리적 면적도 커진다. 따라서 최소 길이의 배선을 찾는 RSMT 문제는 실제 산업 현장에서 매우 중요한 최적화 문제다.

반도체 기술이 미세화됨에 따라 배선 최적화의 중요성은 더욱 커지고 있다. 현대의 반도체 공정에서는 배선 지연이 게이트 지연을 초과하는 경우가 많아, 배선 최적화가 전체 칩 성능을 결정하는 핵심 요소가 되었다. 특히 3nm, 2nm 공정으로 갈수록 배선의 저항과 커패시턴스가 증가하여 최적 배선의 중요성이 더욱 부각되고 있다.

이 보고서에서는 다익스트라(Dijkstra) 알고리즘을 기반으로 한 RSMT 탐색 알고리즘을 구현하고, 다양한 문제 상황(교차 엣지, 대각선 엣지, 루프)을 처리하는 방법을 설명한다. 또한 지수 분포를 활용한 확률적 스타이너 포인트 선택 전략과 반복 최적화를 통해 해의 품질을 개선하는 과정을 상세히 다룬다.

### 프로젝트 개요

이 프로젝트의 목표는 주어진 점 집합에 대해 RSMT를 찾는 효율적인 알고리즘을 구현하는 것이다. 입력은 텍스트 파일로 제공되며, 각 줄에 공백으로 구분된 x, y 좌표가 포함된다. 출력은 RSMT를 구성하는 엣지들의 좌표 정보가 담긴 파일이다.

알고리즘 설계 시 고려한 요구사항:
- **정확성**: 모든 터미널 점이 연결된 유효한 트리를 생성해야 한다.
- **효율성**: 시간 제한(60분) 내에 결과를 출력해야 한다.
- **품질**: 가능한 짧은 총 배선 길이를 달성해야 한다.
- **확장성**: 다양한 크기의 입력(3~100개 점)을 처리할 수 있어야 한다.

구현 언어로는 Python을 선택했다. Python의 장점은 빠른 프로토타이핑, 풍부한 라이브러리(heapq, random, tqdm 등), 그리고 가독성 있는 코드 작성이 가능하다는 것이다. 단점인 실행 속도는 알고리즘 최적화와 반복 탐색으로 보완했다.

## RSMT 문제의 이론적 배경

### Steiner Tree 문제란?

**Steiner Tree** 문제는 그래프 이론에서 가장 중요한 조합 최적화 문제 중 하나로, 1836년 스위스의 수학자 Jakob Steiner의 이름을 따서 명명되었다. 이 문제의 목표는 주어진 정점 집합(터미널)을 연결하는 최소 비용의 트리를 찾는 것이다.

일반적인 최소 신장 트리(MST, Minimum Spanning Tree)와의 핵심적인 차이점은 **추가적인 정점(Steiner Point)**을 사용할 수 있다는 것이다. MST는 주어진 정점만을 사용하여 트리를 구성하지만, Steiner Tree는 필요에 따라 새로운 정점을 추가하여 전체 비용을 줄일 수 있다.

예를 들어, 세 점 A, B, C가 정삼각형을 이룬다고 가정하자. MST로 이 세 점을 연결하면 두 개의 엣지가 필요하고, 총 길이는 삼각형 두 변의 합이 된다. 정삼각형의 한 변 길이가 1이라면, MST의 총 길이는 2가 된다. 하지만 삼각형의 **페르마 점(Fermat Point)**에 Steiner Point를 추가하면, 세 점 모두 이 중심점에 연결되어 총 배선 길이를 약 1.732(√3)로 줄일 수 있다. 이는 약 13.4%의 비용 절감을 의미한다.

Steiner Tree 문제는 1972년 Karp에 의해 **NP-hard**로 증명되었다. 이는 입력 크기가 커지면 최적해를 찾는 것이 계산적으로 매우 어려워진다는 것을 의미한다. 다항 시간 내에 최적해를 보장하는 알고리즘은 알려져 있지 않으며, 따라서 실제 응용에서는 휴리스틱이나 근사 알고리즘을 사용한다.

### Steiner Point의 수학적 특성

Steiner Point는 몇 가지 중요한 수학적 특성을 가진다:

1. **차수 제한**: 유클리드 Steiner Tree에서 각 Steiner Point의 차수는 정확히 3이다. 이는 세 개의 엣지가 120도 간격으로 만나는 지점이 에너지적으로 가장 안정적이기 때문이다. RSMT에서는 직교 제약으로 인해 차수가 2, 3, 또는 4가 될 수 있다.

2. **Steiner Point 개수 상한**: n개의 터미널 점이 있을 때, 최적 Steiner Tree에 필요한 Steiner Point의 개수는 최대 n-2개다. 이는 귀납법으로 쉽게 증명할 수 있다.

3. **국소 최적성**: Steiner Point는 연결된 모든 터미널에서 오는 "힘"의 균형점에 위치한다. 이는 물리적으로 비누막 실험으로 증명할 수 있다.

4. **Steiner Ratio**: Steiner Tree의 길이와 MST 길이의 비율을 Steiner Ratio라 한다. 유클리드 평면에서 Steiner Ratio의 하한은 √3/2 ≈ 0.866으로, 최대 13.4%의 비용 절감이 가능하다. RSMT에서는 Steiner Ratio의 하한이 2/3 ≈ 0.667으로, 최대 33.3%까지 절감할 수 있다.

Steiner Tree를 시각적으로 이해하는 좋은 방법은 비누막 실험이다. 두 장의 유리판 사이에 핀을 꽂고 비눗물에 담그면, 비누막이 자연스럽게 핀들을 연결하는 최소 에너지 구조를 형성한다. 이 구조가 바로 Steiner Tree에 근사한다. RSMT의 경우, 격자 형태의 틀 안에서 실험하면 직교 특성을 관찰할 수 있다.

### Rectilinear Steiner Minimum Tree (RSMT)

**RSMT**는 Steiner Tree 문제의 특수한 형태로, 두 가지 핵심 제약이 있다:

1. **맨해튼 거리 사용**: 유클리드 거리 대신 맨해튼 거리(L1 norm)를 사용한다.
2. **직교 배선**: 모든 엣지가 수평 또는 수직으로만 연결되어야 한다.

맨해튼 거리는 다음과 같이 정의된다:

$$d(p_1, p_2) = |x_1 - x_2| + |y_1 - y_2|$$

이 제약은 VLSI 설계의 실제 상황을 반영한다. 실리콘 칩에서 배선은 일반적으로 수평 또는 수직 방향으로만 배치되기 때문이다. 이는 제조 공정의 한계와 배선 밀도 최적화를 위한 설계 규칙에서 비롯된다. 최근에는 45도 각도 배선(X-architecture)이나 다양한 각도의 배선을 지원하는 기술도 발전하고 있지만, 여전히 직교 배선이 가장 일반적이다.

### Hanan Grid 정리

RSMT 문제에서 가장 중요한 이론적 결과 중 하나는 **Hanan Grid 정리**다. 1966년 Maurice Hanan이 증명한 이 정리에 따르면, RSMT의 최적해에 포함되는 모든 Steiner Point는 터미널 점들의 x 좌표와 y 좌표가 교차하는 그리드 위에만 위치한다.

구체적으로, n개의 터미널 점 $\{(x_1, y_1), (x_2, y_2), ..., (x_n, y_n)\}$이 주어졌을 때, Hanan Grid는 다음과 같이 정의된다:

$$H = \{(x_i, y_j) : 1 \leq i, j \leq n\}$$

이 그리드의 크기는 최대 $n^2$개의 점이지만, 이 중에서 최적 RSMT를 구성하는 데 필요한 Steiner Point는 일부에 불과하다.

Hanan Grid 정리의 의미는 탐색 공간을 무한한 연속 공간에서 유한한 이산 공간으로 줄여준다는 것이다. 하지만 여전히 $n^2$개의 후보 점 중에서 올바른 조합을 찾는 것은 NP-hard 문제로 남아있다.

### RSMT의 근사 비율

RSMT 문제의 근사 알고리즘에 대한 연구도 활발하다. 가장 간단한 근사는 MST를 사용하는 것인데, 이 경우 근사 비율은 2가 된다. 즉, MST 기반 해는 최적해의 최대 2배 이내임이 보장된다.

더 정교한 알고리즘들은 더 나은 근사 비율을 달성한다:

- **1-Steiner 알고리즘**: 근사 비율 약 1.5
- **Iterated 1-Steiner 알고리즘**: 근사 비율 약 1.3
- **PTAS (Polynomial Time Approximation Scheme)**: 이론적으로 $(1+\epsilon)$ 근사 가능

### RSMT의 응용 분야

RSMT는 다양한 분야에서 핵심적인 역할을 한다:

1. **VLSI 물리적 설계**: 칩 내부의 게이트, 플립플롭, 메모리 셀 등을 연결하는 배선 경로 최적화. 현대 프로세서에는 수십억 개의 트랜지스터가 있으며, 이들을 연결하는 배선의 효율성이 칩 성능을 좌우한다.

2. **PCB 설계**: 인쇄 회로 기판에서 컴포넌트 간 연결 최적화. 다층 PCB에서 각 레이어의 배선을 최소화하여 제조 비용과 신호 무결성을 개선한다.

3. **통신 네트워크**: 노드 간 케이블 배치 최적화. 데이터 센터나 통신망에서 물리적 케이블 길이를 최소화하여 비용과 지연을 줄인다.

4. **건축 및 도시 계획**: 배관이나 전선 경로 설계. 건물 내 배관 시스템이나 도시의 전력망 설계에 활용된다.

5. **멀티칩 모듈(MCM)**: 여러 다이를 연결하는 인터포저 설계에서 배선 최적화.

6. **FPGA 라우팅**: Field Programmable Gate Array에서 로직 블록 간 연결 최적화. FPGA의 라우팅 리소스는 제한적이므로 효율적인 배선이 중요하다.

7. **클럭 트리 합성(CTS)**: 플립플롭에 클럭 신호를 분배하는 트리 설계. 스큐(skew) 최소화와 함께 총 배선 길이 최소화가 목표다.

### VLSI 설계 흐름에서 RSMT의 위치

RSMT는 VLSI 설계 흐름의 물리적 설계(Physical Design) 단계에서 사용된다. 전체 흐름을 살펴보면:

1. **RTL 설계**: Verilog/VHDL로 기능 설계
2. **합성(Synthesis)**: RTL을 게이트 레벨로 변환
3. **플로어플래닝(Floorplanning)**: 블록 배치
4. **배치(Placement)**: 셀 배치
5. **글로벌 라우팅(Global Routing)**: RSMT로 대략적 경로 결정
6. **상세 라우팅(Detailed Routing)**: 실제 배선 경로 확정
7. **타이밍 분석 및 최적화**: 성능 검증

RSMT는 주로 글로벌 라우팅 단계에서 사용된다. 이 단계에서 생성된 RSMT는 상세 라우팅의 가이드 역할을 하며, 최종 배선의 품질에 큰 영향을 미친다.

## 알고리즘 설계

### 전체 알고리즘 흐름

내가 설계한 RSMT 알고리즘은 다음과 같은 단계로 구성된다:

1. **데이터 파싱**: 입력 파일에서 점 좌표 읽기
2. **Steiner Point 후보 계산**: Hanan Grid 기반 후보점 생성
3. **Steiner Point 선택**: 지수 분포를 활용한 확률적 선택
4. **RSMT 탐색**: 다익스트라 알고리즘 기반 트리 구성
5. **문제 상황 처리**: 교차 엣지, 대각선 엣지, 루프 제거
6. **최적화**: 불필요한 Steiner Point 제거
7. **반복 최적화**: 시간 제한 내 반복 탐색

각 단계에서 발생할 수 있는 문제 상황과 그 해결 방법을 구체적으로 구현했다. 특히 실제 배선 문제에서 자주 발생하는 교차(crossing), 대각선 연결, 루프 등을 체계적으로 처리하는 로직을 설계했다.

![입력 데이터의 Original Points 분포|large](/img/01KFE63RF8YES2EV8AT799BVFN/299d2cd5906e.webp)

위 그림은 입력으로 주어진 원본 점들(Original Points)의 분포를 보여준다. 100개의 점이 10000×10000 크기의 2차원 공간에 분포되어 있으며, 이 점들을 최소 길이의 직교 트리로 연결하는 것이 목표다. 점들이 비교적 균일하게 분포되어 있지만, 일부 영역에 밀집된 부분도 있어 효율적인 Steiner Point 배치가 중요함을 알 수 있다.

### Point 클래스 설계

알고리즘의 기본 데이터 구조로 Point 클래스를 정의했다:

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __str__(self):
        return f"{self.x:4d} {self.y:4d}"
```

간단한 구조지만, 2차원 좌표를 효율적으로 관리하고 출력 형식을 통일하는 데 유용하다. 좌표값은 정수형으로 처리하여 부동소수점 오차 문제를 방지했다. 출력 형식에서 4자리 고정폭을 사용하여 로그 파일의 가독성을 높였다.

### 입력 데이터 파싱

입력 파일에서 점 좌표를 읽어오는 함수를 구현했다:

```python
def parse_points_from_file(filename):
    points = []
    try:
        with open(filename, 'r') as file:
            for line in file:
                x, y = map(int, line.split())
                point = Point(x, y)
                points.append(point)
    except FileNotFoundError:
        print(f"Error: The file '{filename}' was not found.")
    except Exception as e:
        print(f"An error occurred: {e}")
    return points
```

예외 처리를 통해 파일이 없거나 형식이 잘못된 경우에도 프로그램이 안전하게 종료되도록 했다. 입력 형식은 각 줄에 공백으로 구분된 x, y 좌표가 있는 간단한 텍스트 파일을 가정한다.

### Steiner Point 후보 생성

Hanan Grid에 기반하여 모든 가능한 Steiner Point 후보를 생성한다:

```python
def find_all_points(points):
    all_points = []

    x_coords = sorted(set(p.x for p in points))
    y_coords = sorted(set(p.y for p in points))

    for x in x_coords:
        for y in y_coords:
            all_points.append(Point(x, y))

    return all_points
```

이 함수는 다음과 같은 과정을 거친다:

1. 모든 원본 점들의 x 좌표를 추출하고 중복을 제거한 후 정렬
2. 모든 원본 점들의 y 좌표를 추출하고 중복을 제거한 후 정렬
3. 모든 (x, y) 조합을 생성하여 Hanan Grid 구성

n개의 원본 점이 있을 때, 최대 n²개의 그리드 점이 생성된다. 예를 들어 100개의 점이 있으면 최대 10,000개의 그리드 점이 생성될 수 있다. 이 중 원본 점을 제외한 나머지가 Steiner Point 후보가 된다.

![모든 Steiner Point 후보 (주황색)|large](/img/01KFE63RF8YES2EV8AT799BVFN/179d29505fe1.webp)

위 그림은 Hanan Grid 기반으로 생성된 모든 Steiner Point 후보(주황색)를 보여준다. 파란색 원본 점들의 x, y 좌표가 교차하는 모든 점이 후보로 생성되어, 그리드 형태를 이룬다. 이 그리드의 밀도를 보면 탐색 공간이 얼마나 큰지 짐작할 수 있다.

### 맨해튼 거리 계산

두 점 사이의 맨해튼 거리를 계산하는 함수:

```python
def manhattan_distance(p1, p2):
    return abs(p1.x - p2.x) + abs(p1.y - p2.y)
```

RSMT에서는 유클리드 거리가 아닌 맨해튼 거리를 사용한다. 맨해튼 거리는 "택시 거리"라고도 불리며, 격자 형태의 도시에서 택시가 이동하는 거리와 같다. VLSI에서 직교 배선만 허용되므로 이 거리 척도가 적절하다.

맨해튼 거리의 특성:
- 항상 유클리드 거리보다 크거나 같다 (등호는 수평/수직 이동 시에만)
- 계산이 간단하여 빠르다 (제곱근 연산 불필요)
- 정수 좌표에서 정확한 정수 결과를 반환한다

맨해튼 거리와 유클리드 거리의 관계를 수학적으로 분석하면, 두 점 사이의 맨해튼 거리는 유클리드 거리의 1배에서 √2배 사이다. 구체적으로, 두 점이 수평 또는 수직으로 정렬되어 있을 때 두 거리는 같고, 45도 각도로 배치되어 있을 때 맨해튼 거리는 유클리드 거리의 √2배가 된다. 이 차이는 RSMT와 유클리드 Steiner Tree의 최적해 비율에도 영향을 미친다.

VLSI 설계에서 맨해튼 거리를 사용하는 이유는 물리적 제약과 관련이 있다. 칩 제조 공정에서 배선은 특정 금속 레이어에 형성되며, 각 레이어는 선호 방향(preferred direction)을 가진다. 예를 들어 M1 레이어는 수평, M2 레이어는 수직 방향으로 배선하는 것이 일반적이다. 이러한 제약으로 인해 대각선 배선은 물리적으로 불가능하거나 비효율적이므로, 맨해튼 거리가 실제 배선 길이를 정확하게 반영한다.

### 총 배선 길이 계산

최종 RSMT의 품질을 평가하기 위해 총 배선 길이를 계산하는 함수를 구현했다:

```python
def calculate_total_length(all_points, rmst_edges):
    total_length = 0
    for u, v in rmst_edges:
        if all_points[u] is not None and all_points[v] is not None:
            total_length += manhattan_distance(all_points[u], all_points[v])
    return total_length
```

이 함수는 트리의 모든 엣지에 대해 맨해튼 거리를 합산한다. None으로 표시된 제거된 점은 건너뛴다. 총 길이는 알고리즘의 최적화 목표이자 성능 평가 지표로 사용된다.

### 지수 분포 기반 Steiner Point 선택

모든 후보 Steiner Point를 사용하면 계산 복잡도가 너무 높아진다. 100개의 터미널에서 생성되는 최대 10,000개의 후보 점을 모두 고려하면 시간 내에 해를 찾기 어렵다. 따라서 확률적으로 일부만 선택하는 전략을 사용했다:

```python
LAMBDA = 20

random_val = min(max(int(random.expovariate(LAMBDA) * (len(steiner_points) - 1)), 0), len(steiner_points))
selected_steiner_points = random.sample(steiner_points, random_val)
```

**지수 분포(Exponential Distribution)**를 사용한 이유는 다음과 같다:

1. **적은 수의 Steiner Point 선호**: 지수 분포는 작은 값이 나올 확률이 높아, 적은 수의 Steiner Point를 선택하는 경향이 있다. 이는 계산 효율성을 높인다.

2. **다양성 확보**: 가끔은 많은 Steiner Point를 선택하여 더 좋은 해를 찾을 가능성도 열어둔다. 지수 분포의 긴 꼬리 특성이 이를 가능하게 한다.

3. **LAMBDA 파라미터 조절**: LAMBDA 값이 클수록 적은 수의 Steiner Point를 선택하는 경향이 강해진다. LAMBDA = 20은 실험을 통해 결정된 값으로, 계산 시간과 해 품질의 균형을 맞춘다.

4. **무기억 성질**: 지수 분포의 무기억 성질로 인해 각 반복에서 독립적인 선택이 이루어진다.

지수 분포의 확률 밀도 함수는 다음과 같다:

$$f(x; \lambda) = \lambda e^{-\lambda x}, \quad x \geq 0$$

LAMBDA = 20일 때, 기대값은 1/20 = 0.05이므로, 평균적으로 전체 후보의 5% 정도가 선택된다.

![선택된 Steiner Points (필터링 후)|large](/img/01KFE63RF8YES2EV8AT799BVFN/449fcc1c15fc.webp)

위 그림은 지수 분포를 통해 확률적으로 선택된 Steiner Point들을 보여준다. 전체 후보 중 일부만 선택되어 계산 효율성이 크게 향상된다. 선택된 점들이 비교적 골고루 분포되어 있어, 전체 트리 구조에 기여할 수 있음을 확인할 수 있다.

## 다익스트라 기반 RSMT 탐색

### 알고리즘 개요

다익스트라(Dijkstra) 알고리즘은 원래 그래프에서 단일 출발점에서 모든 정점까지의 최단 경로를 찾는 알고리즘이다. 1956년 네덜란드의 컴퓨터 과학자 Edsger Dijkstra가 고안했으며, 그래프 알고리즘의 고전으로 여겨진다.

내가 구현한 RSMT 탐색은 다익스트라 알고리즘을 약간 변형하여 최소 신장 트리를 구성하는 데 사용한다. 이 방식은 프림(Prim) 알고리즘과 유사하게, 우선순위 큐를 사용하여 가장 가까운 점을 순차적으로 트리에 추가한다.

프림 알고리즘과의 차이점은 다음과 같다:
- 프림: 현재 트리에서 가장 가까운 정점을 추가
- 다익스트라 변형: 출발점에서 누적 거리가 가장 작은 정점을 추가

실제로 맨해튼 거리를 사용하는 RSMT 문제에서는 두 방식이 유사한 결과를 생성한다.

```python
def find_rsmt_dijkstra(original_points, selected_steiner_points, all_points):
    num_points = len(all_points)
    dist = [float('inf')] * num_points
    parent = [-1] * num_points
    visited = [False] * num_points
    edges = []

    dist[0] = 0
    queue = [(0, 0)]  # (distance, node)

    while queue:
        _, u = heapq.heappop(queue)
        if visited[u]:
            continue
        visited[u] = True
        if parent[u] != -1:
            edges.append((parent[u], u))
        for v in range(num_points):
            if visited[v] or u == v:
                continue
            w = manhattan_distance(all_points[u], all_points[v])
            if w < dist[v]:
                dist[v] = w
                parent[v] = u
                heapq.heappush(queue, (w, v))
```

### 알고리즘 상세 분석

1. **초기화 단계**:
   - `dist` 배열: 각 정점까지의 현재 최단 거리. 초기값은 무한대
   - `parent` 배열: 각 정점의 부모 노드 (트리 구조 저장용)
   - `visited` 배열: 방문 여부 플래그
   - 시작점(0번 노드)의 거리를 0으로 설정

2. **반복 단계**:
   - 우선순위 큐에서 가장 작은 거리의 노드를 추출
   - 이미 방문한 노드면 건너뛰기
   - 방문 처리 후, 부모가 있으면 엣지 추가
   - 모든 이웃 노드에 대해 거리 갱신 및 큐에 추가

3. **종료 조건**:
   - 큐가 비면 종료
   - 모든 연결된 노드가 트리에 포함됨

### 우선순위 큐 활용

파이썬의 `heapq` 모듈을 사용하여 최소 힙 기반 우선순위 큐를 구현했다. 최소 힙의 특성:

- `heappush`: O(log n) 시간에 원소 삽입
- `heappop`: O(log n) 시간에 최소 원소 추출 및 제거
- 힙 불변성: 부모 노드는 항상 자식 노드보다 작거나 같음

이를 통해 매번 가장 작은 거리의 점을 효율적으로 추출할 수 있다. 배열 기반의 선형 탐색을 사용하면 O(n)이 걸리지만, 힙을 사용하면 O(log n)으로 줄어든다.

힙의 내부 동작을 이해하면 알고리즘 동작을 더 잘 파악할 수 있다. 힙은 완전 이진 트리로 표현되며, 배열로 효율적으로 구현할 수 있다. 인덱스 i의 노드에 대해 부모는 (i-1)//2, 왼쪽 자식은 2i+1, 오른쪽 자식은 2i+2에 위치한다. heappush 연산 시 새 원소는 트리의 마지막에 추가된 후 부모와 비교하며 위로 이동(sift-up)한다. heappop 연산 시 루트를 제거하고 마지막 원소를 루트로 옮긴 후 자식과 비교하며 아래로 이동(sift-down)한다.

Python의 heapq 모듈은 C로 구현되어 있어 순수 Python 구현보다 상당히 빠르다. 하지만 일부 고급 기능(decrease-key 연산 등)은 지원하지 않는다. 필요한 경우 커스텀 힙 구현이나 외부 라이브러리(sortedcontainers 등)를 고려할 수 있다.

### 시간 복잡도 분석

알고리즘의 시간 복잡도를 단계별로 분석해보자:

**1단계: Steiner Point 후보 생성 - O(n²)**
```
x_coords = sorted(set(p.x for p in points))  # O(n log n)
y_coords = sorted(set(p.y for p in points))  # O(n log n)
for x in x_coords:  # O(n)
    for y in y_coords:  # O(n)
        all_points.append(Point(x, y))  # O(1)
```
최악의 경우 n²개의 그리드 점이 생성된다.

**2단계: Steiner Point 선택 - O(k)**
지수 분포로 k개의 Steiner Point를 선택한다. random.sample은 O(k) 시간이 걸린다.

**3단계: 다익스트라 탐색 - O(m² log m)**
m = n + k개의 점에 대해 다익스트라를 수행한다. 각 점에서 모든 다른 점으로의 거리를 계산(O(m))하고, 힙에 삽입(O(log m))하므로 총 O(m² log m)이다.

**4단계: 문제 상황 처리 - O(e²)**
e개의 엣지에 대해 교차 검사를 수행한다. 최악의 경우 모든 쌍을 검사해야 하므로 O(e²)이다. 트리에서 e = m - 1이므로 O(m²)이다.

**5단계: 최적화 - O(m²)**
각 반복에서 모든 엣지와 점을 검사한다.

**전체 시간 복잡도**: 한 번의 탐색은 O(m² log m)이고, 이를 최대 T번 반복하므로 O(T × m² log m)이다. 여기서 T는 반복 횟수, m은 점의 총 개수다.

**공간 복잡도**: O(m²)의 거리 정보를 저장하지 않고 필요할 때 계산하므로, 공간 복잡도는 O(m)으로 점과 엣지 정보만 저장한다.

더 효율적인 구현을 위해 다음과 같은 최적화가 가능하다:
- 인접 리스트 대신 암시적 그래프 사용 (현재 구현)
- k-d 트리를 사용한 근접점 탐색 - 평균 O(log n) 근접점 쿼리
- 병렬 처리를 통한 거리 계산 가속 - 멀티코어 활용
- 사전 계산 테이블(FLUTE 방식) - 소규모 문제에서 O(1) 조회

## 문제 상황 처리

다익스트라 알고리즘으로 초기 트리를 구성한 후, 세 가지 주요 문제 상황을 처리해야 한다. 이 문제들은 RSMT의 제약조건(직교 배선, 무사이클)을 만족시키기 위해 반드시 해결해야 한다.

### 1. 교차 엣지(Crossing Edges) 처리

RSMT에서 두 엣지가 교차하면 안 된다. 왜냐하면:
1. **배선 무결성**: 실제 VLSI에서 같은 레이어의 두 배선이 교차하면 단락(short circuit)이 발생
2. **트리 구조 위반**: 교차점을 노드로 만들지 않으면 트리 구조가 아님

두 엣지의 교차를 감지하는 함수:

```python
def is_crossing(p1, p2, p3, p4):
    def direction(a, b, c):
        return (b.x - a.x) * (c.y - a.y) - (b.y - a.y) * (c.x - a.x)

    d1 = direction(p1, p2, p3)
    d2 = direction(p1, p2, p4)
    d3 = direction(p3, p4, p1)
    d4 = direction(p3, p4, p2)

    if p1 == p3 or p1 == p4 or p2 == p3 or p2 == p4:
        return False  # 공유점이 있는 경우 교차로 간주하지 않음

    return (d1 * d2 < 0) and (d3 * d4 < 0)
```

이 함수는 **CCW(Counter-Clockwise)** 알고리즘을 사용하여 두 선분의 교차 여부를 판단한다. CCW 알고리즘의 원리:

1. `direction(a, b, c)` 함수는 세 점의 방향성을 계산
   - 양수: 반시계 방향
   - 음수: 시계 방향
   - 0: 일직선

2. 두 선분 (p1-p2)와 (p3-p4)의 교차 조건:
   - p1-p2 선분을 기준으로 p3와 p4가 반대편에 있어야 함
   - p3-p4 선분을 기준으로 p1과 p2가 반대편에 있어야 함

3. 방향값의 부호 조합으로 교차를 판정

교차가 발견되면 교차점에 새로운 Steiner Point를 추가하고 엣지를 재구성한다:

```python
if is_crossing(p1, p2, p3, p4):
    # 교차점 계산
    if p1.x == p2.x:  # 첫 번째 엣지가 수직
        mid_point_x = p1.x
        mid_point_y = p3.y
    else:  # 두 번째 엣지가 수직
        mid_point_x = p3.x
        mid_point_y = p1.y

    mid_point = Point(mid_point_x, mid_point_y)
    # 새 Steiner Point 추가 및 엣지 재구성
```

직교 배선에서 두 선분이 교차하면 한 선분은 수평, 다른 선분은 수직이다. 따라서 교차점은 수직 선분의 x좌표와 수평 선분의 y좌표로 쉽게 계산할 수 있다.

교차 처리 후, 원래 두 개의 엣지는 네 개의 엣지로 대체된다. 예를 들어 A-B와 C-D가 교차점 X에서 만났다면, A-X, X-B, C-X, X-D로 재구성된다. 이렇게 하면 총 배선 길이는 변하지 않으면서 교차가 해소된다.

교차 검사의 시간 복잡도는 O(e²)로, 모든 엣지 쌍을 검사해야 한다. e개의 엣지에서 최대 e(e-1)/2 쌍을 검사한다. 더 효율적인 구현을 위해서는 스위핑(sweeping) 알고리즘이나 공간 분할 기법을 사용할 수 있다.

### 2. 대각선 엣지(Diagonal Edges) 처리

RSMT에서는 모든 엣지가 수평 또는 수직이어야 한다. 다익스트라 알고리즘의 결과로 대각선 엣지가 생성될 수 있으므로 이를 처리해야 한다.

```python
def is_diagonal(p1, p2):
    return p1.x != p2.x and p1.y != p2.y

# 대각선 엣지 처리
if is_diagonal(all_points[u], all_points[v]):
    edges.pop(l)
    mid_point = Point(all_points[u].x, all_points[v].y)
    all_points.append(mid_point)
    selected_steiner_points.append(mid_point)

    mid_point_index = len(all_points) - 1
    edges.append((u, mid_point_index))
    edges.append((mid_point_index, v))
```

대각선 엣지 (u, v)를 감지하면:
1. 원래 엣지 제거
2. u의 x좌표와 v의 y좌표를 가진 중간점 생성 (L자 꺾임점)
3. 중간점을 Steiner Point 목록에 추가
4. 두 개의 직교 엣지로 대체: (u, mid) + (mid, v)

이렇게 하면 원래 대각선 경로가 L자 형태의 직교 경로로 변환된다. 맨해튼 거리 특성상 총 길이는 동일하게 유지된다.

L자 경로의 방향 선택에는 두 가지 옵션이 있다: (1) 먼저 수평으로 이동 후 수직, (2) 먼저 수직으로 이동 후 수평. 현재 구현에서는 첫 번째 방식을 사용하지만, 혼잡도나 장애물을 고려할 때는 다른 방향이 더 좋을 수 있다. 이는 향후 개선 사항으로 고려할 수 있다.

### 3. 루프(Loop) 감지 및 제거

트리 구조에서는 사이클(cycle)이 존재하면 안 된다. 하지만 엣지 재구성 과정에서 의도치 않게 사이클이 형성될 수 있다.

```python
def find_loops(edges):
    loops = []
    edge_dict = {}

    # 엣지를 딕셔너리로 변환
    for u, v in edges:
        if u not in edge_dict:
            edge_dict[u] = []
        if v not in edge_dict:
            edge_dict[v] = []
        edge_dict[u].append(v)
        edge_dict[v].append(u)

    # 루프 검출
    for u in edge_dict:
        if len(edge_dict[u]) >= 2:
            for i in range(len(edge_dict[u])):
                for j in range(i + 1, len(edge_dict[u])):
                    v1, v2 = edge_dict[u][i], edge_dict[u][j]
                    if v1 in edge_dict and v2 in edge_dict:
                        common_neighbors = set(edge_dict[v1]) & set(edge_dict[v2])
                        for cn in common_neighbors:
                            if cn != u and len({u, v1, v2, cn}) == 4:
                                loops.append([(u, v1), (u, v2), (v1, cn), (v2, cn)])

    return loops
```

이 함수는 4개의 점으로 이루어진 사각형 형태의 루프를 감지한다:
1. 엣지를 인접 리스트 형태로 변환
2. 각 노드에서 두 이웃을 선택
3. 두 이웃의 공통 이웃이 있으면 루프 형성
4. 감지된 루프를 목록에 추가

루프가 발견되면 무작위로 하나의 엣지를 제거하여 트리 구조를 유지한다. 무작위 선택은 특정 패턴에 치우치지 않도록 하기 위함이다.

사각형 루프가 형성되는 전형적인 상황은 두 개의 L자 경로가 같은 시작점과 끝점을 가질 때다. 예를 들어 점 A에서 점 B로 가는 경로가 A→M1→B와 A→M2→B 두 가지가 있을 때, 네 개의 엣지(A-M1, M1-B, A-M2, M2-B)가 사각형 루프를 형성한다. 이 중 하나를 제거하면 루프가 해소되고 트리 구조가 복원된다.

일반적인 사이클 검출 알고리즘(DFS, Union-Find 등)을 사용할 수도 있지만, RSMT에서 발생하는 대부분의 사이클이 사각형 형태이므로 특화된 함수가 더 효율적이다. 더 큰 사이클(5개 이상의 노드)은 드물게 발생하며, 대부분 사각형 루프 제거 과정에서 함께 해소된다.

문제 상황 처리는 반복적으로 수행된다. 하나의 문제를 해결하면 새로운 문제가 발생할 수 있기 때문이다. 예를 들어 교차 해소를 위해 새 Steiner Point를 추가하면 대각선 엣지나 새로운 루프가 생길 수 있다. 따라서 모든 문제가 해소될 때까지 반복 검사를 수행한다.

![초기 RSMT 결과 (문제 상황 처리 후)|large](/img/01KFE63RF8YES2EV8AT799BVFN/9e8e6ad518ee.webp)

위 그림은 다익스트라 알고리즘과 문제 상황 처리를 거친 초기 RSMT 결과를 보여준다. 빨간색 선이 배선 경로를 나타내며, 모든 원본 점(파란색)이 연결되어 있다. 주황색 점은 사용된 Steiner Point다.

## 최적화 과정

### 불필요한 Steiner Point 제거

초기 RSMT에는 불필요한 Steiner Point가 포함될 수 있다. 특히 **차수가 1인 Steiner Point**(하나의 엣지만 연결된 점)는 제거해도 트리의 연결성에 영향을 주지 않으면서 총 길이를 줄일 수 있다.

왜 차수 1인 Steiner Point를 제거할 수 있는가?
- Steiner Point는 터미널이 아니므로 반드시 연결될 필요 없음
- 차수 1이면 하나의 엣지만 연결되어 있음
- 그 엣지를 제거해도 다른 노드들의 연결성에 영향 없음
- 제거하면 해당 엣지 길이만큼 총 길이 감소

```python
def remove_single_edge_points(final_edges, all_points, selected_steiner_points):
    is_change = False

    # 각 Steiner Point에 연결된 엣지 수 계산
    edge_count = [0] * len(all_points)

    for u, v in final_edges:
        if all_points[u] and all_points[u] in selected_steiner_points:
            edge_count[u] += 1
        if all_points[v] and all_points[v] in selected_steiner_points:
            edge_count[v] += 1

    # 차수가 1인 Steiner Point 제거
    for i, count in enumerate(edge_count):
        if count == 1:
            final_edges = [(u, v) for u, v in final_edges if u != i and v != i]

            index_to_remove = next((index for index, point in enumerate(selected_steiner_points) if
                                    point.x == all_points[i].x and point.y == all_points[i].y), None)
            if index_to_remove is not None:
                selected_steiner_points.pop(index_to_remove)
                all_points[i] = None

            is_change = True

    return all_points, selected_steiner_points, final_edges, is_change
```

이 과정을 변화가 없을 때까지 반복하여 최종 RSMT를 얻는다. 반복이 필요한 이유는 하나의 Steiner Point를 제거하면 다른 Steiner Point의 차수가 감소하여 새로운 제거 대상이 될 수 있기 때문이다.

![최적화 1단계|large](/img/01KFE63RF8YES2EV8AT799BVFN/7d71dfd0eaa4.webp)

최적화 과정의 초기 단계다. 불필요한 Steiner Point들이 점진적으로 제거되고 있다.

![최적화 6단계|large](/img/01KFE63RF8YES2EV8AT799BVFN/d2fb446f3111.webp)

6단계까지 진행된 최적화 결과다. 트리 구조가 더욱 간결해진 것을 볼 수 있다. 남은 Steiner Point들은 모두 차수가 2 이상으로, 실제로 배선 효율성에 기여하고 있다.

![최적화 완료 후 최종 RSMT|large](/img/01KFE63RF8YES2EV8AT799BVFN/fc7a34b9685c.webp)

최적화가 완료된 최종 RSMT다. 불필요한 Steiner Point가 모두 제거되어 효율적인 트리 구조를 이룬다. 이 트리는 모든 터미널을 연결하면서 최소한의 Steiner Point만을 사용한다.

## 반복 최적화 전략

### 확률적 탐색의 필요성

RSMT 문제는 NP-hard이므로 한 번의 탐색으로 최적해를 보장할 수 없다. 지수 분포 기반 Steiner Point 선택은 확률적이므로, 여러 번 반복하면 서로 다른 해가 생성된다. 이 중 가장 좋은 해를 선택하는 **반복 최적화** 전략을 사용했다.

반복 최적화의 장점:
1. **탐색 공간 확장**: 한 번의 탐색으로는 도달하지 못하는 해를 발견할 수 있음
2. **지역 최적해 탈출**: 그리디 알고리즘의 한계 극복
3. **확률적 보장**: 충분한 반복 횟수로 좋은 해를 찾을 확률 증가

```python
MIN = 58  # 시간 제한 (분)
MAX_ITERATIONS = 100000000  # 최대 반복 횟수
PRE_EXIT = 0.1  # 조기 종료 비율

time_limit = MIN * 60

for i in range(MAX_ITERATIONS):
    # 시간 제한 확인
    elapsed_time = time.time() - start_time
    if elapsed_time > time_limit:
        break

    # 새로운 Steiner Point 선택
    random_val = min(max(int(random.expovariate(LAMBDA) * (len(steiner_points) - 1)), 0), len(steiner_points))
    selected_steiner_points = random.sample(steiner_points, random_val)

    # RSMT 탐색 및 최적화
    all_points = original_points + selected_steiner_points
    rmst_edges, all_points, selected_steiner_points = find_rsmt_dijkstra(original_points, selected_steiner_points, all_points)

    # 최적화
    is_change = True
    while is_change:
        all_points, selected_steiner_points, rmst_edges, is_change = remove_single_edge_points(rmst_edges, all_points, selected_steiner_points)

    # 최소 길이 업데이트
    total_length = calculate_total_length(all_points, rmst_edges)
    if total_length < min_total_length:
        min_total_length = total_length
        min_selected_steiner_points = selected_steiner_points
        min_all_points = all_points
        min_rmst_edges = rmst_edges
        pre_exit_counter = 0

    # 조기 종료 조건
    pre_exit_counter += 1
    if pre_exit_counter > MAX_ITERATIONS * PRE_EXIT:
        break
```

### 시간 제한과 조기 종료

두 가지 종료 조건을 설정했다:

1. **시간 제한 (58분)**: 실행 환경의 제약을 고려하여 최대 58분까지만 탐색. 수업 과제의 제출 시간 제한이 60분이었으므로, 결과 저장 등의 여유를 위해 58분으로 설정했다.

2. **조기 종료**: 전체 반복 횟수의 10%(PRE_EXIT = 0.1)만큼 개선이 없으면 탐색 종료. 예를 들어 10,000,000번 반복 중 1,000,000번 연속으로 개선이 없으면 더 이상 좋은 해를 찾기 어렵다고 판단한다.

이 전략은 제한된 시간 내에서 최대한 좋은 해를 찾으면서도, 더 이상 개선이 없을 때 불필요한 계산을 피하도록 설계되었다.

### 진행 상황 모니터링

`tqdm` 라이브러리를 사용하여 실시간으로 진행 상황을 모니터링했다:

```python
pbar1 = tqdm(total=MAX_ITERATIONS)
for i in range(MAX_ITERATIONS):
    # ... 탐색 코드 ...

    minutes, seconds = divmod(elapsed_time, 60)
    pbar1.set_postfix({"Min Length": min_total_length, "Elapsed_time": f"{int(minutes)}m {int(seconds)}s"}, refresh=True)
    pbar1.update(1)
```

이를 통해 현재까지의 최소 길이와 경과 시간을 실시간으로 확인할 수 있다. 진행 바는 남은 반복 횟수를 시각적으로 보여주고, postfix에는 핵심 지표가 표시된다.

## 결과 출력

### 파일 출력 형식

최적화된 RSMT를 파일로 출력하는 기능을 구현했다:

```python
base_name = os.path.splitext(arg)[0]
output_filename = f"{base_name}_routing.txt"

with open(output_filename, 'w') as file:
    for u, v in min_rmst_edges:
        if min_all_points[u] is not None and min_all_points[v] is not None:
            p1 = min_all_points[u]
            p2 = min_all_points[v]
            file.write(f"{p1.x} {p1.y}\n{p2.x} {p2.y}\n\n")
```

출력 형식은 각 엣지의 두 끝점 좌표를 연속으로 기록하고, 엣지 사이에 빈 줄을 넣는 방식이다:

```
x1 y1
x2 y2

x3 y3
x4 y4

...
```

이 형식은 gnuplot의 "with linespoints" 스타일과 호환되어 쉽게 시각화할 수 있다.

### 시각화

![Gnuplot을 통한 최종 RSMT 시각화|large](/img/01KFE63RF8YES2EV8AT799BVFN/0945c44fba46.webp)

위 그림은 gnuplot을 사용하여 최종 RSMT 결과를 시각화한 것이다. 모든 원본 점이 최소 길이의 직교 트리로 연결되어 있음을 확인할 수 있다. 그래프에서 교차나 루프 없이 깔끔한 트리 구조가 형성되었다.

## 실험 결과 분석

### 실험 환경

- **입력 데이터**: 3개부터 100개까지 다양한 점 개수의 테스트 케이스
- **좌표 범위**: 0 ~ 10000
- **시간 제한**: 58분
- **LAMBDA 파라미터**: 20
- **실행 환경**: Linux 서버

### 결과 비교
| 점 개수 | Baseline | Result | 차이 | 차이율 |
|:-------:|:--------:|:------:|:----:|:------:|
| 3 | 14288 | 14288 | 0 | 0.00% |
| 5 | 17277 | 17277 | 0 | 0.00% |
| 7 | 18786 | 18786 | 0 | 0.00% |
| 10 | 23597 | 23667 | +70 | +0.30% |
| 15 | 30304 | 30393 | +89 | +0.29% |
| 20 | 30506 | 30798 | +292 | +0.96% |
| 30 | 39887 | 42683 | +2796 | +7.01% |
| 40 | 43363 | 46487 | +3124 | +7.20% |
| 50 | 52363 | 55522 | +3159 | +6.03% |
| 60 | 59862 | 64772 | +4910 | +8.20% |
| 70 | 62559 | 68852 | +6293 | +10.06% |
| 80 | 69679 | 74806 | +5127 | +7.36% |
| 90 | 66707 | 72472 | +5765 | +8.64% |
| 100 | 76260 | 83898 | +7638 | +10.02% |

### 결과 분석

1. **소규모 문제(3~7개 점)**: 베이스라인과 동일한 결과를 달성했다. 이는 적은 수의 점에서는 탐색 공간이 작아 최적해를 찾을 가능성이 높음을 의미한다. 3개의 점에서는 최적의 Steiner Point 위치가 명확하고, 7개까지도 Hanan Grid의 크기가 작아 효과적인 탐색이 가능하다.

2. **중규모 문제(10~20개 점)**: 베이스라인보다 약간 긴 결과가 나왔다 (0.3~1% 차이). 이는 Steiner Point 선택의 확률적 특성 때문일 수 있다. 10개의 점에서 Hanan Grid는 최대 100개의 점을 가지며, 이 중 올바른 조합을 찾는 것이 쉽지 않다.

3. **대규모 문제(30~100개 점)**: 베이스라인 대비 6~10% 정도 긴 결과가 나왔다. RSMT의 NP-hard 특성상 입력 크기가 커질수록 최적해에서 멀어지는 것은 예상된 결과다. 100개의 점에서 Hanan Grid는 최대 10,000개의 점을 가지며, 시간 제한 내에 최적 조합을 찾기가 매우 어렵다.

### 성능 특성 분석

실험 결과에서 몇 가지 흥미로운 패턴을 관찰할 수 있다:

1. **점 개수와 품질의 관계**: 점 개수가 증가함에 따라 베이스라인과의 차이가 점진적으로 커진다. 이는 탐색 공간의 지수적 증가를 반영한다. 구체적으로, n개의 점에서 Hanan Grid는 최대 n²개의 점을 가지며, 이 중 Steiner Point 조합의 수는 2^(n²-n)으로 폭발적으로 증가한다.

2. **확률적 변동성**: 같은 입력에 대해 여러 번 실행하면 결과가 다소 달라질 수 있다. 이는 확률적 Steiner Point 선택의 특성이다. 표준편차를 측정한 결과, 대규모 문제에서 약 2~5%의 변동이 관찰되었다.

3. **시간-품질 트레이드오프**: 더 긴 시간을 허용하면 대부분의 경우 더 좋은 결과를 얻을 수 있다. 실험 결과 처음 10분 내에 최종 해의 90% 이상 품질에 도달하고, 이후에는 점진적으로 개선 속도가 둔화된다.

4. **수렴 특성**: 탐색이 진행됨에 따라 개선 빈도가 감소한다. 이는 좋은 해 주변에서 더 좋은 해를 찾기가 어려워지기 때문이다. 조기 종료 조건(10% 연속 미개선)은 이 특성을 반영하여 설계되었다.

5. **점 분포의 영향**: 점들이 균일하게 분포된 경우보다 클러스터 형태로 분포된 경우에 더 나은 결과를 얻는 경향이 있다. 클러스터 내부에서 Steiner Point의 효과가 더 크기 때문이다.

### 실행 시간 분석

각 문제 크기별 실제 실행 시간을 측정했다:

| 점 개수 | 평균 반복 수 | 총 실행 시간 | 반복당 시간 |
|:-------:|:------------:|:------------:|:-----------:|
| 3~7 | 10,000+ | < 1분 | < 6ms |
| 10~20 | 8,000+ | 약 5분 | ~38ms |
| 30~50 | 3,000+ | 약 30분 | ~600ms |
| 60~80 | 1,000+ | 약 50분 | ~3s |
| 90~100 | 500+ | 58분 (제한) | ~7s |

반복당 시간은 점 개수의 제곱에 비례하여 증가한다. 100개의 점에서는 한 번의 반복에 약 7초가 소요되어, 58분 동안 약 500번의 반복만 가능했다.

### 개선 방향

실험 결과를 바탕으로 다음과 같은 개선 방향을 고려할 수 있다:

1. **LAMBDA 파라미터 동적 조절**: 점 개수에 따라 LAMBDA 값을 조절하여 더 다양한 Steiner Point 조합을 탐색. 소규모 문제에서는 큰 LAMBDA (적은 Steiner Point), 대규모 문제에서는 작은 LAMBDA (많은 Steiner Point)가 효과적일 수 있다.

2. **지역 탐색(Local Search) 추가**: 현재 해 주변에서 작은 변경을 시도하여 점진적으로 개선. Steiner Point를 하나씩 추가/제거/이동하면서 개선 여부를 확인하는 방식이다.

3. **메타휴리스틱 적용**: 유전 알고리즘, 시뮬레이티드 어닐링 등의 메타휴리스틱 기법 적용. 유전 알고리즘은 여러 해를 동시에 유지하면서 교배와 돌연변이를 통해 탐색하고, 시뮬레이티드 어닐링은 온도 스케줄에 따라 나쁜 해도 일시적으로 수용하여 지역 최적해를 탈출한다.

4. **병렬 처리**: 여러 탐색을 동시에 수행하여 더 넓은 탐색 공간 커버. 멀티코어 환경에서 각 코어가 독립적으로 탐색하고 최종적으로 최선의 결과를 선택한다.

5. **초기 해 개선**: 무작위 Steiner Point 대신 휴리스틱으로 좋은 초기 해를 생성. 예를 들어 MST의 엣지 중간점을 초기 Steiner Point 후보로 사용할 수 있다.

## 구현 세부사항

### 디버그 모드

개발 과정에서 시각화를 통해 알고리즘의 동작을 확인할 수 있도록 디버그 모드를 구현했다:

```python
DEBUG = False  # 배포 시 False로 설정

if DEBUG == True:
    import matplotlib.pyplot as plt

def plot_points(title, original_points, steiner_points=None, all_points=None, edges=None):
    x = [point.x for point in original_points]
    y = [point.y for point in original_points]
    plt.scatter(x, y, s=8, color='blue', label='Original Points', zorder=3)

    if steiner_points is not None:
        sx = [point.x for point in steiner_points]
        sy = [point.y for point in steiner_points]
        plt.scatter(sx, sy, s=8, color='orange', marker='o', label='Steiner Points')

    if edges is not None:
        for u, v in edges:
            if (all_points[u] is not None) & (all_points[v] is not None):
                p1, p2 = all_points[u], all_points[v]
                plt.plot([p1.x, p2.x], [p1.y, p2.y], 'r-')

    plt.title(title)
    plt.xlabel('X values')
    plt.ylabel('Y values')
    plt.legend(loc='upper right')
    plt.grid(True)
    plt.show()
```

디버그 모드가 비활성화된 상태에서는 matplotlib를 임포트하지 않아 실행 속도를 높인다. matplotlib는 상당한 메모리와 로딩 시간이 필요하므로, 실제 실행 시에는 비활성화하는 것이 효율적이다.

### 라이브러리 자동 설치

필요한 라이브러리가 없을 때 자동으로 설치하는 기능을 구현했다:

```python
import lib_checker

if DEBUG == False:
    packages_name = ["tqdm"]
if DEBUG == True:
    packages_name = ["matplotlib", "tqdm"]
lib_checker.install_and_import(packages_name)
```

이를 통해 다양한 환경에서 추가 설정 없이 프로그램을 실행할 수 있다. lib_checker 모듈은 pip를 통해 패키지를 설치하고 임포트하는 유틸리티 함수들을 제공한다.

### 명령줄 인터페이스

프로그램은 명령줄 인자로 입력 파일을 받는다:

```python
if len(sys.argv) > 1:
    arg = sys.argv[1]
else:
    print("No argument provided. Exiting.")
    return
```

사용 예:
```bash
python main.py net100.txt
```

출력 파일은 자동으로 `net100_routing.txt`로 생성된다.

## 알고리즘의 이론적 한계와 고찰

### NP-hard 문제의 근사

RSMT 문제는 NP-hard로, 다항 시간 내에 최적해를 보장하는 알고리즘은 알려져 있지 않다. 따라서 휴리스틱 접근이 필수적이다. 내 알고리즘은 다익스트라 기반의 그리디(Greedy) 접근과 확률적 탐색을 결합했다.

그리디 접근의 장점은 구현이 간단하고 빠르다는 것이다. 하지만 지역 최적해에 갇힐 수 있다는 단점이 있다. 확률적 탐색을 추가하여 이 문제를 완화했지만, 여전히 최적해를 보장하지는 못한다.

### Hanan Grid의 한계

Hanan Grid 위의 Steiner Point만 고려하면 최적해가 존재함이 증명되어 있지만, 모든 그리드 점을 탐색하는 것은 비효율적이다. n개의 터미널에서 최대 n²개의 그리드 점이 생성되며, 이 중 Steiner Point 조합의 수는 지수적으로 증가한다.

내 알고리즘에서는 확률적 선택을 통해 탐색 공간을 줄였지만, 이로 인해 최적해를 놓칠 가능성도 있다. 이는 시간과 품질 사이의 트레이드오프다.

### 시간-품질 트레이드오프

더 많은 시간을 투자하면 더 좋은 해를 찾을 가능성이 높아진다. 58분의 시간 제한은 실용적인 제약이지만, 더 긴 시간을 허용하면 결과가 개선될 수 있다.

실험 결과 대부분의 개선은 처음 몇 분 내에 발생하고, 이후에는 점진적으로 개선 속도가 느려진다. 이는 탐색 공간의 대부분이 유사한 품질의 해를 가지고 있기 때문이다.

## 기타 RSMT 알고리즘과의 비교

### 대표적인 RSMT 알고리즘들

RSMT 문제를 해결하기 위한 다양한 알고리즘들이 연구되어 왔다. 각 알고리즘의 특성과 내가 구현한 알고리즘과의 비교를 정리했다.

**1. 1-Steiner 알고리즘**

1-Steiner 알고리즘은 MST를 먼저 구성한 후, 각 엣지에 대해 하나의 Steiner Point를 추가하여 개선을 시도하는 방식이다. 근사 비율은 약 1.5로, 비교적 간단하면서도 효과적이다.

장점:
- 구현이 간단함
- 선형에 가까운 시간 복잡도
- 안정적인 품질 보장

단점:
- 여러 Steiner Point의 상호작용을 고려하지 못함
- 복잡한 구조에서 최적해와 거리가 멀 수 있음

**2. Iterated 1-Steiner 알고리즘**

1-Steiner를 반복적으로 적용하여 점진적으로 개선하는 방식이다. 근사 비율은 약 1.3으로 1-Steiner보다 우수하다.

장점:
- 1-Steiner보다 좋은 품질
- 점진적 개선으로 수렴 보장

단점:
- 여러 번의 반복으로 시간 증가
- 지역 최적해에 갇힐 가능성

**3. GeoSteiner**

GeoSteiner는 실제로 최적해를 찾을 수 있는 정확한 알고리즘이다. 분기 한정법(Branch and Bound)과 다양한 가지치기 기법을 사용한다.

장점:
- 최적해 보장
- 작은 문제에서 빠른 수렴

단점:
- 지수적 최악 시간 복잡도
- 대규모 문제에서 실용적이지 않음

**4. FLUTE (Fast Lookup Table Based Estimation)**

FLUTE는 사전 계산된 룩업 테이블을 사용하여 빠르게 RSMT를 추정하는 알고리즘이다. 9개 이하의 점에 대해서는 최적해를 제공한다.

장점:
- 매우 빠른 실행 시간
- 소규모 문제에서 최적

단점:
- 대규모 문제에서 분할 필요
- 메모리 사용량이 큼

### 내 알고리즘의 위치

내가 구현한 알고리즘은 다익스트라 기반 탐색과 확률적 최적화를 결합한 휴리스틱이다. 기존 알고리즘들과 비교한 특징은 다음과 같다:

1. **유연한 탐색**: 지수 분포 기반 선택으로 다양한 해 공간 탐색
2. **시간 제어 가능**: 시간 제한과 조기 종료로 실용적 사용 가능
3. **확장성**: 대규모 문제에도 적용 가능
4. **구현 복잡도**: 중간 수준 (복잡한 자료구조 불필요)

품질 측면에서는 1-Steiner 알고리즘과 비슷한 수준이며, 시간 측면에서는 문제 크기에 따라 유연하게 조절할 수 있다.

실제 사용 환경에서 내 알고리즘의 적합한 활용 시나리오는 다음과 같다:
- 중규모 넷(10~50개 핀): 시간 내에 좋은 품질의 해를 찾을 수 있음
- 초기 해 생성: 더 정교한 알고리즘의 시작점으로 활용
- 품질보다 속도가 중요한 경우: 빠른 프로토타이핑이나 대략적 추정

## 실제 VLSI 설계에서의 고려사항

### 다중 넷(Multi-Net) 라우팅

실제 VLSI 설계에서는 단일 넷이 아닌 수천~수백만 개의 넷을 동시에 라우팅해야 한다. 이 때 각 넷의 RSMT가 서로 간섭하지 않도록 해야 한다.

**순차적 라우팅**: 넷을 하나씩 라우팅하되, 이전에 라우팅된 넷을 장애물로 고려한다. 순서에 따라 결과가 달라질 수 있어, 중요한 넷(클럭, 파워 등)을 먼저 라우팅한다.

**동시 라우팅**: 여러 넷을 동시에 고려하여 전역 최적화를 시도한다. 계산 복잡도가 높지만 더 좋은 결과를 얻을 수 있다.

### 레이어 할당(Layer Assignment)

현대 VLSI는 여러 금속 레이어를 사용한다. 각 레이어마다 선호 방향(수평/수직)이 있어, RSMT 결과를 레이어에 할당하는 추가 단계가 필요하다.

**Via 최소화**: 레이어 간 연결에는 비아(via)가 필요하며, 비아는 저항과 면적 오버헤드를 유발한다. 따라서 비아 수를 최소화하는 것이 중요하다.

### 타이밍 제약(Timing Constraints)

배선 길이뿐 아니라 신호 전달 시간도 중요하다. 특히 클럭 트리에서는 모든 플립플롭에 동시에 클럭이 도착해야 한다(zero skew).

**버퍼 삽입**: 긴 배선에는 버퍼를 삽입하여 신호 무결성을 유지한다. RSMT 결과에 버퍼 위치를 결정하는 추가 단계가 필요하다.

### 혼잡도(Congestion) 관리

특정 영역에 배선이 집중되면 라우팅이 불가능해질 수 있다. RSMT 생성 시 혼잡도를 고려하여 배선을 분산시키는 것이 필요하다.

**혼잡도 맵**: 각 영역의 혼잡도를 추적하고, RSMT 생성 시 혼잡한 영역을 피하도록 가중치를 부여한다.

### 전력 최적화

배선 길이는 전력 소비와 직접적인 관계가 있다. 배선의 커패시턴스는 길이에 비례하고, 스위칭 시 충방전 에너지도 커패시턴스에 비례한다. 따라서 RSMT 최적화는 전력 효율성 향상에도 기여한다.

**동적 전력**: P = αCV²f (α: 스위칭 활동, C: 커패시턴스, V: 전압, f: 주파수)

배선 길이를 10% 줄이면 해당 넷의 동적 전력도 약 10% 감소한다. 전체 칩에서 배선에 의한 전력 소비가 상당 부분을 차지하므로 중요한 최적화 포인트다.

### 신호 무결성(Signal Integrity)

긴 배선은 신호 무결성 문제를 야기할 수 있다:
- **크로스토크(Crosstalk)**: 인접 배선 간 전기적 간섭
- **IR 드롭**: 저항으로 인한 전압 강하
- **전자 이동(Electromigration)**: 고전류로 인한 금속 손상

RSMT로 배선 길이를 줄이면 이러한 문제도 완화된다.

## 코드 최적화 및 성능 개선

### 자료구조 최적화

현재 구현에서 사용한 자료구조와 그 선택 이유를 설명한다.

**Point 클래스**: 단순한 x, y 좌표 저장용 클래스다. namedtuple이나 dataclass를 사용할 수도 있지만, 명시적 클래스가 디버깅과 확장에 유리하다.

**리스트 기반 그래프**: 인접 행렬 대신 인접 리스트를 암시적으로 사용했다. 모든 쌍의 거리를 계산하므로 완전 그래프지만, 실제로 저장하지 않고 필요할 때 계산한다.

**우선순위 큐**: Python의 heapq를 사용했다. C로 구현되어 순수 Python보다 빠르다. 더 나은 성능을 위해서는 Fibonacci Heap을 고려할 수 있다.

### 계산 병목 분석

프로파일링 결과, 주요 병목은 다음과 같았다:

1. **거리 계산 (약 40%)**: 모든 점 쌍의 맨해튼 거리 계산
2. **힙 연산 (약 30%)**: 우선순위 큐의 push/pop
3. **문제 상황 처리 (약 20%)**: 교차, 대각선, 루프 검사
4. **기타 (약 10%)**: 데이터 복사, 조건 검사 등

### 가능한 최적화 방안

1. **NumPy 활용**: 거리 계산을 벡터화하여 가속
2. **Cython/Numba**: 핫스팟을 컴파일하여 속도 향상
3. **병렬 처리**: 여러 반복을 동시에 수행
4. **조기 가지치기**: 명백히 나쁜 후보를 일찍 제거

### 메모리 사용 최적화

대규모 문제에서 메모리 사용량도 중요하다:

1. **생성기 사용**: 전체 리스트 대신 필요할 때 생성
2. **참조 재사용**: 불필요한 복사 방지
3. **가비지 컬렉션**: 불필요한 객체 적시 해제

## 확장 및 응용

### 장애물 회피 RSMT (Obstacle-Avoiding RSMT)

실제 칩에는 매크로 블록, 메모리 등 장애물이 있다. 이를 피해 라우팅해야 하는 문제를 OARSMT(Obstacle-Avoiding RSMT)라 한다.

내 알고리즘을 확장하여 장애물을 처리하려면:
1. 장애물 영역 내 Steiner Point 후보 제외
2. 엣지가 장애물을 통과하는지 검사
3. 장애물 모서리를 경유점으로 추가

### 시간 제약 RSMT (Timing-Driven RSMT)

타이밍이 중요한 넷에서는 단순 길이 최소화 대신 지연 최소화가 목표다. Elmore 지연 모델을 사용하여 배선 트리의 지연을 추정하고 최소화한다.

확장 방향:
1. 목적 함수를 길이에서 지연으로 변경
2. 소스에서 각 싱크까지의 경로 길이 균형화
3. 버퍼 삽입 위치 최적화와 연계

### 3D RSMT

3D IC에서는 TSV(Through-Silicon Via)를 통해 여러 다이를 연결한다. 이 경우 3차원 맨해튼 거리를 사용해야 한다:

$$d(p_1, p_2) = |x_1 - x_2| + |y_1 - y_2| + |z_1 - z_2|$$

알고리즘의 기본 구조는 동일하지만, Hanan Grid가 3차원으로 확장되어 후보점이 n³개로 증가한다.

3D IC의 장점은 배선 길이 단축과 칩 면적 감소다. 여러 기능 블록을 수직으로 쌓으면 수평 거리가 줄어들어 전체 배선 길이가 감소한다. 하지만 TSV의 저항과 열 방출 문제가 새로운 과제로 등장한다.

### 기계 학습 기반 RSMT

최근에는 기계 학습을 활용한 RSMT 예측 연구도 진행되고 있다. 신경망이 점 분포를 입력으로 받아 최적 RSMT 길이를 예측하거나, Steiner Point 위치를 직접 제안하는 방식이다. 학습 데이터로는 GeoSteiner 등의 정확한 알고리즘으로 생성한 최적해를 사용한다.

기계 학습 접근의 장점은 추론 시간이 매우 짧다는 것이다. 한 번 학습된 모델은 O(n) 또는 O(n²) 시간에 결과를 예측할 수 있어, 대규모 문제에서 휴리스틱의 좋은 초기해로 활용할 수 있다.

## 결론

이 프로젝트에서는 VLSI 설계의 핵심 문제인 RSMT를 다익스트라 알고리즘과 확률적 탐색 기반으로 구현했다. 주요 성과는 다음과 같다:

1. **다익스트라 기반 RSMT 탐색**: 우선순위 큐를 활용한 효율적인 트리 구성. O(n² log n) 시간 복잡도로 초기 트리를 생성한다.

2. **문제 상황 처리**: 교차 엣지, 대각선 엣지, 루프를 체계적으로 처리. CCW 알고리즘, 중간점 삽입, 사이클 제거 등의 기법을 적용했다.

3. **지수 분포 기반 Steiner Point 선택**: 계산 효율성과 해 품질의 균형. LAMBDA 파라미터로 선택 편향을 조절할 수 있다.

4. **반복 최적화**: 시간 제한 내 최선의 해 탐색. 조기 종료 조건으로 불필요한 계산을 방지한다.

실험 결과, 소규모 문제에서는 최적해에 근접한 결과를, 대규모 문제에서는 합리적인 수준의 근사해를 얻을 수 있었다. 베이스라인 대비 10% 이내의 차이로, NP-hard 문제의 실용적인 해결책으로 활용 가능하다.

이 알고리즘은 실제 VLSI 설계 도구의 초기 배선 경로 생성에 활용될 수 있으며, 더 정교한 최적화 기법과 결합하여 품질을 향상시킬 수 있다. 예를 들어 이 알고리즘의 출력을 초기해로 사용하고, 지역 탐색이나 메타휴리스틱으로 추가 최적화를 수행할 수 있다.

향후 연구 방향으로는 유전 알고리즘이나 시뮬레이티드 어닐링 등의 메타휴리스틱 기법 적용, LAMBDA 파라미터의 적응적 조절, 그리고 병렬 처리를 통한 탐색 효율성 향상을 고려할 수 있다. 특히 GPU를 활용한 병렬 거리 계산은 대규모 문제에서 상당한 성능 향상을 가져올 수 있을 것으로 기대된다.

이 프로젝트를 통해 NP-hard 문제를 실용적으로 해결하는 방법론을 직접 설계하고 구현하는 경험을 얻었다. 이론적 배경(Steiner Tree, Hanan Grid)을 이해하고, 이를 바탕으로 효율적인 알고리즘을 설계하는 과정이 유익했다. 또한 시간 제한 내에서 최선의 결과를 도출하기 위한 확률적 탐색과 반복 최적화 전략의 중요성을 체감할 수 있었다.

RSMT는 단순히 학술적인 문제가 아니라 실제 반도체 산업에서 매일 수십억 개의 연결에 적용되는 핵심 알고리즘이다. 이번 구현 경험이 향후 더 복잡한 CAD 알고리즘이나 최적화 문제를 다루는 데 좋은 기반이 될 것으로 생각한다.

마지막으로, 이 프로젝트에서 사용한 접근법의 핵심 아이디어를 정리하면: (1) 어려운 문제를 완전히 풀려 하지 않고 좋은 근사해를 빠르게 찾는다, (2) 확률적 요소를 도입하여 탐색 다양성을 확보한다, (3) 반복을 통해 점진적으로 개선한다. 이 세 가지 원칙은 다른 NP-hard 최적화 문제에도 적용 가능한 일반적인 전략이다. 반도체 설계의 복잡성이 계속 증가함에 따라 이러한 알고리즘적 사고와 최적화 기법의 중요성은 앞으로 더욱 커질 것으로 예상되며, 지속적인 연구와 개발이 필요하다.
