---
id: "01M6EJDSNBMM4DKNDGF4PX1EYM"
slug: "neural-network-backpropagation"
title: "신경망과 역전파: NumPy로 직접 구현하고 하드웨어로 들여다보기"
category: "01M6EJDSNB9ZFWAS5MYXPCQAST"
tags: ["neural network", "backpropagation", "numpy", "mnist"]
summary: "AI Note 1편. 신경망과 역전파를 식으로 유도하고 NumPy로 784-128-10 신경망을 직접 짜서 MNIST를 97.7%까지 학습했다. 같은 코드가 배치 크기에 따라 180배 빨라지는 이유를 산술 강도로 살펴본다."
thumbnail: "/img/01M6EJDSNBMM4DKNDGF4PX1EYM/750cb9dacd03.webp"
note: true
order: 1
created: "2026-11-03T01:47:04+09:00"
updated: "2026-11-03T01:47:04+09:00"
---
:::note
**이 글의 핵심 3줄**
- 신경망은 "행렬곱 + 비선형 함수"를 쌓은 함수이고, 학습은 손실을 줄이는 쪽으로 가중치를 조금씩 옮기는 일이다.
- 역전파는 연쇄 법칙(chain rule)을 뒤에서부터 한 번에 적용해, 모든 가중치의 그래디언트를 순전파 한 번 정도의 비용으로 구하는 방법이다.
- NumPy만으로 784-128-10 신경망을 직접 짜서 MNIST를 학습하면, 맥 M1 Max CPU에서 1.6초 만에 테스트 정확도 97.7%가 나온다. 같은 코드도 배치 크기에 따라 샘플당 시간이 180배까지 달라진다.
:::

CNN부터 LLM까지 이어지는 AI Note 시리즈의 첫 글이다. 이 시리즈는 개념 설명에서 끝나지 않고, 매 글마다 **직접 돌린 실습 결과**와 **하드웨어로 보면 무엇이 비싼지**를 함께 본다. 첫 글은 모든 모델의 바닥에 있는 두 가지, 신경망과 역전파다.

## 1. 뉴런 하나: 가중합과 활성화

뉴런 하나는 입력 $x$에 가중치 $w$를 곱해 더하고(가중합), 그 결과를 비선형 함수 $f$에 통과시킨다.

$$y = f\left(\sum_i w_i x_i + b\right)$$

여기서 $f$가 없으면 아무리 층을 쌓아도 전체가 하나의 선형 변환(행렬 하나)으로 합쳐진다. 그래서 층 사이에 비선형 함수를 둔다. 이 글에서는 가장 널리 쓰는 ReLU, $f(z) = \max(0, z)$를 쓴다. 음수는 0으로 막고 양수는 그대로 통과시키는 단순한 함수다.

## 2. 층으로 쌓으면 행렬곱이 된다

뉴런을 여러 개 나란히 두면 가중합은 행렬곱 한 번으로 바뀐다. 28×28 손글씨 숫자(MNIST)를 784칸짜리 벡터로 펴고, 128개 뉴런의 은닉층과 10개 출력(숫자 0~9)을 두면 다음과 같다. $B$는 한 번에 넣는 샘플 수(배치 크기)다.

$$z_1 = XW_1 + b_1, \quad h = \mathrm{ReLU}(z_1), \quad z_2 = hW_2 + b_2$$

| 값 | 모양 | 뜻 |
|---|---|---|
| $X$ | $(B, 784)$ | 입력 그림 |
| $W_1$ | $(784, 128)$ | 첫 번째 층 가중치 |
| $h$ | $(B, 128)$ | 은닉층 출력 |
| $W_2$ | $(128, 10)$ | 두 번째 층 가중치 |
| $z_2$ | $(B, 10)$ | 숫자별 점수(로짓) |

가중치와 편향을 모두 합하면 101,770개다. 32비트 실수로 약 407KB이고, 이 숫자는 뒤의 "하드웨어로 보면"에서 다시 쓴다.

