---
title: "[백준] 23290번 마법사 상어와 복제 - 시뮬레이션 최적화 학습기"
categories: [Algorithm, 백준]
tags: [시뮬레이션, 구현, 자료구조, 최적화]
date: 2025-08-16
---

## 문제 분석

[백준 23290번 마법사 상어와 복제](https://www.acmicpc.net/problem/23290) 문제는 4×4 격자에서 물고기와 상어가 상호작용하는 복잡한 시뮬레이션 문제입니다. 

주요 과정:
1. 물고기 복사
2. 물고기 이동 (반시계방향 45도씩 회전하며 이동 가능한 칸 탐색)
3. 상어 3칸 이동 (가장 많은 물고기를 잡을 수 있는 경로 선택)
4. 냄새 감소
5. 복사된 물고기 추가

## 초기 접근 방식의 문제점

### 1. 비효율적인 자료구조 선택
처음에는 각 물고기를 개별 객체로 관리하려 했습니다. 하지만 같은 위치에 여러 물고기가 있을 때 모든 물고기를 개별적으로 처리하면 시간복잡도가 급격히 증가합니다.

```python
# 비효율적인 접근
fish_list = [(r1, c1, d1), (r2, c2, d2), (r3, c3, d3), ...]
# 같은 위치의 물고기들도 모두 개별 처리
```

### 2. 상태 변화 시 데이터 정리 부족
물고기 이동이나 복제 후에 같은 위치, 같은 방향의 물고기들을 정리하지 않으면 데이터가 중복되어 관리가 복잡해집니다.

### 3. 시간복잡도 고려 부족
각 턴마다 모든 물고기를 개별적으로 처리하면 O(M×8) (M: 물고기 수)의 시간이 소요되는데, 물고기가 많아질수록 비효율적입니다.

## 개선된 접근 방식

### 1. 효율적인 자료구조 활용
```python
# 개선된 접근: [r, c, d, cnt] 형태로 통합 관리
fish = [[r, c, d, cnt], ...]  # 같은 위치, 같은 방향의 물고기를 count로 관리
```

이렇게 하면 같은 위치의 물고기들을 하나의 데이터로 처리할 수 있어 시간복잡도가 크게 개선됩니다.

### 2. merge 함수 도입
```python
def merge(fish):
    fish.sort(key = lambda x: (x[0], x[1], x[2]))  # r, c, d 순으로 정렬
    for i in range(len(fish)-1, 0, -1):
        if fish[i][:3] == fish[i-1][:3]:  # 위치와 방향이 같으면
            fish[i-1][3] += fish[i][3]    # 개수 합치기
            fish.pop(i)                   # 중복 제거
```

**핵심 포인트**: 물고기 상태가 변할 때마다 merge를 호출해야 합니다!
- 물고기 이동 후
- 복제된 물고기 추가 후

### 3. set을 활용한 좌표 처리
```python
# 상어 이동 경로에서 중복 위치 자동 제거
shark_set = {(nr1, nc1), (nr2, nc2), (nr3, nc3)}
```

상어가 같은 위치를 여러 번 지나가더라도 set을 사용하면 중복이 자동으로 제거되어 물고기 개수 계산이 정확해집니다.

## 핵심 학습 포인트

### 1. 시뮬레이션에서의 자료구조 최적화
- 같은 속성을 가진 객체들은 통합 관리
- count 방식으로 개수 처리하여 메모리와 시간 효율성 확보

### 2. 상태 변화 시점의 데이터 정리
- 매 턴마다 데이터 일관성 유지 필요
- merge 함수를 통한 체계적인 데이터 정리

### 3. 복잡한 조건 처리의 체계화
- 하드코딩을 활용한 편의성 확보 (`(2,0,6,4)` 방향 처리)
- set 자료구조를 활용한 중복 제거

## 정답 코드

```python
M, S = map(int, input().split())
fish = [list(map(lambda x: int(x)-1, input().split())) + [1] for _ in range(M)]

si, sj = map(lambda x: int(x)-1, input().split()) # 상어 위치
dr = [0, -1, -1, -1, 0, 1, 1, 1]
dc = [-1, -1, 0, 1, 1, 1, 0, -1]
v = [[0] * 4 for _ in range(4)]

def merge(fish):
    fish.sort(key = lambda x: (x[0], x[1], x[2]))
    for i in range(len(fish)-1, 0, -1):
        if fish[i][:3] == fish[i-1][:3]:
            fish[i-1][3] +=fish[i][3]
            fish.pop(i)

def move_shark(si, sj):
    max_shk = -1; del_set = {}
    for d1 in ((2,0,6,4)):
        nr1, nc1 = si+dr[d1], sj+dc[d1]
        if 0<=nr1<4 and 0<=nc1<4:
            for d2 in ((2, 0, 6, 4)):
                nr2, nc2 = nr1+dr[d2], nc1+dc[d2]
                if 0 <= nr2 < 4 and 0 <= nc2 < 4:
                    for d3 in ((2, 0, 6, 4)):
                        nr3, nc3 = nr2 + dr[d3], nc2 + dc[d3]
                        if 0 <= nr3 < 4 and 0 <= nc3 < 4:
                            shark_set = {(nr1, nc1), (nr2, nc2), (nr3, nc3)}
                            cnt = 0
                            for i in range(len(fish)):
                                if (fish[i][0], fish[i][1]) in shark_set:
                                    cnt+=fish[i][3]
                            if max_shk < cnt:
                                del_set = shark_set
                                nsi, nsj = nr3, nc3
                                max_shk = cnt
    for i in range(len(fish)-1, -1, -1):
        fi, fj = fish[i][0], fish[i][1]
        if (fi, fj) in del_set:
            v[fi][fj] =3
            fish.pop(i)

    return nsi, nsj

for s in range(S):
# [1] 상어 복사
    copy_f = [f[:] for f in fish]

# [2] 물고기 이동
    for i in range(len(fish)):
        for d in range(8):
            nd = (fish[i][2] - d)%8
            nr, nc = fish[i][0]+dr[nd], fish[i][1]+dc[nd]
            if 0<=nr<4 and 0<=nc<4 and not v[nr][nc] and (nr, nc) != (si, sj):
                fish[i][0], fish[i][1], fish[i][2] = nr, nc, nd
                break

    merge(fish)

# [3]상어 이동
    si, sj = move_shark(si, sj)

# [4] 냄새 삭제
    for i in range(4):
        for j in range(4):
            v[i][j] = max(0, v[i][j] -1)
# [5] 복제
    fish+=copy_f
    merge(fish)

ans =0
for i in range(len(fish)):
    ans+=fish[i][3]
print(ans)
```

## 마무리

이 문제를 통해 배운 가장 중요한 교훈은 **시뮬레이션 문제에서의 자료구조 최적화**입니다. 

단순히 문제의 조건을 그대로 구현하는 것이 아니라, 같은 속성을 가진 객체들을 통합 관리하고, 상태 변화 시점마다 데이터 일관성을 유지하는 것이 핵심이었습니다.

특히 `merge` 함수의 중요성을 놓치기 쉬운데, 물고기 상태가 변할 때마다 호출해야 한다는 점을 반드시 기억해야 합니다.
