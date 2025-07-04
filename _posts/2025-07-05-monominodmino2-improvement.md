---
layout: post
title: "백준 모노미노도미노 2 - 복잡한 코드를 우아하게 바꾼 과정"
categories: [Algorithm, 백준]
---

## 처음엔 이렇게 복잡했다

백준 20061번을 처음 접했을 때, 나는 기능별로 함수를 세분화하는 것이 좋은 코드라고 생각했다.

```python
def is_available(r, c, type, arr):  # 놓을 수 있는지 확인
def move(r, c, type, arr):         # 블록 이동
def transform(r, c, t):            # 좌표 변환  
def rollup(arr, cnt):              # 라인 제거 후 정리
def put(r, c, t, arr):             # 블록 배치
def calculate(arr):                # 점수 계산
def transform(lst,arr):            # 또 다른 변환 (중복!)
```

결과는? **함수 중복 정의**, **변수명 혼동**, **복잡한 인덱스 계산**... 완전 재앙이었다.

## 핵심 깨달음: pop과 insert의 마법

### 문제: 라인 완성 후 어떻게 처리할까?

기존 방식:
```
[0,0,0,0] ← 새로 만들어야 함
[0,1,0,1] 
[1,1,1,1] ← 이 라인 제거하고
[1,0,1,1] ← 이것들을 아래로...
[1,1,0,1]
```

복잡한 인덱스 계산으로 하나하나 옮기려 했다. 하지만...

### 해답: Python 리스트의 특성 활용

```python
# 완성된 라인 발견시
arr.pop(i)                    # i번째 행 통째로 제거
arr.insert(0, [0,0,0,0])      # 맨 위에 빈 행 추가
```

**시뮬레이션:**
```
Before:                After pop(2):           After insert(0, [0,0,0,0]):
[0,0,0,0] idx 0       [0,0,0,0] idx 0        [0,0,0,0] idx 0  ← 새로 추가
[0,1,0,1] idx 1       [0,1,0,1] idx 1        [0,0,0,0] idx 1  
[1,1,1,1] idx 2  →    [1,0,1,1] idx 2   →    [0,1,0,1] idx 2  
[1,0,1,1] idx 3       [1,1,0,1] idx 3        [1,0,1,1] idx 3  
[1,1,0,1] idx 4                              [1,1,0,1] idx 4  
```

**왜 이게 가능한가?**
- 2차원 배열을 1차원 리스트들의 집합으로 구성
- 각 행이 독립적인 리스트 객체
- Python 리스트의 동적 크기 조절 특성
- 인덱스 재계산 없이 자동으로 구조 유지

## 범위 처리의 핵심: i, j 기준 사고

### 기존의 착각
복잡한 좌표 변환으로 두 보드를 매핑하려 했다.

### 실제 해답
```python
def domino(arr, j, t):
    if t==1:
        for i in range(2, 6):           # i가 행, j가 열
            if arr[i][j] != 0:          # 충돌 지점 발견
                arr[i-1][j] = 1         # 바로 위에 배치
                break
        else:                           # 끝까지 못 찾으면
            arr[5][j] = 1               # 바닥에 배치
```

**시뮬레이션 - 타입1 블록이 열1에 떨어질 때:**
```
    0 1 2 3
0  [0,0,0,0]  
1  [0,0,0,0]  
2  [0,0,0,0]  ← i=2, arr[2][1]=0, 계속
3  [0,0,0,0]  ← i=3, arr[3][1]=0, 계속  
4  [0,1,0,0]  ← i=4, arr[4][1]=1, 충돌!
5  [0,1,1,0]     → arr[3][1]=1로 배치 (i-1)
```

**왜 i, j 기준이 좋은가?**
- "못 놓는 곳"을 바로 특정 가능
- 복잡한 좌표 변환 불필요
- 충돌 감지가 직관적

## 관찰의 중요성: 그림을 훼손하지 말라

가장 큰 실수는 **처음부터 코드부터 짜려 했던 것**이다.

### 올바른 접근법:
1. **문제 그림을 오래 관찰**
2. **패턴 파악** (두 보드의 관계, 블록 타입별 매핑)
3. **자료구조의 특성 파악** (리스트의 pop/insert 가능성)
4. **그제서야 코딩 시작**

### 최종 결과:
```python
def domino(arr, j, t):
    global s
    # 블록 배치 (간단한 for문으로)
    # 라인 완성 체크 및 제거 (pop/insert로)
    # 특별 영역 처리 (while로)
```

**7개 함수 → 1개 함수**로 줄었지만, 훨씬 명확하고 버그 없는 코드가 되었다.

## 교훈

1. **자료구조의 내장 메서드 활용**: pop, insert의 위력
2. **문제의 본질 파악**: i, j 기준으로 생각하기  
3. **충분한 관찰 시간**: 코딩보다 사고가 먼저
4. **복잡함보다 명확함**: 함수 분리가 항상 좋은 건 아니다

결국 프로그래밍은 **"어떻게 구현할까"**보다 **"어떻게 생각할까"**가 더 중요하다는 걸 다시 한번 깨달았다.