## 3. 손실과 경사 하강

출력 점수 $z_2$를 softmax로 확률 $p$로 바꾸고, 정답 칸의 확률이 1에 가까울수록 작아지는 교차 엔트로피를 손실로 쓴다.

$$p_k = \frac{e^{z_{2,k}}}{\sum_j e^{z_{2,j}}}, \qquad L = -\frac{1}{B}\sum_{n=1}^{B} \log p_{n, y_n}$$

학습은 손실 $L$을 가장 빨리 줄이는 방향, 즉 그래디언트의 반대 방향으로 가중치를 조금 옮기는 일을 반복하는 것이다. $\eta$는 한 번에 옮기는 크기(학습률)다.

$$W \leftarrow W - \eta \frac{\partial L}{\partial W}$$

문제는 $\partial L / \partial W$를 어떻게 구하느냐다. 가중치 하나를 아주 조금 바꿔 손실 변화를 재는 방법(수치 미분)도 있지만, 가중치가 10만 개면 순전파를 10만 번 넘게 돌려야 한다. 역전파는 이것을 순전파 한 번 정도의 비용으로 해결한다.

## 4. 역전파: 연쇄 법칙을 뒤에서부터

![순전파와 역전파 계산 그래프. 순전파는 X에서 z1, h, z2를 거쳐 손실 L을 계산하고, 역전파는 L에서 시작해 z2, h, z1 순서로 그래디언트를 전달하며 W2와 W1의 그래디언트를 구한다|full](/img/01M6EJDSNBMM4DKNDGF4PX1EYM/750cb9dacd03.webp)

역전파의 핵심은 위 그림의 아래 줄이다. 손실에서 시작해 한 칸씩 뒤로 가며 "이 값이 조금 바뀌면 손실이 얼마나 바뀌나"를 전달한다. 각 칸에서는 바로 다음 칸의 그래디언트에 자기 연산의 미분만 곱하면 된다(연쇄 법칙).

1. **softmax + 교차 엔트로피:** 둘을 합쳐 미분하면 놀랄 만큼 단순해진다. 정답 위치에서만 1을 빼면 된다.
$$\frac{\partial L}{\partial z_2} = \frac{1}{B}(p - \mathrm{onehot}(y))$$
2. **두 번째 층:** $z_2 = hW_2 + b_2$이므로
$$\frac{\partial L}{\partial W_2} = h^\top \frac{\partial L}{\partial z_2}, \qquad \frac{\partial L}{\partial h} = \frac{\partial L}{\partial z_2} W_2^\top$$
3. **ReLU:** 순전파 때 음수였던 칸은 출력이 0으로 막혔으니 그래디언트도 0이다.
$$\frac{\partial L}{\partial z_1} = \frac{\partial L}{\partial h} \odot \mathbb{1}[z_1 > 0]$$
4. **첫 번째 층:** 두 번째 층과 같은 꼴이다.
$$\frac{\partial L}{\partial W_1} = X^\top \frac{\partial L}{\partial z_1}$$

여기서 눈여겨볼 점이 두 가지다. 첫째, 역전파에 필요한 $X$, $z_1$, $h$는 모두 **순전파 때 계산한 값**이다. 그래서 학습할 때는 이 중간값을 메모리에 남겨 둬야 한다. 둘째, 역전파도 결국 **행렬곱**이다. 순전파와 같은 종류의 연산이 비슷한 양만큼 한 번 더 일어난다.

## 5. NumPy로 직접 구현하기

위 식을 그대로 코드로 옮겼다. PyTorch 같은 프레임워크 없이 NumPy만 쓴다.

