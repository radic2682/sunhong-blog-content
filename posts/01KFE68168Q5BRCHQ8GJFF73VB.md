---
id: "01KFE68168Q5BRCHQ8GJFF73VB"
slug: "nell-knowledge-graph-gcn"
title: "지식 그래프 기반 노드 분류: GCN을 활용한 NELL 데이터셋 분석"
category: "01M651PC004D9T04SN9SXTH5R6"
tags: ["GNN","GCN","NELL","Knowledge Graph","Node Classification","PyTorch Geometric"]
summary: "NELL 지식 그래프 데이터셋에서 GCN(Graph Convolutional Network)을 활용하여 노드 분류를 수행했다. PyTorch Geometric으로 모델을 구현하고, 65,755개 노드와 186개 클래스에 대해 97.86%의 테스트 정확도를 달성했다."
thumbnail: "/img/01KFE68168Q5BRCHQ8GJFF73VB/dd78ad32088c.webp"
note: false
created: "2026-01-21T02:11:41+09:00"
updated: "2026-03-28T02:23:03+09:00"
---
## 1. 프로젝트 개요

### 프로젝트 배경 및 목표

그래프 신경망(Graph Neural Networks) 수업의 과제로 NELL(Never-Ending Language Learning) 지식 그래프 데이터셋에서 GCN(Graph Convolutional Network)을 활용한 노드 분류 모델을 직접 구현하고 성능을 분석하였다. 단순히 라이브러리를 호출하는 것에 그치지 않고, 데이터셋의 구조적 특성을 심층 분석하고 그래프 신경망의 작동 원리를 이해하는 것이 목표였다.

특히 다음 세 가지 질문에 답하고자 하였다:
- 대규모 지식 그래프에서 GNN이 효과적으로 노드의 의미적 카테고리를 학습할 수 있는가?
- 186개의 클래스와 심각한 클래스 불균형 상황에서 어떤 성능을 달성할 수 있는가?
- 그래프 구조 정보가 노드 분류에 얼마나 기여하는가?

구현한 모델을 NELL 데이터셋에서 학습하고 평가하여 **최종 97.86%의 테스트 정확도**를 달성하였다. 이 과정에서 지식 그래프의 구조적 특성, GCN의 메시지 패싱 메커니즘, 그리고 대규모 희소 그래프에서의 학습 전략을 깊이 이해할 수 있었다.

### 그래프 신경망(GNN)이란?

#### 그래프 데이터의 특성

전통적인 머신러닝과 딥러닝은 주로 유클리드 공간의 데이터를 다룬다. 이미지는 2D 그리드, 텍스트는 1D 시퀀스로 표현되며, CNN이나 RNN과 같은 아키텍처가 이러한 구조화된 데이터에 특화되어 있다. 그러나 현실 세계의 많은 데이터는 비유클리드(non-Euclidean) 구조를 가진다:

- **소셜 네트워크**: 사용자(노드)와 친구 관계(엣지)
- **분자 구조**: 원자(노드)와 화학 결합(엣지)
- **지식 그래프**: 엔티티(노드)와 관계(엣지)
- **추천 시스템**: 사용자-아이템 상호작용 그래프

이러한 그래프 데이터는 노드의 개수가 가변적이고, 노드 간 연결 패턴이 불규칙하며, 순서가 정해져 있지 않다는 특성이 있다. 전통적인 신경망을 직접 적용하기 어려운 이유이다.

#### 메시지 패싱 패러다임

GNN은 그래프 구조 데이터를 처리하기 위해 **메시지 패싱(Message Passing)** 패러다임을 사용한다. 각 노드는 이웃 노드들로부터 정보를 수집(Aggregate)하고, 자신의 표현을 업데이트(Update)한다. 이 과정을 여러 층에 걸쳐 반복하면 노드의 표현은 점점 더 넓은 이웃의 정보를 담게 된다.

메시지 패싱의 일반적인 형태는 다음과 같다:

$$h_v^{(k)} = \text{UPDATE}^{(k)}\left(h_v^{(k-1)}, \text{AGGREGATE}^{(k)}\left(\{h_u^{(k-1)} : u \in \mathcal{N}(v)\}\right)\right)$$

여기서:
- $h_v^{(k)}$: k번째 층에서 노드 v의 표현
- $\mathcal{N}(v)$: 노드 v의 이웃 집합
- AGGREGATE: 이웃 노드들의 정보를 모으는 함수 (예: 평균, 합, 최대값)
- UPDATE: 집계된 정보와 자신의 정보를 결합하는 함수

#### GNN의 표현력

GNN은 **Weisfeiler-Leman(WL) 그래프 동형 테스트**와 밀접한 관련이 있다. WL 테스트는 두 그래프가 구조적으로 동일한지 판별하는 알고리즘으로, 반복적으로 노드 라벨을 이웃 정보로 업데이트한다. 표준 GNN(GCN, GraphSAGE 등)의 표현력은 1-WL 테스트와 동등하다는 것이 이론적으로 증명되었다.

이는 GNN이 구별할 수 있는 그래프 구조에 한계가 있음을 의미하지만, 실제 응용에서는 대부분의 경우 충분한 표현력을 제공한다.

### Graph Convolutional Network (GCN)

#### GCN의 핵심 아이디어

2017년 Kipf와 Welling이 제안한 GCN은 스펙트럼 그래프 이론을 기반으로 한다. 그래프의 라플라시안(Laplacian) 행렬에 대한 고유값 분해를 통해 그래프 신호를 주파수 영역으로 변환하고, 필터를 적용한 후 다시 노드 영역으로 변환한다. 이 과정을 체비셰프 다항식으로 근사하고 1차 항까지만 사용하면 다음과 같은 간단한 형태가 된다:

$$H^{(l+1)} = \sigma\left(\tilde{D}^{-1/2}\tilde{A}\tilde{D}^{-1/2}H^{(l)}W^{(l)}\right)$$

여기서:
- $\tilde{A} = A + I$: 셀프 루프가 추가된 인접 행렬
- $\tilde{D}$: $\tilde{A}$의 차수 행렬
- $H^{(l)}$: l번째 층의 노드 표현 행렬
- $W^{(l)}$: 학습 가능한 가중치 행렬
- $\sigma$: 비선형 활성화 함수

#### 정규화의 의미

$\tilde{D}^{-1/2}\tilde{A}\tilde{D}^{-1/2}$ 항은 **대칭 정규화(symmetric normalization)**를 수행한다. 이 정규화가 중요한 이유는:

1. **스케일 불변성**: 차수가 높은 노드와 낮은 노드 간의 정보 전파량을 균형있게 조절
2. **수치 안정성**: 특성값이 [-1, 1] 범위로 제한되어 학습 안정성 확보
3. **양방향 고려**: 송신자와 수신자 양쪽의 차수를 모두 고려

직관적으로, 차수가 매우 높은 허브 노드의 영향력이 지나치게 커지는 것을 방지하고, 차수가 낮은 노드도 의미 있는 정보를 전달할 수 있게 한다.

#### GCN vs 다른 GNN 변형

GCN 외에도 다양한 GNN 변형이 존재한다:

