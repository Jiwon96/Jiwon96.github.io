---
layout: post
title: "[백준 19143] 어른 상어 - 시뮬레이션 문제의 핵심: 메소드 설계와 우선순위 정렬"
date: 2025-07-13
categories: [Algorithm, 백준]
---

## 1. 문제를 만났을 때의 접근법

시뮬레이션 문제를 풀 때 가장 중요한 것은 **"이 문제를 풀기 위한 핵심 메소드가 뭘까?"**라는 질문부터 시작하는 것이다.

어른 상어 문제의 핵심 제약 조건을 보면:
- **한 자리에 하나의 상어만 존재 가능**
- 상어들이 동시에 이동
- 같은 위치로 이동 시 우선순위에 따라 처리

이 제약 조건에서 **핵심 메소드**를 도출할 수 있다:
1. **이동 로직**: 각 상어가 어디로 이동할지 결정
2. **우선순위 정렬**: 충돌 시 어떤 상어가 살아남을지 결정

## 2. 초기 접근에서 놓쳤던 관점

처음 문제를 접했을 때는 복잡한 이동 조건들에만 매몰되어 있었다:
- 인접한 칸 중 아무도 없는 칸으로 이동
- 그런 칸이 없으면 자신의 냄새가 있는 칸으로 이동
- 방향 우선순위에 따른 선택

하지만 정작 중요한 **"충돌 시 어떻게 처리할까?"**라는 핵심 질문을 나중에 고려하게 되었다. 이로 인해 move() 함수가 복잡해지고, 로직이 여러 곳에 분산되는 문제가 발생했다.

## 3. 문제 해결의 핵심 아이디어

### 우선순위를 정해서 정렬 후 하나만 남기기

이 문제의 핵심은 **여러 상어가 같은 위치로 이동하려 할 때의 처리**이다:

```python
# 우선순위 기준: (행, 열, 냄새지속시간, 상어번호)
t_shark.sort(key=lambda x: (x[1], x[2], -x[4], x[0]))

# 같은 위치에 있는 상어들 중 우선순위가 낮은 것들 제거
for i in range(len(t_sharks)-1, 0, -1):
    if (t_sharks[i][1], t_sharks[i][2]) == (t_sharks[i-1][1], t_sharks[i-1][2]):
        t_s = t_sharks.pop(i)
        if t_s[4] == k:  # 새로 이동한 상어라면
            v[t_s[0]] = 0  # 죽음 처리
```

이렇게 **정렬 후 중복 제거**라는 단순한 아이디어로 복잡한 충돌 상황을 깔끔하게 해결할 수 있다.

## 4. 이동 로직 설계

### Empty vs Next의 차이

초기 코드에서 헷갈렸던 부분은 두 가지 탐색의 차이였다:

- **Empty**: 아무 상어도 없는 빈 칸을 우선적으로 탐색
- **Next**: 빈 칸이 없을 때만 자신의 냄새가 있는 칸을 탐색

```python
# 빈 칸이 있는지 확인
if is_near_empty(r, c, shark):
    nr, nc, nd = find_near_empty(idx, r, c, d, shark)
else:
    # 빈 칸이 없으면 자신의 냄새가 있는 곳으로
    nr, nc, nd = find_next_pos(idx, r, c, d, shark)
```

### for-else 구문 활용

조건을 만족하는 칸을 찾지 못했을 때의 처리에서 for-else 구문이 유용했다:

```python
def find_near_empty(idx, r, c, d, shark):
    dir_priority = shark_dir[idx]
    for nd in dir_priority[d]:
        nr, nc = r+dir_move[nd][0], c+dir_move[nd][1]
        if 0<=nr<N and 0<=nc<N:
            for i in range(len(shark)):
                if (nr, nc) == (shark[i][1], shark[i][2]):
                    break
            else:  # break가 실행되지 않았을 때만 실행
                return (nr, nc, nd)
```