```python
import numpy as np

def forward(p, X):
    z1 = X @ p['W1'] + p['b1']          # (B,128)
    h = np.maximum(z1, 0)               # ReLU
    z2 = h @ p['W2'] + p['b2']          # (B,10) 로짓
    z2 = z2 - z2.max(1, keepdims=True)  # softmax 수치 안정화
    prob = np.exp(z2) / np.exp(z2).sum(1, keepdims=True)
    return prob, (X, z1, h)             # 역전파에 쓸 중간값을 남겨 둔다

def backward(p, prob, cache, y):
    X, z1, h = cache
    B = len(y)
    dz2 = prob.copy(); dz2[np.arange(B), y] -= 1; dz2 /= B   # p - onehot
    g = {'W2': h.T @ dz2, 'b2': dz2.sum(0)}
    dh = dz2 @ p['W2'].T
    dz1 = dh * (z1 > 0)                                       # ReLU를 지나며 음수였던 칸은 0
    g['W1'] = X.T @ dz1; g['b1'] = dz1.sum(0)
    return g

# 미니배치 SGD: 배치 64, 학습률 0.1, 10 에폭 (배치 나누기는 줄여서 적음)
for epoch in range(10):
    for idx in batches(len(X_train), 64):
        prob, cache = forward(p, X_train[idx])
        for k, v in backward(p, prob, cache, y_train[idx]).items():
            p[k] -= 0.1 * v
```

가중치는 He 초기화($\mathcal{N}(0, 2/n_{in})$)로 시작했다. 초기화가 왜 중요한지는 다음 글에서 다룬다. 전체 코드(초기화, 데이터 읽기, 측정 포함)는 70줄 남짓이다.

### 역전파가 맞는지 먼저 검증

역전파 코드는 틀려도 학습이 "어느 정도는" 되기 때문에 버그를 놓치기 쉽다. 그래서 작은 모델(20-8-3, 배치 5)에서 모든 가중치에 대해 수치 미분 $\frac{L(w+\epsilon) - L(w-\epsilon)}{2\epsilon}$과 역전파 결과를 비교했다. 가장 크게 어긋난 상대 오차가 $2.4 \times 10^{-8}$로, 두 값이 사실상 같다. 이 확인을 거친 뒤에 학습을 돌렸다.

### 학습 결과

![에폭별 학습 손실과 테스트 정확도. 손실은 0.37에서 0.056으로 줄고, 테스트 정확도는 93.2%에서 97.7%로 오른다|xl](/img/01M6EJDSNBMM4DKNDGF4PX1EYM/d4ae9025b809.webp)

| 에폭 | 1 | 2 | 5 | 10 |
|---|---|---|---|---|
| 학습 손실 | 0.365 | 0.197 | 0.101 | 0.056 |
| 테스트 정확도 | 93.2% | 95.2% | 97.1% | **97.7%** |

학습 데이터 6만 장을 10번 도는 데 모두 1.6초(에폭마다 테스트 평가 포함, 에폭당 약 0.16초)가 걸렸다. 장비는 맥북 프로 M1 Max의 CPU이고, GPU는 쓰지 않았다(NumPy 2.5). 첫 에폭에서 이미 93%를 넘고, 이후로는 손실이 천천히 줄며 정확도가 조금씩 오른다. 은닉층 하나짜리 단순한 신경망은 98% 언저리에서 더 오르지 않는데, 이 벽을 넘는 것이 3편부터 다룰 CNN이다.

:::info
**하드웨어로 보면: 같은 코드가 왜 180배 빨라지나**

샘플 하나를 학습하는 데 드는 연산은 순전파 약 20만 번, 역전파 약 21만 번의 곱셈·덧셈(FLOP)으로 **약 41만 FLOP**이다. 역전파가 순전파와 거의 같은 양인 이유는 4절에서 본 대로 행렬곱이 한 번 더 일어나기 때문이다(입력 $X$에 대한 그래디언트는 필요 없어서 2배가 아니라 1배 남짓이다).

그런데 같은 연산량인데도 한 번에 넣는 샘플 수에 따라 걸리는 시간이 크게 달라진다.