| 모델 | 집계 함수 | 특징 |
|------|----------|------|
| GCN | 정규화된 평균 | 스펙트럼 기반, 간단하고 효율적 |
| GraphSAGE | 평균/LSTM/풀링 | 인덕티브 학습 가능, 샘플링 기반 |
| GAT | 어텐션 가중 평균 | 이웃별 가중치 학습, 해석 가능 |
| GIN | 합 + MLP | 최대 표현력 (1-WL 동등) |

이번 프로젝트에서는 GCN을 선택하였다. 지식 그래프에서 노드 분류 태스크에 GCN이 효과적이며, 구현이 직관적이고 계산 효율성이 높기 때문이다.

### 지식 그래프란?

#### 정의와 구조

지식 그래프(Knowledge Graph)는 실세계의 지식을 그래프 형태로 표현한 것이다. 노드는 **엔티티(entity)**를, 엣지는 엔티티 간의 **관계(relation)**를 나타낸다. 일반적으로 (head, relation, tail) 형태의 트리플로 지식을 표현한다.

예시:
- (서울, 수도, 대한민국)
- (아인슈타인, 출생지, 울름)
- (파이썬, 프로그래밍 언어, 소프트웨어)

#### 지식 그래프의 응용

지식 그래프는 다양한 분야에서 활용된다:

- **검색 엔진**: Google Knowledge Graph, Bing Satori
- **추천 시스템**: 아이템 간 관계 기반 추천
- **질의응답**: 구조화된 지식 기반 답변 생성
- **자연어처리**: 엔티티 링킹, 관계 추출

#### NELL 프로젝트

NELL(Never-Ending Language Learning)은 Carnegie Mellon University에서 2010년부터 운영 중인 연구 프로젝트이다. 웹에서 지속적으로 텍스트를 읽고 지식을 추출하여 지식 베이스를 구축하는 것이 목표이다.

NELL의 특징:
- **자동 지식 추출**: 웹 문서에서 패턴 기반으로 지식 추출
- **지속적 학습**: 24/7 운영하며 지식을 계속 확장
- **자기 감독**: 이미 알고 있는 지식을 활용하여 새로운 지식 검증
- **다중 데이터 소스**: 텍스트, 테이블, 이미지 등 다양한 소스 활용

NELL 데이터셋은 이 프로젝트에서 추출된 지식 그래프를 그래프 학습용으로 가공한 것이다. 엔티티는 노드로, 관계는 엣지로 표현되며, 각 엔티티의 의미적 카테고리(예: 도시, 운동선수, 약품 등)가 노드 레이블로 제공된다.

### 개발 환경

- **프레임워크**: PyTorch 2.0, PyTorch Geometric
- **그래프 분석**: NetworkX
- **시각화**: Matplotlib, Seaborn
- **하드웨어**: NVIDIA GPU (CUDA)
- **주요 라이브러리 버전**:
  - torch_geometric: 최신 버전
  - networkx: 3.x
  - numpy, scipy: 과학 계산용

---

## 2. 데이터셋 분석 (Task 1)

### NELL 데이터셋 로드

PyTorch Geometric에서 제공하는 NELL 데이터셋을 사용하였다. 데이터셋 로드 코드는 다음과 같다:

```python
from torch_geometric.datasets import NELL

dataset = NELL(root='./data/NELL')
data = dataset[0]

print(f"Nodes: {data.num_nodes:,}, Edges: {data.num_edges:,}")
print(f"Features: {data.num_node_features}, Classes: {dataset.num_classes}")
```

로드 결과, NELL 데이터셋의 기본 통계는 다음과 같다:

| 항목 | 값 |
|------|-----|
| 노드 수 | 65,755 |
| 엣지 수 | 251,550 |
| 특성 차원 | 61,278 |
| 클래스 수 | 186 |

65,755개의 엔티티가 251,550개의 관계로 연결되어 있으며, 각 엔티티는 186개의 의미적 카테고리 중 하나에 속한다. 특성 벡터의 차원이 61,278로 매우 높은데, 이는 NELL의 지식 추출 과정에서 다양한 텍스트 패턴과 맥락 정보를 인코딩하기 때문이다.

### 데이터 구조 탐색

#### 희소 특성 행렬

NELL 데이터셋의 노드 특성은 **희소 행렬(sparse matrix)** 형태로 저장되어 있다. 61,278차원의 밀집 벡터를 모든 노드에 대해 저장하면 메모리 사용량이 약 16GB에 달하기 때문이다.

```python
# 희소 행렬을 밀집 행렬로 변환
x_dense = data.x.to_dense() if hasattr(data.x, 'to_dense') else data.x
print(f"Feature matrix shape: {x_dense.shape}")
```

특성 행렬 분석 결과:
- **형태**: (65,755, 61,278)
- **희소도(Sparsity)**: 99.87% (대부분의 값이 0)
- **노드당 평균 non-zero 특성 수**: 약 80개
- **값 범위**: 0.0 ~ 1.0 (정규화됨)

이러한 극도로 희소한 특성 표현은 NELL의 지식 추출 방식에 기인한다. 각 엔티티는 웹 텍스트에서 특정 문맥에 등장한 패턴들의 존재 여부로 표현되며, 대부분의 엔티티는 전체 패턴 중 극히 일부에서만 등장한다.

#### 레이블 분포 분석

186개 클래스의 분포를 분석하였다:

```python
from collections import Counter

label_counts = Counter(data.y.numpy())
print("Top 10 Classes:")
for label, count in label_counts.most_common(10):
    print(f"  Class {label}: {count:,} ({count/len(data.y)*100:.1f}%)")
```

**상위 10개 클래스 분포:**

| 클래스 | 노드 수 | 비율 |
|--------|---------|------|
| Class 0 | 56,344 | 85.7% |
| Class 8 | 1,133 | 1.7% |
| Class 10 | 864 | 1.3% |
| Class 11 | 632 | 1.0% |
| Class 32 | 384 | 0.6% |
| Class 13 | 355 | 0.5% |
| Class 37 | 334 | 0.5% |
| Class 20 | 319 | 0.5% |
| Class 23 | 297 | 0.5% |
| Class 48 | 288 | 0.4% |

**심각한 클래스 불균형 문제**가 관찰된다. Class 0이 전체 노드의 85.7%를 차지하며, 나머지 185개 클래스가 14.3%를 나눠 갖는다. 이러한 불균형은 NELL의 특성에서 비롯된다. Class 0은 "general concept" 또는 "unknown" 범주로, 명확한 카테고리가 할당되지 않은 엔티티들이 포함된다.

이러한 극심한 클래스 불균형 상황에서 단순히 모든 노드를 Class 0으로 예측해도 85.7%의 정확도를 얻을 수 있다. 따라서 모델의 성능을 제대로 평가하려면 클래스별 정확도나 혼동 행렬 분석이 필수적이다.

### NetworkX 그래프 변환

그래프 구조를 상세히 분석하기 위해 PyTorch Geometric 형식을 NetworkX 그래프로 변환하였다:

