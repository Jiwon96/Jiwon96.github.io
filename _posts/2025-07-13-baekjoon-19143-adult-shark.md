---
layout: post
title: "[백준 19143] 어른 상어 - 처리 순서가 곧 우선순위: 효율적인 충돌 해결법"
date: 2025-07-13
categories: [Algorithm, 백준]
---

## 1. 문제의 핵심: 충돌 처리 방식 선택

어른 상어 문제에서 가장 중요한 제약 조건:
- **한 자리에 하나의 상어만 존재 가능**
- 같은 위치에 여러 상어가 도착할 때 **낮은 번호가 우선순위**
- 상어들이 동시에 이동

핵심 질문: **"어떻게 우선순위를 효율적으로 처리할까?"**

이 문제를 해결하는 두 가지 접근 방식을 비교해보자.

## 2. 두 가지 접근 방식 비교

### 방식 1 (초기 접근): 모든 이동 → 정렬 → 중복 제거

```python
def move(shark):
    t_shark = []
    # 모든 상어를 일단 이동시킴
    for idx, r, c, d, cur_smell in shark:
        # ... 이동 로직
        t_shark.append([idx, nr, nc, nd, k])
    
    # 정렬로 우선순위 결정
    t_shark.sort(key=lambda x: (x[1], x[2], -x[4], x[0]))
    return t_shark

# 충돌 처리: 같은 위치의 상어들 중 우선순위 낮은 것 제거
for i in range(len(t_sharks)-1, 0, -1):
    if (t_sharks[i][1], t_sharks[i][2]) == (t_sharks[i-1][1], t_sharks[i-1][2]):
        t_s = t_sharks.pop(i)
```

### 방식 2 (개선된 접근): 순서대로 처리 → 실시간 충돌 해결

```python
# 상어를 번호순으로 처리 (0번부터)
for i in range(len(shk)):
    sn, si, sj, sd = shk[i]
    # 이동 로직...

# 실시간 충돌 처리
i = 0
while i < len(shk):
    sn, si, sj, sd = shk[i]
    # 이미 다른 상어가 있으면 즉시 제거
    if v[si][sj][0] != -1 and v[si][sj][0] != sn:
        shk.pop(i)  # 높은 번호이므로 바로 죽임
    else:
        v[si][sj] = [sn, K]  # 자리 차지
        i += 1
```

## 3. 핵심 통찰: 왜 순서대로 처리하는가?

### 처리 순서 = 우선순위 자동 보장

**핵심 아이디어**: 상어 배열이 이미 번호순으로 정렬되어 있다는 점을 활용

1. **0번 상어부터 순차 처리** → 낮은 번호가 먼저 자리 차지
2. **나중에 오는 상어는 "이미 누군가 있네?"** → 바로 제거
3. **복잡한 정렬 없이도 우선순위 규칙 자동 준수**

```python
# 상어가 이미 번호순으로 저장되어 있음
shk = [[0,1,1,1], [1,2,2,2], [2,3,3,3], ...]  # 0번, 1번, 2번 순서

# 순서대로 처리하면 자연스럽게 낮은 번호 우선
for i in range(len(shk)):  # 0번부터 처리
    # 먼저 처리된 상어가 자리 차지
```

이 방식의 장점:
- **O(n log n) 정렬이 O(n)으로 개선**
- 코드가 더 직관적이고 이해하기 쉬움
- 메모리 효율성 향상 (새로운 배열 생성 불필요)

## 4. 데이터 구조의 차이

### 초기 방식: 분리된 데이터 관리
```python
sharks = [[idx, r, c, d, smell], ...]  # 상어 정보
v = [1, 1, 0, 1, ...]                  # 생존 여부 (별도 관리)
```

### 개선된 방식: 통합된 데이터 관리
```python
v = [[[-1]*2 for _ in range(N)] for _ in range(N)]
# v[i][j][0] = 상어번호 (-1이면 빈칸)
# v[i][j][1] = 냄새 지속시간
```

통합된 데이터 구조의 장점:
- 상어의 위치와 냄새 정보를 한 번에 관리
- 충돌 감지가 단순한 배열 접근으로 가능
- 데이터 일관성 보장

## 5. 단계별 처리의 명확성

개선된 방식은 각 단계가 독립적으로 처리되어 디버깅과 이해가 쉽다:

