---
layout: post
title: "백준 21611번 마법사 상어와 블리자드 - 효율적인 시뮬레이션 구현 기법"
date: 2024-07-25
categories: [Algorithm, 백준]
tags: [백준, 시뮬레이션, 센티넬값, 차원변환]
---

## 문제 개요
백준 21611번은 달팽이 모양 격자에서 구슬의 폭발/이동/변화를 시뮬레이션하는 문제입니다. 이 문제의 **핵심은 2차원 달팽이 구조를 1차원으로 변환하여 처리하는 것**입니다.

## 핵심 설계 아이디어: 차원 변환 전략

### 🔹 2D → 1D → 2D 변환의 타당성

**왜 이 접근법이 효과적인가?**

1. **달팽이 경로의 선형성**: 
   - 달팽이는 본질적으로 **순서가 있는 일렬 구조**
   - 2차원 좌표이지만 이동 순서는 명확히 정의됨

2. **구슬 조작의 특성**:
   - 폭발: 연속된 같은 구슬 4개 이상 제거
   - 당기기: 빈 공간을 앞으로 메우기  
   - 그룹화: 연속된 같은 구슬을 (개수, 종류) 쌍으로 변환
   - → **모두 순서가 중요한 1차원 배열 연산**

3. **알고리즘 복잡도 단순화**:
```python
# 2차원에서 직접 처리하면 복잡
# - 달팽이 방향 추적하며 폭발 검사
# - 빈 공간 처리를 위한 복잡한 좌표 계산

# 1차원으로 변환하면 간단
lst_arr = []
for r, c in snail_pos:  # 달팽이 순서대로 변환
    if arr[r][c] > 0:
        lst_arr.append(arr[r][c])
```

### 🔹 변환 과정의 구체적 구현

**Step 1: 2D → 1D 변환**
```python
# 미리 계산된 달팽이 경로 활용
snail_pos = [(중심부터 바깥으로의 좌표 순서)]

# 2차원 격자를 1차원 배열로 변환
lst_arr = []
for r, c in snail_pos:
    if arr[r][c] > 0:  # 빈 공간 제외
        lst_arr.append(arr[r][c])
```

**Step 2: 1D에서 모든 로직 처리**
- 폭발, 당기기, 그룹화 모두 1차원 배열 연산으로 처리
- 복잡한 좌표 계산 없이 인덱스만으로 처리 가능

**Step 3: 1D → 2D 복원**
```python
# 완전히 새로운 2차원 배열 생성
arr = [[0] * N for _ in range(N)]
for i in range(min(len(grp), len(snail_pos))):
    arr[snail_pos[i][0]][snail_pos[i][1]] = grp[i]
```

## 핵심 구현 기법들

### 🔹 센티넬 값을 활용한 구간 처리

**핵심 아이디어: [i, j) 반열린 구간 패턴**
- 길이 계산: `i <= x < j`이면 개수는 `j - i`
- 이 패턴을 깔끔하게 구현하려면 **끝값이 열려있어야** 함

**문제 상황:**
```python
# [1,1,1,2,2,3] 에서 마지막 구간 처리
i = 4  # 마지막 '3'의 위치
j = i + 1
while j < len(lst) and lst[i] == lst[j]:  # 마지막이면 조건 실패!
    j += 1
# 결과: j = 5 (정상), 하지만 경계 체크 때문에 복잡함
```

**센티넬 해결책:**
```python
lst.append(-1)  # 센티넬 추가: [1,1,1,2,2,3,-1]
i = 4  # 마지막 '3'의 위치  
j = i + 1
while lst[i] == lst[j]:  # 경계 체크 불필요!
    j += 1
# lst[4](3) != lst[6](-1) 에서 자연스럽게 멈춤
# 결과: j = 6, 길이 = j - i = 6 - 5 = 1 ✓

lst.pop()  # ⚠️ 센티넬 제거 필수!
```

**왜 마지막에 pop()이 필요한가?**
- 이후 로직에서 **배열 길이를 비교**하기 때문
- 센티넬이 포함된 상태로 길이를 비교하면 잘못된 결과
- 실제 데이터만으로 길이 비교해야 정확한 변화 감지 가능

```python
# bomb 함수 내부
def bomb(lst):
    lst.append(-1)  # 센티넬 추가
    # ... 구간 처리 로직 ...
    lst.pop()       # 센티넬 제거 (길이 복원)
    return nlst

# 메인 로직에서 길이 비교
while True:
    new_lst = bomb(lst_arr)
    if len(new_lst) == len(lst_arr):  # 정확한 길이 비교 가능
        break
```