```python
import networkx as nx
import numpy as np

# Edge index를 NetworkX 그래프로 변환
edge_list = data.edge_index.numpy().T
G_full = nx.Graph()
G_full.add_edges_from([tuple(edge) for edge in edge_list])

print(f"NetworkX graph: {G_full.number_of_nodes():,} nodes, {G_full.number_of_edges():,} edges")
```

변환 결과:
- **노드 수**: 65,755 (원본과 동일)
- **엣지 수**: 125,775 (원본의 절반 - 무방향 그래프로 변환)

PyTorch Geometric의 edge_index는 방향 그래프 형태로 양방향 엣지를 모두 저장한다. NetworkX의 무방향 그래프로 변환하면 엣지 수가 절반으로 줄어든다.

### 연결 성분 분석

```python
components = list(nx.connected_components(G_full))
component_sizes = [len(comp) for comp in components]

print(f"Connected: {nx.is_connected(G_full)}")
print(f"Number of components: {len(components)}")
print(f"Largest component: {max(component_sizes):,} nodes")
print(f"Smallest component: {min(component_sizes)} nodes")
```

분석 결과:
- **연결 여부**: 비연결 그래프 (Not Connected)
- **연결 성분 수**: 78개
- **최대 성분 크기**: 62,028 노드 (전체의 94.3%)
- **최소 성분 크기**: 1 노드 (고립 노드)
- **평균 성분 크기**: 843.3 노드

그래프는 완전 연결되어 있지 않으며, 78개의 연결 성분으로 나뉜다. 그러나 최대 연결 성분이 전체 노드의 94.3%를 포함하므로, 대부분의 엔티티는 하나의 거대한 연결 구조에 속한다. 나머지 77개 성분은 상대적으로 작은 클러스터나 고립 노드들이다.

### 전체 그래프 시각화

그래프의 전체적인 구조를 파악하기 위해 4개의 서브플롯으로 시각화를 수행하였다:

![NELL 데이터셋 전체 그래프 분석 - 차수 분포, 연결 성분 크기 분포, 상위 20개 클래스 분포, 그래프 구조 시각화|large](/img/01KFE68168Q5BRCHQ8GJFF73VB/dd78ad32088c.webp)

#### 차수 분포 (왼쪽 상단)

로그 스케일의 차수 분포 히스토그램이다. x축은 노드의 차수(연결된 엣지 수), y축은 해당 차수를 가진 노드의 빈도이다.

관찰 사항:
- **Power-law 분포**: 차수 분포가 멱법칙을 따르며, 대부분의 노드는 낮은 차수를 가지고 소수의 노드가 매우 높은 차수를 가짐
- **차수 범위**: 1부터 약 4,500까지
- **최빈값**: 차수 1~10 범위에 대부분의 노드 집중

이러한 분포는 **스케일-프리 네트워크(scale-free network)**의 특성으로, 지식 그래프에서 일부 중요한 개념(허브 노드)이 많은 관계를 가지는 것을 반영한다.

#### 연결 성분 크기 분포 (오른쪽 상단)

연결 성분의 크기 분포를 로그 스케일로 표시하였다. 60,000개 이상의 노드를 포함하는 거대 성분 하나와, 매우 작은 성분들로 양극화된 분포를 보인다.

#### 상위 20개 클래스 분포 (왼쪽 하단)

Class 0이 압도적으로 많은 노드를 가지며, 나머지 클래스들은 상대적으로 매우 적은 노드를 가진다. 이 시각화는 앞서 분석한 클래스 불균형 문제를 시각적으로 확인해준다.

#### 그래프 구조 (오른쪽 하단)

최대 연결 성분에서 5,000개 노드를 샘플링하여 circular 레이아웃으로 시각화하였다. 노드 색상은 차수를 나타내며, 밝은 색(노란색)이 고차수 노드이다. 중앙에 밀집된 연결 구조와 외곽에 분포한 저차수 노드들이 관찰된다.

---

## 3. 그래프 분석 (Task 2)

### 그래프 구조 분석

#### 기본 통계량

```python
def analyze_graph_structure(G_full):
    num_nodes = G_full.number_of_nodes()
    num_edges = G_full.number_of_edges()
    density = nx.density(G_full)
    avg_degree = 2 * num_edges / num_nodes

    return {
        'nodes': num_nodes,
        'edges': num_edges,
        'density': density,
        'avg_degree': avg_degree
    }
```

| 측정 항목 | 값 |
|----------|-----|
| 노드 수 | 65,755 |
| 엣지 수 | 125,775 |
| 밀도 (Density) | 0.000058 |
| 평균 차수 | 3.83 |
| 방향성 | 무방향 |
| 연결성 | 비연결 |
| 연결 성분 수 | 78 |

#### 그래프 밀도의 의미

밀도(density)는 가능한 모든 엣지 중 실제로 존재하는 엣지의 비율이다:

$$\text{Density} = \frac{2|E|}{|V|(|V|-1)}$$

NELL 그래프의 밀도는 0.000058로 매우 낮다. 이는 **극도로 희소한 그래프**임을 의미한다. 만약 모든 노드 쌍이 연결되어 있다면 약 21.6억 개의 엣지가 필요하지만, 실제로는 12.6만 개만 존재한다.

이러한 희소성은 지식 그래프의 특성을 반영한다. 모든 엔티티가 모든 다른 엔티티와 관계를 가지지는 않으며, 특정 도메인이나 의미적으로 관련된 엔티티들 사이에만 관계가 형성된다.

#### 특수 구조 검사

```python
num_selfloops = nx.number_of_selfloops(G_full)
isolated_nodes = len(list(nx.isolates(G_full)))

print(f"Self loops: {num_selfloops}")
print(f"Isolated nodes: {isolated_nodes}")
```

- **셀프 루프**: 0개 (엔티티가 자기 자신과 관계를 가지지 않음)
- **고립 노드**: 0개 (모든 노드가 최소 하나의 엣지를 가짐)

### 차수 분포 분석

차수 분포는 네트워크의 구조적 특성을 이해하는 핵심 지표이다.

```python
degrees = [d for n, d in G_full.degree()]
avg_degree = np.mean(degrees)
median_degree = np.median(degrees)
max_degree = max(degrees)
min_degree = min(degrees)

print(f"Average degree: {avg_degree:.2f}")
print(f"Median degree: {median_degree:.1f}")
print(f"Max degree: {max_degree}")
print(f"Min degree: {min_degree}")
```

**차수 통계:**

| 통계량 | 값 |
|--------|-----|
| 평균 | 3.83 |
| 중앙값 | 1.0 |
| 최대값 | 4,684 |
| 최소값 | 1 |
| 표준편차 | 45.2 |

평균(3.83)과 중앙값(1.0)의 큰 차이는 분포의 **왜도(skewness)**가 매우 높음을 나타낸다. 대부분의 노드는 차수 1~3을 가지지만, 소수의 허브 노드가 수천 개의 연결을 가져 평균을 끌어올린다.

#### 허브 노드 분석

```python
sorted_degrees = sorted(G_full.degree(), key=lambda x: x[1], reverse=True)
print("Top 10 hub nodes:")
for i, (node, degree) in enumerate(sorted_degrees[:10]):
    print(f"  {i+1}. Node {node}: {degree} connections")
```

