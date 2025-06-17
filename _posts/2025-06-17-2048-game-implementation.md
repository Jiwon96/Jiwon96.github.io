---
layout: post
title: "2048 게임 구현하기 - 차원 분리와 함수 설계의 중요성"
date: 2025-06-17 23:00:00 +0900
categories: [Algorithm, 백준]
---

# 2048 게임 구현하기 - 차원 분리와 함수 설계의 중요성

## 문제 분석
2048 게임은 4×4 격자에서 블록을 상하좌우로 밀어서 같은 숫자끼리 합치는 게임입니다. 5번의 이동으로 만들 수 있는 가장 큰 수를 찾는 문제를 해결하면서 배운 설계 원칙들을 정리해보겠습니다.

## 초기 설계의 문제점

처음에는 모든 로직을 하나의 함수에 넣으려 했습니다:

```python
# 나쁜 설계 - 모든 것을 한 함수에
def process_direction(arr, direction):
    # 회전 로직 + 병합 로직 + 복원 로직이 뒤섞임
    if direction == 'left':
        # 복잡한 중첩 로직...
    elif direction == 'right':
        # 또 다른 복잡한 로직...
    # 디버깅이 어려움!
```

## 핵심 인사이트: 차원 분리

**중요한 깨달음**: 2048은 2차원 게임이지만, 실제로는 **1차원 배열들의 독립적이고 병렬적인 처리**입니다.

```python
# 각 행은 완전히 독립적
[2, 0, 2, 4] → [4, 4, 0, 0]  # 첫 번째 행
[0, 4, 4, 0] → [8, 0, 0, 0]  # 두 번째 행
[2, 2, 0, 8] → [4, 8, 0, 0]  # 세 번째 행
```

이 관찰을 바탕으로 **책임을 분리**했습니다:
- `process()`: 1차원 배열의 블록 병합 담당
- `roll()`: 2차원 배열의 회전과 전체 게임 로직 담당

## 1차원 블록 병합 로직 (process 함수)

```python
def process(lst):  # 순수 1차원 로직
    new_lst = []
    i = 0
    while i < N:
        num = lst[i]
        if num == 0:  # 빈 칸 건너뛰기
            i += 1
            continue
        
        # 같은 숫자 찾기 (중간의 0은 무시)
        for j in range(i + 1, len(lst)):
            if lst[j] == 0:
                continue  # 0은 무시하고 계속
            if lst[j] != num:
                break     # 다른 숫자면 중단
            
            # 같은 숫자 발견! 병합
            num *= 2
            lst[j] = 0    # 병합된 블록은 0으로 표시
            i = j         # 병합된 위치부터 다시 시작
            break
        
        if num != 0:
            new_lst.append(num)
        i += 1
    
    return new_lst + [0] * (N - len(new_lst))
```

**핵심 아이디어:**
1. **연속된 같은 값 병합**: 중간에 0이 있어도 무시하고 같은 숫자 찾기
2. **한 번만 병합 규칙**: `i = j`로 이미 병합된 블록은 건너뛰기
3. **한 방향 압축**: 모든 블록이 왼쪽으로 밀림

**예시:**
```
[2, 0, 2, 4] → [4, 4, 0, 0]
[4, 4, 2, 2] → [8, 4, 0, 0]  # 첫 번째 4들만 합쳐짐
[2, 2, 2, 2] → [4, 4, 0, 0]  # 순차적으로 앞의 두 개, 뒤의 두 개
```

## 4방향 회전을 왼쪽 밀기로 통일

모든 방향을 **"왼쪽으로 밀기"**로 변환하여 `process` 함수를 재사용:

```python
def roll(cnt, tmp):
    arr_t = list(map(list, zip(*tmp)))  # 전치행렬 (행↔열 변환)

    for d in range(4):
        if d == 0:      # 좌: 행을 그대로
            temp = [lst[::] for lst in tmp]
        elif d == 1:    # 우: 행을 뒤집어서 (우→좌 변환)
            temp = [lst[::-1] for lst in tmp]
        elif d == 2:    # 상: 열을 행으로 (전치행렬)
            temp = [lst[::] for lst in arr_t]
        elif d == 3:    # 하: 열을 행으로 + 뒤집기
            temp = [lst[::-1] for lst in arr_t]

        # 모든 방향을 왼쪽 밀기로 처리!
        for i in range(N):
            temp[i] = process(temp[i])

        roll(cnt + 1, temp)
```