### 🔹 길이 변화를 통한 변경 감지
```python
while True:
    new_lst = bomb(lst_arr)
    if len(new_lst) == len(lst_arr):  # 변화 없음 감지
        break
    lst_arr = new_lst
```
**핵심**: 폭발은 삭제만 발생 → 길이가 같으면 더 이상 변화 없음

### 🔹 원본 보존 후 새 배열 생성 패턴
```python
arr = [[0] * N for _ in range(N)]  # 완전히 새로 생성
for i in range(min(len(grp), len(snail_pos))):
    arr[snail_pos[i][0]][snail_pos[i][1]] = grp[i]
```
**타당성**: 시간복잡도가 O(N²)이므로 재생성 비용 무시 가능

## 달팽이 경로 생성의 수학적 패턴
```python
# 이동 거리: 1→1→2→2→3→3→4→4...
if even:
    even = False
    mx_cnt += 1  # 짝수 번째마다 거리 증가
else:
    even = True
```

## 정답 코드

```python
N, M = map(int, input().split())
arr = [list(map(int, input().split())) for _ in range(N)]
d_exp = []
s_exp = []
dir = [(0, -1), (1, 0), (0, 1), (-1, 0)]  # 달팽이 경로용 방향
snail_pos = []

for i in range(M):
    d, s = map(int, input().split())
    d_exp.append(d-1)
    s_exp.append(s)

r, c = N//2, N//2

def bomb(lst):
    lst.append(-1)  # 센티넬 값 추가
    nlst = []
    i = 0
    while i < len(lst) - 1:
        j = i + 1
        while lst[i] == lst[j]:
            j += 1
        if j - i > 3:  # 4개 이상 연속이면 폭발
            ans[lst[i]] += j - i
        else:
            nlst.extend(lst[i:j])
        i = j
    lst.pop()  # 센티넬 값 제거
    return nlst

ans = [0] * 4

# 달팽이 경로 미리 계산
cnt = 0
d = 0
mx_cnt = 1
even = False
while (r, c) != (0, 0):
    r, c = r + dir[d][0], c + dir[d][1]
    snail_pos.append((r, c))
    
    cnt += 1
    if cnt == mx_cnt:
        cnt = 0
        d = (d + 1) % 4
        if even:
            even = False
            mx_cnt += 1
        else:
            even = True

dir = [(-1, 0), (1, 0), (0, -1), (0, 1)]  # 블리자드용 방향
for i in range(M):
    # 블리자드 마법으로 구슬 파괴
    r, c, cd = N//2, N//2, d_exp[i]
    for dist in range(1, s_exp[i] + 1):
        arr[r + dir[cd][0] * dist][c + dir[cd][1] * dist] = 0
    
    # 2차원 → 1차원 변환
    lst_arr = []
    for r, c in snail_pos:
        if arr[r][c] > 0:
            lst_arr.append(arr[r][c])
    
    # 폭발 반복 (길이 변화로 종료 조건 감지)
    while True:
        lst = bomb(lst_arr)
        if len(lst) == len(lst_arr):
            break
        lst_arr = lst
    
    # 그룹화 (연속된 같은 구슬을 개수와 종류로 변환)
    i = 0
    grp = []
    lst_arr.append(-1)  # 센티넬 값 추가
    while i < (N * N - 1) and lst_arr[i] > 0:
        j = i + 1
        while lst_arr[i] == lst_arr[j]:
            j += 1
        grp.extend([j - i, lst_arr[i]])  # [개수, 종류] 순서로 추가
        i = j
    lst_arr.pop()  # 센티넬 값 제거
    
    # 1차원 → 2차원 복원
    arr = [[0] * N for _ in range(N)]
    for i in range(0, min(len(grp), len(snail_pos))):
        arr[snail_pos[i][0]][snail_pos[i][1]] = grp[i]

print(ans[1] * 1 + ans[2] * 2 + ans[3] * 3)
```

## 핵심 인사이트

**이 문제에서 배운 중요한 개념들:**

1. **차원 변환 설계**: 복잡한 2D 문제를 1D 문제로 단순화하는 통찰력
2. **달팽이의 본질**: 2차원 구조이지만 순서가 있는 1차원 데이터라는 인식
3. **센티넬 패턴**: [i, j) 구간으로 일관된 길이 계산과 경계 처리
4. **상태 관리**: 원본 보존 후 새 상태 생성으로 안전성 확보
5. **길이 기반 변화 감지**: 직접 비교 대신 메타데이터 활용

**결론**: 이 문제의 핵심은 2차원 달팽이 구조의 특성을 파악하여 1차원 배열 문제로 변환하는 통찰력입니다. 복잡해 보이는 시뮬레이션도 적절한 자료구조 변환을 통해 단순하고 효율적으로 해결할 수 있다는 것을 보여주는 좋은 예제입니다.