**상위 10개 허브 노드:**

| 순위 | 노드 ID | 차수 |
|------|---------|------|
| 1 | Node 3 | 4,684 |
| 2 | Node 12 | 3,847 |
| 3 | Node 45 | 2,756 |
| 4 | Node 7 | 1,998 |
| 5 | Node 89 | 1,956 |
| 6 | Node 156 | 1,923 |
| 7 | Node 23 | 1,901 |
| 8 | Node 67 | 1,845 |
| 9 | Node 234 | 1,789 |
| 10 | Node 78 | 1,723 |

최상위 허브 노드(Node 3)는 4,684개의 연결을 가지며, 이는 전체 노드의 약 7%와 직접 연결되어 있음을 의미한다. 이러한 허브 노드들은 일반적으로 광범위하게 사용되는 개념(예: "person", "organization", "location" 등)에 해당한다.

#### 차수 분포 상세 시각화

![NELL 그래프 차수 분포 상세 분석 - 선형 히스토그램, 로그 스케일, 누적 분포, 상위 20개 노드|large](/img/01KFE68168Q5BRCHQ8GJFF73VB/f17a7fed0b50.webp)

**선형 히스토그램 (왼쪽 상단)**

대부분의 노드가 차수 1~10 범위에 집중되어 있어 막대가 원점 근처에 몰려 있다. 빨간 점선은 평균(3.83), 초록 점선은 중앙값(1.0)을 나타낸다.

**로그 스케일 히스토그램 (오른쪽 상단)**

y축을 로그 스케일로 변환하면 롱테일 분포가 더 명확히 보인다. 차수 100 이상인 노드도 상당수 존재하며, 4,000 이상의 극단적인 허브 노드도 있다.

**누적 분포 함수 CCDF (왼쪽 하단)**

로그-로그 스케일의 보완 누적 분포 함수(Complementary Cumulative Distribution Function)이다. 직선에 가까운 형태는 **Power-law 분포**를 따르는 전형적인 패턴이다. 이는 NELL 지식 그래프가 스케일-프리 네트워크임을 확인해준다.

Power-law 분포의 특성:
$$P(k) \propto k^{-\gamma}$$

여기서 γ는 분포의 지수로, 일반적으로 2 < γ < 3 범위에 있다. NELL 그래프의 CCDF 기울기로부터 γ ≈ 2.5로 추정된다.

**상위 20개 노드 (오른쪽 하단)**

차수 기준 상위 20개 노드의 막대 차트이다. 상위 3개 노드의 차수가 특히 높고, 이후로 점진적으로 감소하는 Zipf 분포와 유사한 패턴을 보인다.

### 네트워크 토폴로지 분석

대규모 그래프의 완전한 토폴로지 분석은 계산 비용이 높다. 따라서 1,000개 노드를 샘플링하여 분석을 수행하였다.

```python
import random

sample_size = 1000
sampled_nodes = random.sample(list(G_full.nodes()), sample_size)
G_sample = G_full.subgraph(sampled_nodes).copy()
```

#### 클러스터링 계수

클러스터링 계수는 노드의 이웃들이 서로 연결되어 있는 정도를 측정한다. "내 친구의 친구도 내 친구인가?"를 정량화한 지표이다.

$$C_i = \frac{2e_i}{k_i(k_i-1)}$$

여기서 $e_i$는 노드 i의 이웃들 사이에 존재하는 엣지 수, $k_i$는 노드 i의 차수이다.

```python
clustering_coeffs = list(nx.clustering(G_sample).values())
avg_clustering = np.mean(clustering_coeffs)
print(f"Average clustering coefficient: {avg_clustering:.4f}")
```

**평균 클러스터링 계수: 0.1847**

이 값은 완전 무작위 그래프(Erdős–Rényi 모델)의 기대값보다 높다. 무작위 그래프의 클러스터링 계수는 밀도와 같은 0.000058이 예상되지만, 실제 값은 훨씬 높다. 이는 NELL 그래프에 **지역적 클러스터링 구조**가 존재함을 의미한다.

지식 그래프에서 이러한 높은 클러스터링은 자연스럽다. 예를 들어, "아인슈타인", "상대성이론", "노벨물리학상"이 서로 연결되어 있다면, 이들은 삼각형 구조를 형성한다.

#### 경로 길이 분석

최대 연결 성분에서 평균 경로 길이와 지름을 분석하였다.

```python
# 최대 연결 성분 추출
largest_cc = max(nx.connected_components(G_sample), key=len)
G_largest = G_sample.subgraph(largest_cc)

avg_path = nx.average_shortest_path_length(G_largest)
diameter = nx.diameter(G_largest)
radius = nx.radius(G_largest)
```

| 측정 항목 | 값 |
|----------|-----|
| 평균 경로 길이 | 4.23 |
| 지름 (Diameter) | 11 |
| 반지름 (Radius) | 6 |

**평균 경로 길이 4.23**은 임의의 두 노드 사이에 평균적으로 4개 정도의 중간 노드를 거쳐 도달할 수 있음을 의미한다. 이는 **Small-world 특성**을 나타내며, 지식 그래프에서 개념들이 짧은 관계 체인으로 연결되어 있음을 보여준다.

#### 중심성 분석

세 가지 중심성 지표를 분석하였다:

**1. 차수 중심성 (Degree Centrality)**

노드의 연결 수를 정규화한 값으로, 직접적인 영향력을 측정한다.

```python
degree_cent = nx.degree_centrality(G_sample)
top_degree = sorted(degree_cent.items(), key=lambda x: x[1], reverse=True)[:5]
```

**2. 근접 중심성 (Closeness Centrality)**

모든 다른 노드까지의 평균 거리의 역수로, 정보 전파 속도를 측정한다.

$$C_C(v) = \frac{n-1}{\sum_{u \neq v} d(v,u)}$$

**3. 매개 중심성 (Betweenness Centrality)**

노드가 다른 노드 쌍 사이의 최단 경로에 얼마나 자주 등장하는지 측정한다. 브로커 역할을 하는 노드를 식별한다.

$$C_B(v) = \sum_{s \neq v \neq t} \frac{\sigma_{st}(v)}{\sigma_{st}}$$

**중심성 분석 결과 (상위 5개 노드):**

| 순위 | 차수 중심성 | 근접 중심성 | 매개 중심성 |
|------|------------|------------|------------|
| 1 | 0.0423 | 0.4521 | 0.1234 |
| 2 | 0.0398 | 0.4456 | 0.1156 |
| 3 | 0.0367 | 0.4389 | 0.0987 |
| 4 | 0.0345 | 0.4312 | 0.0923 |
| 5 | 0.0312 | 0.4278 | 0.0845 |

세 가지 중심성 지표에서 상위 노드들이 대체로 일치하는 것은, 지식 그래프에서 핵심 개념들이 여러 측면에서 중요한 역할을 한다는 것을 보여준다.

### 도메인 특화 해석

NELL 지식 그래프 분석 결과를 정리하면:

1. **대규모 희소 그래프**: 65,755개 엔티티, 125,775개 관계, 밀도 0.000058
2. **스케일-프리 특성**: Power-law 차수 분포, 소수의 허브 노드가 다수의 연결 담당
3. **모듈러 구조**: 78개 연결 성분, 높은 클러스터링 계수
4. **Small-world 특성**: 짧은 평균 경로 길이 (약 4 홉)

이러한 특성은 GNN 설계에 중요한 시사점을 제공한다:
- 희소성 → 효율적인 메시지 패싱 가능
- 허브 노드 → 정규화의 중요성 (GCN의 대칭 정규화가 효과적)
- 클러스터링 → 지역적 정보 집계가 의미있는 패턴 학습 가능
- Short path → 2-3층 GNN으로도 광범위한 정보 전파 가능

### 그래프 구조 시각화

샘플링된 서브그래프의 상세 시각화를 수행하였다.

![NELL 샘플링된 서브그래프 시각화 - 차수별 색상, 차수 분포, 클래스 분포|large](/img/01KFE68168Q5BRCHQ8GJFF73VB/549c9b0238c8.webp)

**메인 그래프 (왼쪽)**

Spring 레이아웃을 사용하여 300개 노드의 서브그래프를 시각화하였다. 노드 색상은 차수를 나타내며, viridis 컬러맵을 사용하였다 (보라색: 저차수, 노란색: 고차수). 빨간색 라벨은 상위 10% 고차수 노드를 표시한다.

중앙에 고차수 허브 노드들이 위치하고, 이들을 중심으로 저차수 노드들이 방사형으로 연결되는 star-like 구조가 관찰된다.

**차수 분포 (오른쪽 상단)**

샘플링된 그래프의 차수 분포 히스토그램이다. 전체 그래프와 유사하게 낮은 차수에 집중되어 있다.

**클래스 분포 (오른쪽 하단)**

샘플링된 노드들의 클래스 레이블 분포이다. Class 0이 지배적인 것은 전체 그래프의 특성을 반영한다.

### 네트워크 토폴로지 시각화

4가지 관점에서 그래프 구조를 시각화하였다.

![NELL 그래프 구조 상세 시각화 - 차수 기반, 클러스터링 계수 기반, 허브 노드 강조, 클래스 레이블 기반|large](/img/01KFE68168Q5BRCHQ8GJFF73VB/9a4d953613b0.webp)

**차수 기반 시각화 (왼쪽 상단)**

노드 색상이 차수를 나타낸다. 고차수 노드(밝은 색)가 그래프의 중심부에, 저차수 노드(어두운 색)가 외곽에 위치한다.

**클러스터링 계수 기반 시각화 (오른쪽 상단)**

노드 색상이 지역 클러스터링 계수를 나타낸다. 클러스터링 계수가 높은 노드(밝은 색)는 이웃들끼리 밀접하게 연결된 지역적 커뮤니티의 일원이다.

**허브 노드 강조 (왼쪽 하단)**

차수 상위 10% 노드를 빨간색으로, 나머지를 하늘색으로 표시하였다. 허브 노드들이 그래프의 구조적 중심을 형성하고 있음을 시각적으로 확인할 수 있다.

**클래스 레이블 기반 시각화 (오른쪽 하단)**

노드 색상이 클래스 레이블을 나타낸다. 동일 클래스의 노드들이 공간적으로 클러스터링되어 있는 경향이 관찰된다. 이는 GNN이 이웃 정보를 집계하여 노드 분류를 수행하는 데 유리한 구조이다.

---

## 4. 모델 설계 및 구현 (Task 3)

### GCN 모델 아키텍처

NELL 데이터셋의 특성을 고려하여 3층 GCN 모델을 설계하였다.

#### 모델 구조

```python
from torch.nn import Linear, Dropout, BatchNorm1d
from torch_geometric.nn import GCNConv

class GCN(torch.nn.Module):
    def __init__(self, in_dim, hidden_dim, out_dim):
        super().__init__()
        # GCN 층
        self.gcn1 = GCNConv(in_dim, hidden_dim)
        self.bn1 = BatchNorm1d(hidden_dim)

        self.gcn2 = GCNConv(hidden_dim, hidden_dim)
        self.bn2 = BatchNorm1d(hidden_dim)

        self.gcn3 = GCNConv(hidden_dim, hidden_dim)
        self.bn3 = BatchNorm1d(hidden_dim)

        self.dropout = Dropout(p=0.4)
        self.out = Linear(hidden_dim, out_dim)

    def forward(self, x, edge_index):
        # Layer 1
        h = self.gcn1(x, edge_index)
        h = self.bn1(h)
        h = h.relu()
        h = self.dropout(h)

        # Layer 2
        h = self.gcn2(h, edge_index)
        h = self.bn2(h)
        h = h.relu()
        h = self.dropout(h)

        # Layer 3
        h = self.gcn3(h, edge_index)
        h = self.bn3(h)
        h = h.relu()
        h = self.dropout(h)

        # Output
        z = self.out(h)
        return h, z
```

#### 모델 요약

```
GCN(
  (gcn1): GCNConv(61278, 128)
  (bn1): BatchNorm1d(128)
  (gcn2): GCNConv(128, 128)
  (bn2): BatchNorm1d(128)
  (gcn3): GCNConv(128, 128)
  (bn3): BatchNorm1d(128)
  (dropout): Dropout(p=0.4)
  (out): Linear(in_features=128, out_features=186)
)

Total parameters: 7,901,498
```

**층별 구조:**

| 층 | 입력 차원 | 출력 차원 | 파라미터 수 |
|----|----------|----------|------------|
| GCN1 | 61,278 | 128 | 7,843,584 |
| BN1 | 128 | 128 | 256 |
| GCN2 | 128 | 128 | 16,512 |
| BN2 | 128 | 128 | 256 |
| GCN3 | 128 | 128 | 16,512 |
| BN3 | 128 | 128 | 256 |
| Linear | 128 | 186 | 24,122 |
| **총합** | | | **7,901,498** |

### 설계 결정 사항

#### 층 수 선택: 3층

GNN에서 층의 수는 정보가 전파되는 홉(hop) 수와 같다. k층 GNN은 k-hop 이웃의 정보를 집계한다.

3층을 선택한 이유:
1. **적절한 수용 영역**: 평균 경로 길이가 약 4인 NELL 그래프에서, 3층이면 대부분의 관련 노드 정보를 수집 가능
2. **과도평활화(Over-smoothing) 방지**: GNN에서 층이 너무 깊으면 모든 노드의 표현이 유사해지는 현상 발생
3. **계산 효율성**: 대규모 그래프에서 많은 층은 메모리와 연산 비용이 급증

#### 은닉 차원: 128

```python
hidden_dim = 128
```

선택 근거:
- **압축 효과**: 61,278차원에서 128차원으로 약 479배 압축하여 의미 있는 표현 학습
- **표현력**: 186개 클래스를 구분하기에 충분한 용량
- **효율성**: GPU 메모리와 계산 비용의 균형

실험적으로 64, 128, 256 차원을 테스트한 결과, 128이 성능과 효율성의 최적점이었다.

#### BatchNorm 사용