```python
for ans in range(1, 1001):
    # [1] 상어 이동: 빈칸 우선 → 자기 냄새 차순위
    for i in range(len(shk)):
        # 이동 로직...
    
    # [2-1] 냄새 감소: 모든 칸의 냄새 -1
    for i in range(N):
        for j in range(N):
            if v[i][j][1] > 0:
                v[i][j][1] -= 1
    
    # [2-2] 충돌 처리: 이미 차지된 자리에 온 상어 제거
    i = 0
    while i < len(shk):
        # 충돌 검사 및 처리...
    
    # [2-3] 냄새 뿌리기: 생존한 상어들이 새 냄새 남김
    # (충돌 처리에서 동시에 처리됨)
```

## 6. for-else 구문의 효과적 활용

개선된 코드에서 for-else 구문이 더 자연스럽게 활용된다:

```python
# 빈칸 탐색 시도
for dr in dtbl[sn][sd]:
    ni, nj = si + di[dr], sj + dj[dr]
    if 0 <= ni < N and 0 <= nj < N and v[ni][nj][0] == -1:
        shk[i] = [sn, ni, nj, dr]
        break
else:  # 빈칸이 없었다면 자기 냄새 탐색
    for dr in dtbl[sn][sd]:
        ni, nj = si + di[dr], sj + dj[dr]
        if 0 <= ni < N and 0 <= nj < N and v[ni][nj][0] == sn:
            shk[i] = [sn, ni, nj, dr]
            break
```

이는 "빈칸 우선, 없으면 자기 냄새"라는 규칙을 매우 자연스럽게 표현한다.

## 7. 얻은 교훈

### 처리 순서 설계의 중요성

문제에서 요구하는 우선순위를 **별도의 정렬 없이** 자연스러운 처리 순서로 해결할 수 있다면, 그것이 가장 효율적인 방법이다.

### 복잡한 정렬보다는 자연스러운 순서 활용

- 이미 정렬된 데이터의 순서를 활용
- 처리 순서 자체가 우선순위를 보장하도록 설계
- 추가적인 정렬 연산 불필요

### 실시간 처리 vs 배치 처리

- **실시간 처리**: 문제가 발생하는 즉시 해결
- **배치 처리**: 모든 작업 후 일괄 처리
- 상황에 따라 실시간 처리가 더 효율적일 수 있음

## 8. 코드 비교

### 초기 코드 (복잡한 정렬 방식)
```python
def move(shark):
    t_shark = []
    for idx, r, c, d, cur_smell in shark:
        if cur_smell == 1:
            continue
        if cur_smell == k:
            if is_near_empty(r, c, shark):
                nr, nc, nd = find_near_empty(idx, r, c, d, shark)
            else:
                nr, nc, nd = find_next_pos(idx, r, c, d, shark)
            t_shark.append([idx, nr, nc, nd, k])
        t_shark.append([idx, r, c, d, cur_smell-1])
    
    # 복잡한 정렬
    t_shark.sort(key=lambda x: (x[1], x[2], -x[4], x[0]))
    return t_shark
```

### 개선된 코드 (순서 기반 처리)
```python
# 상어 이동 (이미 번호순으로 처리됨)
for i in range(len(shk)):
    sn, si, sj, sd = shk[i]
    for dr in dtbl[sn][sd]:
        ni, nj = si + di[dr], sj + dj[dr]
        if 0 <= ni < N and 0 <= nj < N and v[ni][nj][0] == -1:
            shk[i] = [sn, ni, nj, dr]
            break
    else:
        for dr in dtbl[sn][sd]:
            ni, nj = si + di[dr], sj + dj[dr]
            if 0 <= ni < N and 0 <= nj < N and v[ni][nj][0] == sn:
                shk[i] = [sn, ni, nj, dr]
                break

# 실시간 충돌 처리
i = 0
while i < len(shk):
    sn, si, sj, sd = shk[i]
    if v[si][sj][0] != -1 and v[si][sj][0] != sn:
        shk.pop(i)  # 즉시 제거
    else:
        v[si][sj] = [sn, K]  # 자리 차지
        i += 1
```

## 결론

복잡한 시뮬레이션 문제에서 가장 중요한 것은 **"어떤 순서로 처리할 것인가?"**이다. 

문제에서 요구하는 우선순위를 자연스러운 처리 순서로 해결할 수 있다면, 복잡한 정렬이나 비교 연산 없이도 깔끔하고 효율적인 코드를 작성할 수 있다.

**핵심 아이디어**: 처리 순서가 곧 우선순위다.