![배치 크기별 샘플 하나당 순전파+역전파 시간. 배치 1에서 174마이크로초, 8에서 6.2, 64에서 1.5, 512에서 1.0, 4096에서 1.3마이크로초|xl](/img/01M6EJDSNBMM4DKNDGF4PX1EYM/92870f99be21.webp)

| 배치 크기 | 샘플당 시간 | 실제 연산 속도 |
|---|---|---|
| 1 | 174µs | 2.4 GFLOP/s |
| 64 | 1.5µs | 281 GFLOP/s |
| 512 | **1.0µs** | **424 GFLOP/s** |
| 4096 | 1.3µs | 318 GFLOP/s |

이유는 **데이터를 한 번 가져와서 몇 번 쓰느냐**(산술 강도, arithmetic intensity)에 있다. 첫 번째 층 순전파를 보면, 배치 1일 때는 407KB 가중치를 통째로 읽어 와서 곱셈·덧셈을 20만 번만 한다. 읽은 바이트당 연산이 0.5번뿐이라, 계산기는 대부분 데이터를 기다리며 논다. 배치 512에서는 같은 가중치를 512개 샘플에 재사용해 바이트당 약 45번을 계산한다. 계산기가 쉬지 않고 일하게 되는 것이다.

배치 4096에서 다시 느려지는 것은, 입력만 12.8MB로 커져서 CPU 캐시에 다 들어가지 않고 메모리를 오가는 일이 늘었기 때문으로 보인다(추정).

이 관찰이 이 시리즈 내내 반복된다. GPU, NPU, 그리고 메모리 안에서 계산하는 Compute-In-Memory 같은 AI 가속기는 모두 **"계산보다 데이터 이동이 비싸다"** 는 문제에 대한 서로 다른 답이다. 또 하나 기억할 것은 4절의 "중간값 저장"이다. 학습은 추론과 달리 순전파의 중간값을 역전파 때까지 들고 있어야 해서, 배치와 모델이 커질수록 메모리가 먼저 모자라게 된다. LLM 편에서 이 문제를 다시 만난다.
:::

## 6. 정리

- 신경망은 행렬곱과 비선형 함수를 번갈아 쌓은 함수이고, 학습은 경사 하강으로 손실을 줄이는 일이다.
- 역전파는 손실에서 시작해 연쇄 법칙으로 그래디언트를 한 칸씩 뒤로 전달한다. 각 칸의 계산은 단순하고, 대부분 행렬곱이다.
- 역전파 코드는 수치 미분으로 먼저 검증한다.
- 연산량이 같아도 데이터를 얼마나 재사용하느냐에 따라 속도가 수백 배 달라진다. 이것이 AI 하드웨어를 이해하는 출발점이다.

**다음 글:** 이번에는 He 초기화와 학습률 0.1을 별 설명 없이 썼다. 다음 글에서는 초기화를 잘못하면 무슨 일이 생기는지, 그리고 정규화와 SGD·Momentum·Adam 같은 최적화 방법이 학습을 어떻게 바꾸는지 같은 실습으로 비교한다.

## 참고 문헌

- D. E. Rumelhart, G. E. Hinton, R. J. Williams, "[Learning representations by back-propagating errors](https://www.nature.com/articles/323533a0)", *Nature* 323, 1986.
- K. He et al., "[Delving Deep into Rectifiers](https://arxiv.org/abs/1502.01852)", ICCV 2015 (He 초기화).
- I. Goodfellow, Y. Bengio, A. Courville, *[Deep Learning](https://www.deeplearningbook.org/)*, 6장 "Deep Feedforward Networks", MIT Press, 2016.
- Y. LeCun, C. Cortes, C. J. C. Burges, [MNIST 손글씨 숫자 데이터셋](https://yann.lecun.com/exdb/mnist/).
- S. Williams, A. Waterman, D. Patterson, "[Roofline: An Insightful Visual Performance Model](https://dl.acm.org/doi/10.1145/1498765.1498785)", *Communications of the ACM*, 2009 (산술 강도).