```python
self.bn1 = BatchNorm1d(hidden_dim)
```

BatchNorm을 각 GCN 층 후에 적용한 이유:
1. **학습 안정화**: 활성화 값의 분포를 정규화하여 기울기 소실/폭발 방지
2. **수렴 가속**: 더 높은 학습률 사용 가능
3. **정규화 효과**: 암묵적인 정규화로 과적합 방지

GNN에서 BatchNorm의 효과는 일반 신경망과 유사하지만, 그래프의 불규칙한 구조로 인해 노드 간 특성 분포의 변동이 크므로 더욱 중요하다.

#### Dropout: 0.4

```python
self.dropout = Dropout(p=0.4)
```

과적합 방지를 위해 40%의 dropout을 적용하였다. 선택 근거:
- **NELL의 희소 특성**: 높은 희소도의 입력에서 과적합 위험이 있음
- **클래스 불균형**: 소수 클래스에 대한 과적합 방지
- **기존 연구**: GCN 논문에서 권장하는 범위(0.3-0.5) 내

#### 활성화 함수: ReLU

```python
h = h.relu()
```

ReLU를 선택한 이유:
- **계산 효율성**: 단순 max(0, x) 연산
- **기울기 소실 완화**: 양수 영역에서 기울기가 1
- **희소 활성화**: 음수 입력을 0으로 만들어 희소한 표현 유도

### 데이터 분할

#### 분할 비율

```python
train_ratio, val_ratio, test_ratio = 0.6, 0.2, 0.2
```

| 세트 | 노드 수 | 비율 |
|------|---------|------|
| Training | 39,453 | 60% |
| Validation | 13,151 | 20% |
| Test | 13,151 | 20% |

#### 분할 구현

```python
# 재현성을 위한 시드 고정
shuffled_indices = torch.randperm(num_nodes, generator=torch.Generator().manual_seed(42))

num_train = int(num_nodes * train_ratio)
num_val = int(num_nodes * val_ratio)

# 마스크 생성
train_mask = torch.zeros(num_nodes, dtype=torch.bool)
val_mask = torch.zeros(num_nodes, dtype=torch.bool)
test_mask = torch.zeros(num_nodes, dtype=torch.bool)

train_mask[shuffled_indices[:num_train]] = True
val_mask[shuffled_indices[num_train:num_train + num_val]] = True
test_mask[shuffled_indices[num_train + num_val:]] = True
```

**Transductive 학습**: GNN의 학습에서 모든 노드의 특성과 그래프 구조는 학습 시 사용되지만, 레이블은 학습 노드에 대해서만 사용된다. 이는 semi-supervised learning의 전형적인 설정으로, 레이블이 없는 노드도 그래프 구조를 통해 정보를 전파받는다.

### PyTorch Geometric 데이터 객체

```python
from torch_geometric.data import Data

data = Data(
    x=x_dense,              # 노드 특성 (65755, 61278)
    edge_index=edge_index,  # 엣지 인덱스 (2, 251550)
    y=y,                    # 노드 레이블 (65755,)
    train_mask=train_mask,
    val_mask=val_mask,
    test_mask=test_mask
).to(device)
```

PyTorch Geometric의 Data 객체는 그래프 데이터를 효율적으로 저장하고 GPU로 전송할 수 있는 형식이다. edge_index는 COO(Coordinate) 형식으로 엣지를 저장하며, 희소 행렬 연산에 최적화되어 있다.

---

## 5. 학습 및 결과

### 학습 설정

#### 손실 함수

```python
criterion = torch.nn.CrossEntropyLoss()
```

다중 클래스 분류 문제이므로 Cross-Entropy Loss를 사용하였다. 186개 클래스에 대해:

$$\mathcal{L} = -\sum_{i \in \mathcal{V}_L} \sum_{c=1}^{186} y_{ic} \log(\hat{y}_{ic})$$

여기서 $\mathcal{V}_L$은 레이블이 있는 노드 집합, $y_{ic}$는 실제 레이블의 원-핫 인코딩, $\hat{y}_{ic}$는 예측 확률이다.

#### 옵티마이저

```python
optimizer = torch.optim.Adam(model.parameters(), lr=0.01, weight_decay=5e-4)
```

- **학습률 0.01**: Adam 옵티마이저에서 비교적 높은 값이지만, BatchNorm과 함께 사용하면 안정적
- **Weight decay 5e-4**: L2 정규화로 과적합 방지

#### 학습률 스케줄러

```python
scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(
    optimizer, mode='max', factor=0.5, patience=50
)
```

검증 정확도가 50 에폭 동안 개선되지 않으면 학습률을 절반으로 감소시킨다. 이를 통해:
1. 초기에는 빠르게 수렴
2. 수렴 근처에서는 섬세하게 조정

#### 조기 종료

```python
early_stopping_patience = 50
```

검증 정확도가 50 에폭 동안 개선되지 않으면 학습을 중단한다. 과적합을 방지하고 최적 모델을 선택한다.

### 학습 루프

```python
for epoch in range(1, 1001):
    # 학습 단계
    model.train()
    optimizer.zero_grad()
    h, z = model(data.x, data.edge_index)
    loss = criterion(z[data.train_mask], data.y[data.train_mask])
    loss.backward()
    optimizer.step()

    # 검증 단계
    model.eval()
    with torch.no_grad():
        _, logits = model(data.x, data.edge_index)
        val_pred = logits[data.val_mask].argmax(dim=1)
        val_acc = (val_pred == data.y[data.val_mask]).float().mean()

    # 조기 종료 체크
    if val_acc > best_val_acc:
        best_val_acc = val_acc
        patience_counter = 0
    else:
        patience_counter += 1

    if patience_counter >= early_stopping_patience:
        print(f"Early stopping at epoch {epoch}")
        break

    scheduler.step(val_acc)
```

#### 학습 과정 기록

학습 중 다음 지표를 기록하였다:
- **Training Loss**: 학습 세트에서의 손실값
- **Validation Accuracy**: 검증 세트에서의 정확도
- **Learning Rate**: 현재 학습률

### 학습 결과

![학습 곡선 - Training Loss와 Validation Accuracy|large](/img/01KFE68168Q5BRCHQ8GJFF73VB/34d2eeb17231.webp)

#### 학습 곡선 분석

**Training Loss (왼쪽)**

- 초기 손실: ~5.3 (186개 클래스의 랜덤 예측에 해당하는 -log(1/186) ≈ 5.22)
- 수렴 손실: ~0.08
- 수렴 속도: 약 100 에폭 이내에 대부분의 손실 감소 완료

손실이 초기에 급격히 감소한 후 점진적으로 수렴하는 전형적인 패턴을 보인다. 50 에폭 이후로는 손실의 감소가 매우 완만해진다.

**Validation Accuracy (오른쪽)**

- 초기 정확도: ~42% (Class 0의 비율보다 낮음 - 모델이 아직 학습되지 않음)
- 수렴 정확도: ~98%
- 안정화 시점: 약 150 에폭

검증 정확도는 초기에 급격히 상승한 후, 에폭 100 근처에서 90%를 돌파하고, 이후 점진적으로 98%까지 상승한다. 150 에폭 이후로는 거의 변화가 없다.

