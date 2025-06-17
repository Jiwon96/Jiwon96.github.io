---
title: "신경망 프로그래밍 기초: 이진 분류와 로지스틱 회귀"
date: 2024-12-01
categories: [AI]
tags: [neural network, binary classification, logistic regression, deep learning, gradient descent, vectorization]
---

# 신경망 프로그래밍 기초: 이진 분류와 로지스틱 회귀

Andrew Ng의 deeplearning.ai 강의 자료를 기반으로 신경망 프로그래밍의 기초 개념들을 정리해보겠습니다.

## 1. 이진 분류 (Binary Classification)

이진 분류는 입력 데이터를 두 개의 클래스 중 하나로 분류하는 문제입니다. 대표적인 예시로 고양이 vs 고양이가 아닌 것을 구분하는 문제가 있습니다.

### 이미지 데이터 표현

64x64 픽셀의 컬러 이미지는 다음과 같이 표현됩니다:
- Red, Green, Blue 각 채널마다 64x64 = 4,096개의 픽셀 값
- 총 특성 개수: n = nx = 4,096 × 3 = 12,288

이미지는 벡터 x로 변환되어 처리되며, 출력 y는 다음과 같습니다:
- y = 1 (고양이)
- y = 0 (고양이가 아님)

## 2. 로지스틱 회귀 (Logistic Regression)

### 기본 수식

로지스틱 회귀의 핵심 수식은 다음과 같습니다:

```
ŷ = σ(wᵀx + b)
```

여기서 σ(z) = 1/(1+e⁻ᶻ) 는 시그모이드 함수입니다.

### 손실 함수 (Loss Function)

단일 훈련 예시에 대한 손실 함수:

```
ℒ(ŷ, y) = -(y log(ŷ) + (1-y) log(1-ŷ))
```

### 비용 함수 (Cost Function)

전체 m개 훈련 예시에 대한 비용 함수:

```
J(w, b) = (1/m) Σᵢ₌₁ᵐ ℒ(ŷ⁽ⁱ⁾, y⁽ⁱ⁾)
```

## 3. 경사 하강법 (Gradient Descent)

경사 하강법은 비용 함수 J(w, b)를 최소화하는 매개변수 w와 b를 찾는 최적화 알고리즘입니다.

### 업데이트 규칙

```
w := w - α(∂J/∂w)
b := b - α(∂J/∂b)
```

여기서 α는 학습률(learning rate)입니다.

## 4. 미분과 계산 그래프

### 미분의 직관

함수 f(a) = 3a에서:
- a = 2일 때, f(a) = 6
- a = 2.001일 때, f(a) = 6.003
- 기울기 = 3 = df/da

### 계산 그래프

복잡한 함수의 미분을 효율적으로 계산하기 위해 계산 그래프를 사용합니다:

예시: J = 3(a + bc)에서 a=5, b=3, c=2
- u = bc = 6
- v = a + u = 11  
- J = 3v = 33

역전파를 통해 각 변수에 대한 미분을 계산할 수 있습니다.

## 5. 로지스틱 회귀의 경사 하강법

### 단일 예시에 대한 그래디언트

```
dz = a - y
dw₁ = x₁ × dz
dw₂ = x₂ × dz  
db = dz
```

### m개 예시에 대한 알고리즘

```python
J = 0, dw1 = 0, dw2 = 0, db = 0
for i in range(m):
    z = w1*x1[i] + w2*x2[i] + b
    a = sigmoid(z)
    J += -(y[i]*log(a) + (1-y[i])*log(1-a))
    dz = a - y[i]
    dw1 += x1[i] * dz
    dw2 += x2[i] * dz
    db += dz

J /= m
dw1 /= m
dw2 /= m  
db /= m
```

## 6. 벡터화 (Vectorization)

### 벡터화의 중요성

명시적인 for 루프를 피하고 벡터화된 연산을 사용하면 계산 속도가 크게 향상됩니다.

### NumPy 예시

벡터화 전:
```python
z = 0
for i in range(n):
    z += w[i] * x[i]
z += b
```

벡터화 후:
```python
z = np.dot(w, x) + b
```

## 7. 로지스틱 회귀 벡터화

### Forward Propagation

```python
Z = np.dot(w.T, X) + b
A = sigmoid(Z)
```

### Backward Propagation

```python
dZ = A - Y
dw = (1/m) * np.dot(X, dZ.T)
db = (1/m) * np.sum(dZ)
```

### 완전한 벡터화된 구현

```python
# 하나의 반복으로 모든 m개 예시 처리
Z = np.dot(w.T, X) + b
A = sigmoid(Z)
dZ = A - Y
dw = (1/m) * np.dot(X, dZ.T)
db = (1/m) * np.sum(dZ)

# 매개변수 업데이트
w = w - alpha * dw
b = b - alpha * db
```

## 8. Python의 브로드캐스팅

NumPy의 브로드캐스팅은 다른 크기의 배열 간 연산을 가능하게 합니다.

### 일반적인 브로드캐스팅 규칙

- (m,n) + (1,n) → (m,n)
- (m,n) + (m,1) → (m,n)
- (m,1) + (1,n) → (m,n)

## 결론

이진 분류를 위한 로지스틱 회귀는 신경망의 기본 구성 요소입니다. 벡터화와 효율적인 그래디언트 계산을 통해 대규모 데이터셋에서도 빠르게 학습할 수 있습니다. 이러한 기초 개념들은 더 복잡한 신경망 아키텍처를 이해하는 데 필수적입니다.

---

*이 포스트는 Andrew Ng의 deeplearning.ai 강의 자료를 기반으로 작성되었습니다.*