## 함수 분리의 장점들

### 1. **단일 책임 원칙 (SRP)**
```python
def process(lst):     # 책임: 1차원 배열 병합만
def roll():          # 책임: 2차원 회전과 게임 로직만
```

### 2. **테스트 용이성**
```python
# 1차원 로직을 독립적으로 테스트 가능
assert process([2, 0, 2, 4]) == [4, 4, 0, 0]
assert process([4, 4, 2, 2]) == [8, 4, 0, 0]

# 문제 발생시 1차원 vs 2차원 로직 구분 가능
```

### 3. **재사용성**
`process` 로직은 다른 문제에서도 활용 가능:
- 테트리스의 블록 떨어뜨리기
- 캔디 크러쉬의 매치3 제거
- Run-length encoding
- 중력 시뮬레이션

## 내가 범했던 실수와 개선점

### 1. **for문에서 인덱스 직접 조작**
```python
# 위험했던 코드
for i in range(N):
    # ...
    i = j  # for문 인덱스 직접 변경은 예측 어려움
```

**해결**: while문으로 변경하여 인덱스 조작을 명시적으로 만듦

### 2. **모놀리식 함수 설계**
처음에는 모든 로직을 하나의 함수에 넣으려 했지만, 차원별로 분리하니 훨씬 명확해졌습니다.

### 3. **2048 게임 규칙 이해 부족**
"한 번의 이동에서 하나의 블록은 최대 한 번만 합쳐진다"는 규칙을 놓쳐서 잘못된 결과가 나왔었습니다.

## 최적화: 가지치기

```python
# 현재 최대값으로 남은 횟수 동안 최선을 다해도 기존 답을 못 넘으면 중단
if (max(map(max, tmp)) * (2 ** (5 - cnt))) <= ret:
    return
```

## 전체 코드

```python
N = int(input())
arr = [list(map(int, input().split())) for _ in range(N)]
ret = 0

def process(lst):
    new_lst = []
    i = 0
    while i < N:
        num = lst[i]
        if num == 0:
            i += 1
            continue
        for j in range(i + 1, len(lst)):
            if lst[j] == 0:
                continue
            if lst[j] != num:
                break
            num *= 2
            lst[j] = 0
            i = j
            break
        if num != 0:
            new_lst.append(num)
        i += 1
    return new_lst + [0] * (N - len(new_lst))

def roll(cnt, tmp):
    global ret
    if (max(map(max, tmp)) * (2 ** (5 - cnt))) <= ret:
        return

    if cnt == 5:
        ret = max(ret, max(map(max, tmp)))
        return

    arr_t = list(map(list, zip(*tmp)))

    for d in range(4):
        if d == 0:
            temp = [lst[::] for lst in tmp]
        elif d == 1:
            temp = [lst[::-1] for lst in tmp]
        elif d == 2:
            temp = [lst[::] for lst in arr_t]
        elif d == 3:
            temp = [lst[::-1] for lst in arr_t]

        for i in range(N):
            temp[i] = process(temp[i])

        roll(cnt + 1, temp)

roll(0, arr)
print(ret)
```

## 배운 설계 원칙들

1. **차원별 책임 분리**: 복잡한 다차원 문제를 단순한 차원으로 분해
2. **Pure Function 지향**: 입력만으로 출력이 결정되는 함수
3. **단일 책임 원칙**: 하나의 함수는 하나의 일만
4. **코드 재사용**: 4방향을 1방향으로 통일하여 로직 재사용
5. **명시적 제어 흐름**: while문으로 인덱스 조작을 명확하게

이런 원칙들은 2048뿐만 아니라 다른 시뮬레이션 문제에서도 활용할 수 있는 중요한 설계 패턴입니다.