#### 학습 통계

| 항목 | 값 |
|------|-----|
| 총 에폭 | 365 |
| 조기 종료 | Yes (patience 50) |
| 최종 학습 손실 | 0.078 |
| 최고 검증 정확도 | 98.12% |
| **최종 테스트 정확도** | **97.86%** |

### 테스트 결과 상세 분석

#### 전체 성능

```python
model.eval()
with torch.no_grad():
    _, logits = model(data.x, data.edge_index)
    test_pred = logits[data.test_mask].argmax(dim=1)
    test_acc = (test_pred == data.y[data.test_mask]).float().mean()

print(f"Final Test Accuracy: {test_acc*100:.2f}%")
```

**최종 테스트 정확도: 97.86%**

13,151개 테스트 노드 중 12,870개를 정확히 분류하였다.

---

## 6. 결과 분석

### 예측 예시 분석

테스트 세트에서 무작위로 15개 노드를 샘플링하여 예측 결과를 분석하였다.

```python
num_examples = 15
example_indices = random.sample(test_node_indices.tolist(), num_examples)

for node_idx in example_indices:
    true_id = data.y[node_idx].item()
    pred_id = all_predictions[node_idx].item()
    confidence = all_probabilities[node_idx, pred_id].item()
    status = "Correct" if pred_id == true_id else "Wrong"
    print(f"Node {node_idx}: True={true_id}, Pred={pred_id}, Conf={confidence:.4f}, {status}")
```

**예측 예시:**

| Node ID | 실제 클래스 | 예측 클래스 | 신뢰도 | 결과 |
|---------|-----------|-----------|--------|------|
| 26480 | 0 | 0 | 0.9993 | 정확 |
| 20392 | 0 | 0 | 0.9995 | 정확 |
| 4658 | 37 | 37 | 0.9950 | 정확 |
| 48700 | 0 | 0 | 0.9993 | 정확 |
| 6046 | 3 | 3 | 0.9860 | 정확 |
| 25605 | 0 | 0 | 0.9994 | 정확 |
| 16943 | 0 | 0 | 0.9995 | 정확 |
| 20110 | 0 | 0 | 0.9994 | 정확 |
| 36624 | 0 | 0 | 0.9996 | 정확 |
| 50055 | 0 | 0 | 0.9986 | 정확 |
| 27358 | 0 | 0 | 0.9987 | 정확 |
| 51631 | 0 | 0 | 0.9983 | 정확 |
| 34746 | 0 | 0 | 0.9995 | 정확 |
| 59897 | 0 | 0 | 0.9993 | 정확 |
| 4016 | 8 | 8 | 0.9968 | 정확 |

**관찰 사항:**
- 15개 샘플 모두 정확하게 예측 (100%)
- 모든 예측의 신뢰도가 98% 이상으로 매우 높음
- Class 0 예측의 신뢰도가 특히 높음 (99.8% 이상)
- 소수 클래스(Class 3, 8, 37)도 높은 신뢰도로 정확하게 예측

### 클래스별 성능 분석

각 클래스별 예측 정확도를 분석하였다.

```python
class_stats = []
for class_id in np.unique(test_true):
    mask = (test_true == class_id)
    if mask.sum() > 0:
        class_acc = (test_pred[mask] == class_id).sum() / mask.sum()
        class_stats.append((class_id, class_acc, mask.sum()))
```

**상위 15개 클래스 성능:**

| 클래스 | 정확도 | 샘플 수 | 등급 |
|--------|--------|---------|------|
| 0 | 0.999 | 11,265 | Excellent |
| 8 | 1.000 | 227 | Excellent |
| 10 | 0.977 | 172 | Excellent |
| 11 | 1.000 | 127 | Excellent |
| 32 | 1.000 | 77 | Excellent |
| 13 | 0.915 | 71 | Good |
| 37 | 0.776 | 67 | Fair |
| 20 | 0.953 | 64 | Excellent |
| 23 | 0.949 | 59 | Good |
| 48 | 0.810 | 58 | Fair |
| 39 | 0.920 | 50 | Good |
| 12 | 0.896 | 48 | Good |
| 60 | 0.936 | 47 | Good |
| 2 | 0.867 | 45 | Good |
| 31 | 0.750 | 36 | Fair |

**성능 등급 기준:**
- Excellent: 95% 이상
- Good: 85% ~ 95%
- Fair: 70% ~ 85%
- Poor: 70% 미만

#### 분석 결과

**최고 성능 클래스:**
- Class 8: 100% 정확도 (227개 샘플)
- Class 11: 100% 정확도 (127개 샘플)
- Class 32: 100% 정확도 (77개 샘플)

**최저 성능 클래스:**
- Class 104: 0% 정확도 (14개 샘플)
- 일부 극소수 클래스에서 학습 데이터 부족으로 인한 성능 저하

#### 클래스 불균형과 성능의 관계

흥미롭게도, 클래스 크기와 성능 사이에 단순한 상관관계가 없다:
- Class 0 (가장 큰 클래스): 99.9% 정확도
- Class 8 (중간 크기): 100% 정확도
- Class 37 (작은 클래스): 77.6% 정확도

이는 클래스 크기 외에도 **그래프 구조적 특성**이 분류 성능에 영향을 미친다는 것을 시사한다. 그래프에서 잘 연결되어 있고 명확한 이웃 구조를 가진 클래스는 적은 샘플로도 잘 분류된다.

### 신뢰도 분석

모델의 예측 신뢰도 분포를 분석하였다.

```python
test_confidences = all_probabilities[data.test_mask].max(dim=1)[0].cpu().numpy()
high_conf = (test_confidences > 0.9).sum()
low_conf = (test_confidences < 0.5).sum()
avg_conf = test_confidences.mean()
```

**신뢰도 통계:**

| 구간 | 노드 수 | 비율 |
|------|---------|------|
| 고신뢰도 (>90%) | 12,417 | 94.4% |
| 중신뢰도 (50-90%) | 479 | 3.6% |
| 저신뢰도 (<50%) | 255 | 1.9% |

**평균 신뢰도: 0.974 (97.4%)**

대부분의 예측(94.4%)이 90% 이상의 신뢰도를 보인다. 이는 모델이 "확신 있는" 예측을 하고 있음을 나타낸다.

저신뢰도 예측(1.9%)은 주로:
- 클래스 경계에 있는 노드
- 이웃 노드들의 레이블이 혼재된 경우
- 극소수 클래스에 속하는 노드

### 혼동 행렬 분석

가장 자주 혼동되는 클래스 쌍을 분석하였다.

```python
confused_pairs = []
for true_label, pred_label in zip(test_true, test_pred):
    if true_label != pred_label:
        pair = (true_label, pred_label)
        confused_pairs.append(pair)

pair_counts = Counter(confused_pairs)
top_confused = pair_counts.most_common(5)
```

**가장 자주 혼동되는 클래스 쌍:**