for-else 구문은 내부 for문에서 break 없이 완료되었을 때만 else가 실행되어, "해당 위치에 다른 상어가 없음"을 확인할 수 있다.

## 5. 얻은 교훈

### 문제 분석 시 핵심 메소드부터 고민하기

복잡한 시뮬레이션 문제도 결국 몇 개의 핵심 메소드로 구성된다. 세부 구현에 바로 들어가기보다는:

1. **문제의 제약 조건 파악**
2. **제약 조건에서 핵심 메소드 도출**
3. **각 메소드의 역할 명확히 정의**
4. **메소드 간의 상호작용 설계**

이런 순서로 접근하면 더 깔끔한 코드를 작성할 수 있다.

### 제약 조건에서 해결 방법의 힌트 찾기

"한 자리에 하나만"이라는 제약 조건 자체가 **우선순위 정렬**이라는 해결 방법을 암시하고 있었다. 문제의 제약 조건을 단순히 구현해야 할 조건으로만 보지 말고, 해결 방법의 힌트로 활용하는 관점이 중요하다.

## 6. 정답 코드

```python
def is_near_empty(r, c, shark):
    for dr, dc in [(1, 0), (-1, 0), (0, 1), (0, -1)]:
        nr, nc = r + dr, c + dc
        if 0 <= nr < N and 0 <= nc < N:
            for i in range(len(shark)):
                if (nr, nc) == (shark[i][1], shark[i][2]):
                    break
            else:
                return True
    return False

def find_near_empty(idx, r, c, d, shark):
    dir_priority = shark_dir[idx]
    for nd in dir_priority[d]:
        nr, nc = r+dir_move[nd][0], c+dir_move[nd][1]
        if 0<=nr<N and 0<=nc<N:
            for i in range(len(shark)):
                if (nr, nc) == (shark[i][1], shark[i][2]):
                    break
            else:
                return (nr, nc, nd)

def find_next_pos(idx, r, c, d,shark):
    dir_priority = shark_dir[idx]
    for nd in dir_priority[d]:
        nr, nc = r + dir_move[nd][0], c + dir_move[nd][1]
        if 0 <= nr < N and 0 <= nc < N:
            for i in range(len(shark)):
                if (idx, nr, nc) == (shark[i][0], shark[i][1], shark[i][2]):
                    return (nr, nc, nd)

def move(shark):
    t_shark= []
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
    t_shark.sort(key=lambda x: (x[1], x[2], -x[4], x[0]))
    return t_shark

def solve():
    global sharks
    t = 0
    while sum(v) >1 and t<=1000:
        t += 1
        t_sharks = move(sharks)

        for i in range(len(t_sharks)-1, 0, -1):
            if (t_sharks[i][1], t_sharks[i][2]) == (t_sharks[i-1][1], t_sharks[i-1][2]):
                t_s = t_sharks.pop(i)
                if t_s[4] == k:
                    v[t_s[0]]=0

        sharks = t_sharks
    return t

dir_move = [(-1, 0), (1, 0), (0, -1), (0, 1)]
N, M, k = map(int, input().split())
sharks = [[] for _ in range(M)]
for i in range(N):
    lst = list(map(int, input().split()))
    for j in range(N):
        if lst[j] >0:
            sharks[lst[j]-1] = [lst[j]-1, i, j]
v = [1] * M
lst = list(map(int, input().split()))
for i in range(M):
    sharks[i].extend([lst[i] -1 , k])
shark_dir = dict()
for i in range(M):
    lst = [list(map(lambda x: int(x) -1, input().split())) for _ in range(4)]
    shark_dir[i] = lst
ret= solve()
print( ret if ret <= 1000 else -1)
```

복잡한 시뮬레이션 문제도 결국 단순한 핵심 아이디어들의 조합이다. 문제를 만나면 먼저 "핵심 메소드가 무엇인지" 고민하는 습관을 기르자.