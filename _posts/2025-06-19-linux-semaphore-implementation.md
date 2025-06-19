---
title: "리눅스 시스템 프로그래밍: 세마포어(Semaphore) 구현과 P/V 연산"
date: 2025-06-19 22:00:00 +0900
categories: [Linux, 시스템프로그래밍]
tags: [semaphore, 세마포어, IPC, 동기화, 시스템콜]
---

# 리눅스 세마포어: 프로세스 간 동기화의 핵심

프로세스 간 통신(IPC)에서 동기화는 매우 중요한 개념입니다. 특히 공유 자원에 대한 접근을 제어할 때 세마포어는 핵심적인 역할을 담당합니다. 이번 포스트에서는 리눅스 환경에서 세마포어를 직접 구현하고 분석해보겠습니다.

## 1. 세마포어란?

**세마포어(Semaphore)**는 1965년 다익스트라(Dijkstra)가 제안한 동기화 도구로, 공유 자원에 대한 접근을 제어하는 정수 변수입니다.

### 핵심 개념
- **P연산(wait)**: 세마포어 값을 1 감소시키며, 0이 되면 대기
- **V연산(signal)**: 세마포어 값을 1 증가시키며, 대기 중인 프로세스 깨움
- **상호배제**: 임계구역에 동시에 하나의 프로세스만 접근 허용

### 동기화에서의 역할
세마포어는 다음과 같은 문제를 해결합니다:
- **경쟁 상태(Race Condition)** 방지
- **데드락(Deadlock)** 예방
- **자원의 안전한 공유** 보장

## 2. 리눅스 시스템에서의 세마포어 구현

### 2.1 주요 시스템 콜 분석

**근거**: 제공된 코드에서 사용된 시스템 콜들을 순차적으로 분석

#### semget() - 세마포어 집합 생성/접근
```c
semid = semget(IPC_PRIVATE, 1, IPC_CREAT | 0666)
```
- **IPC_PRIVATE**: 새로운 세마포어 집합 생성
- **1**: 세마포어 개수
- **IPC_CREAT | 0666**: 생성 플래그와 권한 설정

#### semctl() - 세마포어 제어
```c
semctl(semid, 0, SETVAL, arg)  // 초기값 설정
semctl(semid, 0, IPC_RMID, arg)  // 삭제
```
- **SETVAL**: 세마포어 값 설정 (초기값 1로 설정)
- **IPC_RMID**: 세마포어 집합 삭제

#### semop() - 세마포어 연산 수행
```c
semop(semid, &pbuf, 1)  // P연산
semop(semid, &vbuf, 1)  // V연산
```

### 2.2 핵심 구조체 분석

**근거**: 코드에서 정의된 구조체들의 멤버 변수와 용도

#### struct sembuf - 세마포어 연산 구조체
```c
struct sembuf pbuf;
pbuf.sem_num = 0;     // 세마포어 번호 (0번째)
pbuf.sem_op = -1;     // P연산 (-1)
pbuf.sem_flg = SEM_UNDO;  // 프로세스 종료 시 자동 복구
```

**SEM_UNDO 플래그의 중요성**:
- 프로세스가 비정상 종료되어도 세마포어 연산이 자동으로 되돌려짐
- 시스템 자원 누수 방지

#### union semun - 세마포어 제어 공용체
```c
union semun {
    int val;                    // SETVAL에서 사용
    struct semid_ds *buf;       // 상태 정보
    unsigned short int *array;  // 배열 값
} arg;
```

## 3. 실제 코드 구현 분석

### 3.1 P연산(감소) 구현
```c
void p() {
    struct sembuf pbuf;
    pbuf.sem_num = 0;
    pbuf.sem_op = -1;           // 세마포어 값 1 감소
    pbuf.sem_flg = SEM_UNDO;
    if(semop(semid, &pbuf, 1) == -1) {
        perror("p: semop()");
    }
}
```

**추론 과정**:
1. sem_op = -1로 세마포어 값을 감소시킴
2. 값이 0보다 작아지면 프로세스는 대기 상태로 전환
3. 자원 획득을 의미함

### 3.2 V연산(증가) 구현
```c
void v() {
    struct sembuf vbuf;
    vbuf.sem_num = 0;
    vbuf.sem_op = 1;            // 세마포어 값 1 증가
    vbuf.sem_flg = SEM_UNDO;
    if(semop(semid, &vbuf, 1) == -1) {
        perror("v : semop()");
    }
}
```

**추론 과정**:
1. sem_op = 1로 세마포어 값을 증가시킴
2. 대기 중인 프로세스가 있다면 깨워줌
3. 자원 해제를 의미함

### 3.3 메인 로직 분석

**근거**: while 루프의 조건문과 카운터 변수 cnt의 변화

```c
while(1) {
    if(cnt >= 8) {
        cnt--;
        p();                    // P연산 수행
        printf("decrease : %d\n", cnt);
        break;
    } else {
        cnt++;
        v();                    // V연산 수행
        printf("increase: %d\n", cnt);
        usleep(100);           // 100마이크로초 대기
    }
}
```

**동작 순서 추론**:
1. cnt를 0부터 8까지 증가시키며 V연산 수행
2. cnt가 8에 도달하면 P연산 수행 후 종료
3. 각 단계마다 세마포어 상태 출력

## 4. 코드 실행 결과와 해석

**예상 출력**:
```
increase: 1
increase: 2
increase: 3
increase: 4
increase: 5
increase: 6
increase: 7
increase: 8
decrease: 7
```

**해석**:
- 세마포어 초기값: 1
- V연산 8번 수행 후 세마포어 값: 9
- P연산 1번 수행 후 세마포어 값: 8
- 정상적인 P/V 연산 동작 확인

## 5. 실무 활용 팁

### 에러 처리의 중요성
**근거**: 모든 시스템 콜에서 반환값 검사 수행
```c
if(semget(...) == -1) {
    perror("semget()");
    return -1;
}
```

### 메모리 누수 방지
**근거**: 프로그램 종료 전 IPC_RMID로 세마포어 삭제
```c
if(semctl(semid, 0, IPC_RMID, arg) == -1) {
    perror("semctl(): IPC_RMID");
    return -1;
}
```

### 멀티프로세스 환경 고려사항
- **SEM_UNDO 플래그** 사용으로 안전성 보장
- **적절한 초기값 설정**으로 데드락 방지
- **에러 상황 대비** 복구 메커니즘 구현

## 결론

세마포어는 리눅스 시스템 프로그래밍에서 프로세스 간 동기화를 위한 강력한 도구입니다. 본 예제를 통해 P/V 연산의 기본 원리와 구현 방법을 학습했습니다. 

**핵심 포인트**:
- 세마포어는 공유 자원의 안전한 접근을 보장
- P연산은 자원 획득, V연산은 자원 해제를 의미
- 시스템 콜의 올바른 사용과 에러 처리가 중요
- SEM_UNDO 플래그로 시스템 안정성 확보

실제 멀티프로세스 환경에서는 더 복잡한 동기화 시나리오를 다뤄야 하지만, 이 기본 개념을 바탕으로 확장할 수 있습니다.