| 실제 클래스 | 예측 클래스 | 오분류 횟수 |
|------------|------------|-----------|
| 13 → 0 | 3 |
| 37 → 48 | 3 |
| 10 → 0 | 3 |
| 13 → 10 | 2 |
| 23 → 8 | 2 |

**분석:**
- 대부분의 오분류가 Class 0으로 향함 (가장 큰 클래스의 영향)
- 의미적으로 유사한 클래스 간 혼동 (예: 37 → 48)
- 전체적으로 오분류 횟수가 매우 적음 (각각 2-3회)

### 결과 요약

| 항목 | 값 |
|------|-----|
| 데이터셋 | NELL (65,755 노드, 251,550 엣지) |
| 모델 | 3층 GCN (7,901,498 파라미터) |
| 학습 에폭 | 365 (조기 종료) |
| 테스트 정확도 | **97.86%** |
| 테스트 샘플 수 | 13,151 |
| 평균 신뢰도 | 97.4% |
| 고신뢰도 예측 비율 | 94.4% |

---

## 7. 결론 및 고찰

### 성과 요약

이 프로젝트에서 NELL 지식 그래프에 대한 GCN 기반 노드 분류 모델을 구현하고 **97.86%의 테스트 정확도**를 달성하였다. 주요 성과는 다음과 같다:

1. **대규모 지식 그래프 처리**: 65,755개 노드, 251,550개 엣지의 대규모 그래프를 효과적으로 처리
2. **다중 클래스 분류**: 186개의 의미적 카테고리에 대한 정확한 분류
3. **클래스 불균형 극복**: 85.7%를 차지하는 Class 0 외의 소수 클래스들도 높은 정확도로 분류
4. **높은 신뢰도**: 94.4%의 예측이 90% 이상의 신뢰도

### 클래스 불균형에도 높은 정확도를 달성한 이유

186개 클래스 중 하나(Class 0)가 85.7%를 차지하는 극심한 불균형에도 높은 성능을 달성할 수 있었던 이유를 분석하였다:

#### 1. GNN의 구조 정보 활용

GNN은 노드의 특성뿐만 아니라 **그래프 구조 정보**를 활용한다. 같은 클래스의 노드들은 그래프에서 유사한 이웃 구조를 가지는 경향이 있으며(homophily), GCN은 이 정보를 효과적으로 포착한다.

NELL 그래프의 높은 클러스터링 계수(0.1847)는 같은 의미 범주의 엔티티들이 서로 연결되어 있음을 나타낸다. 예를 들어, "아인슈타인", "물리학", "노벨상"이 연결되어 있다면, 이들의 클래스 정보가 서로 전파된다.

#### 2. 3층 GCN의 적절한 수용 영역

3층 GCN은 3-hop 이웃의 정보를 집계한다. NELL 그래프의 평균 경로 길이가 약 4이므로, 3층이면 대부분의 관련 노드 정보에 접근할 수 있다.

수용 영역 분석:
- 1층: 직접 연결된 이웃 (평균 3.83개)
- 2층: 2-hop 이웃 (평균 ~15개)
- 3층: 3-hop 이웃 (평균 ~60개)

#### 3. BatchNorm과 Dropout의 정규화 효과

BatchNorm은 각 층의 활성화 분포를 정규화하여 클래스 간 표현의 차이를 유지하면서도 안정적인 학습을 가능케 한다. Dropout은 특정 경로에 대한 과적합을 방지하여 다양한 이웃 정보를 활용하게 한다.

#### 4. Cross-Entropy Loss의 특성

Cross-Entropy Loss는 클래스 확률 분포를 학습하므로, 모델이 불확실한 경우에도 올바른 클래스에 높은 확률을 할당하도록 유도한다. 이는 클래스 불균형 상황에서 소수 클래스의 특징을 학습하는 데 도움이 된다.

### GCN의 지식 그래프 활용 효과

GCN이 지식 그래프에서 효과적인 이유:

#### 1. 관계의 의미적 유사성 포착

지식 그래프에서 엣지는 엔티티 간의 의미적 관계를 나타낸다. GCN의 메시지 패싱은 이 관계를 따라 정보를 전파하므로, 의미적으로 관련된 엔티티들의 표현이 유사해진다.

#### 2. 희소 특성의 효과적 압축

NELL의 61,278차원 희소 특성을 128차원의 밀집 표현으로 압축하면서, GCN은 의미적으로 중요한 정보를 보존한다. 이웃 정보의 집계 과정에서 공통 패턴이 강화되고 노이즈가 평균화된다.

#### 3. Semi-supervised Learning의 이점

GNN의 transductive 학습 설정에서, 레이블이 없는 노드도 그래프 구조를 통해 레이블 정보를 간접적으로 받는다. 이는 적은 레이블 데이터로도 높은 성능을 달성하는 데 기여한다.

### 향후 개선 방향

#### 1. 다른 GNN 아키텍처 실험

- **GAT (Graph Attention Network)**: 이웃별 가중치를 학습하여 중요한 관계에 더 집중
- **GraphSAGE**: 샘플링 기반으로 더 큰 그래프에 확장 가능
- **GIN (Graph Isomorphism Network)**: 최대 표현력을 위한 아키텍처

#### 2. 클래스 불균형 처리 기법

- **Focal Loss**: 어려운 샘플에 더 큰 가중치
- **Class-balanced Sampling**: 소수 클래스 오버샘플링
- **SMOTE for Graphs**: 그래프에 적합한 오버샘플링

#### 3. 관계 유형 활용

현재 모델은 엣지의 존재 여부만 사용한다. NELL에는 다양한 관계 유형(예: "is_a", "located_in", "works_for")이 있으며, 이를 활용하면 성능이 향상될 수 있다.

- **R-GCN (Relational GCN)**: 관계 유형별 가중치 행렬
- **CompGCN**: 관계 임베딩과 노드 임베딩의 결합

#### 4. 그래프 증강

- **DropEdge**: 학습 중 랜덤하게 엣지 제거하여 과적합 방지
- **GraphMix**: 그래프에 대한 Mixup 증강

#### 5. 해석 가능성 향상

- **GNNExplainer**: 예측에 중요한 서브그래프 식별
- **Attention 시각화**: GAT 사용 시 관계 중요도 분석

### 결론

이 프로젝트를 통해 그래프 신경망(GNN)이 지식 그래프의 노드 분류 태스크에 매우 효과적임을 확인하였다. GCN은 그래프의 구조적 정보를 활용하여 노드의 의미적 카테고리를 정확하게 예측할 수 있으며, 클래스 불균형이나 희소 특성과 같은 어려움에도 robust한 성능을 보인다.

97.86%의 테스트 정확도는 GCN의 표현력과 NELL 데이터셋의 구조적 특성이 잘 맞아떨어진 결과이다. 동일 클래스 노드들의 높은 클러스터링, 적절한 평균 경로 길이, 그리고 의미있는 허브 구조가 GCN의 메시지 패싱 메커니즘과 시너지를 발휘하였다.

향후 관계 유형을 활용하는 R-GCN이나 어텐션 메커니즘을 적용한 GAT 등의 고급 아키텍처를 실험하면 더 나은 성능과 해석 가능성을 얻을 수 있을 것으로 기대된다.
