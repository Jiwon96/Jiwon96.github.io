---
layout: post
title: "Deep Neural Networks 완벽 가이드 - Andrew Ng 강의 요약"
date: 2025-06-17 20:00:00 +0900
categories: [AI]
tags: [deep learning, neural networks, andrew ng, forward propagation, backward propagation, ai]
pin: true
---

# Deep Neural Networks 완벽 가이드

## 🧠 Deep Neural Network란?

Deep Neural Network(DNN)은 여러 개의 은닉층(hidden layer)을 가진 신경망을 말합니다. 

### 네트워크 깊이별 분류
- **Shallow**: Logistic Regression (은닉층 0개)
- **얕은 네트워크**: 1-2개 은닉층
- **깊은 네트워크**: 3개 이상 은닉층

![네트워크 진화 과정]
```
로지스틱 회귀 → 1 은닉층 → 2 은닉층 → 5+ 은닉층 (Deep)
```

## 📊 DNN 표기법 (Notation)

### 기본 표기
- **L**: 전체 층의 개수 (L=4인 경우 4층 네트워크)
- **n^[l]**: l번째 층의 유닛(뉴런) 개수
- **a^[l]**: l번째 층의 활성화(activation) 값
- **W^[l], b^[l]**: l번째 층의 가중치와 편향

### 예시
```
n^[0] = 입력 특성 개수
n^[1] = 5 (첫 번째 은닉층 뉴런 5개)
n^[2] = 3 (두 번째 은닉층 뉴런 3개)  
n^[L] = 1 (출력층 뉴런 1개)
```

## ⚡ Forward Propagation (순전파)

각 층에서 다음 계산을 수행합니다:

```
Z^[l] = W^[l] * A^[l-1] + b^[l]
A^[l] = g^[l](Z^[l])
```

### 전체 과정
1. **입력층**: X = A^[0]
2. **은닉층들**: 
   - Z^[1] = W^[1]A^[0] + b^[1]
   - A^[1] = g^[1](Z^[1])
   - Z^[2] = W^[2]A^[1] + b^[2]
   - A^[2] = g^[2](Z^[2])
   - ...
3. **출력층**: ŷ = A^[L]

## 🔄 Backward Propagation (역전파)

### 핵심 공식들
```
dZ^[L] = A^[L] - Y
dW^[l] = (1/m) * dZ^[l] * A^[l-1]^T
db^[l] = (1/m) * np.sum(dZ^[l], axis=1, keepdims=True)
dZ^[l-1] = W^[l]^T * dZ^[l] * g'^[l-1](Z^[l-1])
```

### 역전파 흐름
```
손실함수 → dZ^[L] → dW^[L], db^[L] → dZ^[L-1] → ... → dZ^[1] → dW^[1], db^[1]
```

## 🎯 매트릭스 차원 맞추기

### 가중치 행렬 차원
- **W^[l]**: (n^[l], n^[l-1])
- **b^[l]**: (n^[l], 1)
- **A^[l]**: (n^[l], m) - m은 샘플 개수

### 벡터화 구현
```python
# Forward
Z = np.dot(W, A_prev) + b
A = activation_function(Z)

# Backward  
dW = (1/m) * np.dot(dZ, A_prev.T)
db = (1/m) * np.sum(dZ, axis=1, keepdims=True)
dA_prev = np.dot(W.T, dZ)
```

## 🤔 왜 Deep한 네트워크가 필요할까?

### 1. 계층적 특성 학습
- **1층**: 엣지, 선 감지
- **2층**: 기본 도형, 패턴 감지  
- **3층**: 얼굴 부위 감지
- **4층**: 전체 얼굴 인식

### 2. 표현력의 기하급수적 증가
- 얕은 네트워크로 같은 함수를 구현하려면 **지수적으로 많은 뉴런**이 필요
- 깊은 네트워크는 **효율적인 계층적 표현** 가능

### 3. 실제 예시: 얼굴 인식
```
픽셀 → 엣지 → 얼굴 부위 → 얼굴 전체
```

## 🏗️ 구현 시 Building Blocks

### Forward 함수
```python
def forward_layer(A_prev, W, b, activation):
    Z = np.dot(W, A_prev) + b
    A = activation(Z)
    cache = (A_prev, W, b, Z)  # 역전파용 저장
    return A, cache
```

### Backward 함수  
```python
def backward_layer(dA, cache, activation_backward):
    A_prev, W, b, Z = cache
    dZ = activation_backward(dA, Z)
    dW = (1/m) * np.dot(dZ, A_prev.T)
    db = (1/m) * np.sum(dZ, axis=1, keepdims=True)
    dA_prev = np.dot(W.T, dZ)
    return dA_prev, dW, db
```

## ⚙️ Parameters vs Hyperparameters

### Parameters (학습되는 값)
- W^[1], b^[1], W^[2], b^[2], ...
- 네트워크가 학습 과정에서 자동으로 업데이트

### Hyperparameters (사람이 설정)
- **학습률 (α)**: 0.01, 0.001, ...
- **반복 횟수**: 1000, 5000, ...
- **은닉층 개수**: 2, 3, 4, ...
- **은닉 유닛 개수**: 50, 100, ...
- **활성화 함수**: ReLU, Sigmoid, tanh
- **정규화 매개변수**

## 🔬 실무에서의 Deep Learning

### 경험적 과정 (Empirical Process)
```
아이디어 💡 → 코드 구현 💻 → 실험 🧪 → 결과 분석 📊 → 개선된 아이디어 💡
```

### 도메인별 특성
- **Computer Vision**: 매우 깊은 네트워크 선호
- **Speech Recognition**: 다양한 아키텍처 실험
- **NLP**: 트랜스포머 등 특화 구조
- **광고/추천**: 상대적으로 얕은 네트워크도 효과적

## 🧠 뇌와의 연관성

### 유사점
- 뉴런들의 계층적 연결
- 신호 전달 방식

### 차이점  
- 실제 뇌는 훨씬 복잡한 구조
- 인공 신경망은 수학적 모델에 불과
- **"뇌에서 영감을 얻었지만, 뇌를 모방한 것은 아니다"**

## 💡 핵심 정리

1. **Deep Network = 많은 은닉층**을 가진 신경망
2. **계층적 특성 학습**으로 복잡한 패턴 인식 가능
3. **Forward-Backward 프로세스**로 학습 진행
4. **Hyperparameter 튜닝**이 성능에 매우 중요
5. **경험과 실험**을 통한 반복적 개선이 핵심

Deep Learning은 이론보다는 **실험과 경험**이 중요한 분야입니다. 다양한 하이퍼파라미터를 시도해보고, 결과를 분석하며 점진적으로 개선해나가는 것이 성공의 열쇠입니다! 🚀