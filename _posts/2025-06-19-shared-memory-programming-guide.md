---
layout: post
title: "리눅스 공유메모리(Shared Memory) 프로그래밍 가이드"
date: 2025-06-19 23:00:00 +0900
categories: [Linux, 시스템프로그래밍]
tags: [공유메모리, IPC, shmget, shmat, 세마포어, 리눅스시스템프로그래밍]
---

## 서론

프로세스 간 통신(IPC, Inter-Process Communication)은 리눅스 시스템 프로그래밍에서 핵심적인 개념입니다. 그 중에서도 공유메모리(Shared Memory)는 가장 빠른 IPC 메커니즘 중 하나로, 여러 프로세스가 동일한 메모리 영역에 접근할 수 있게 해줍니다.

## 공유메모리란?

공유메모리는 두 개 이상의 프로세스가 동일한 물리적 메모리 공간을 공유하여 데이터를 주고받는 IPC 방법입니다. 일반적으로 각 프로세스는 독립된 가상 주소 공간을 가지지만, 공유메모리를 통해 특정 메모리 영역을 여러 프로세스가 공동으로 사용할 수 있습니다.

### 공유메모리의 장점

1. **빠른 속도**: 커널을 거치지 않고 직접 메모리에 접근하므로 가장 빠른 IPC 방법
2. **메모리 복사 최소화**: 파이프나 메시지 큐와 달리 데이터를 커널 버퍼로 복사할 필요가 없어 메모리 복사 횟수를 대폭 줄여 성능 향상
3. **효율성**: 데이터 복사 없이 직접 공유하므로 메모리 사용량 절약
4. **대용량 데이터**: 큰 데이터 구조체나 배열을 효율적으로 공유 가능

### 공유메모리의 단점

1. **동기화 문제**: 여러 프로세스가 동시에 접근할 때 데이터 일관성 문제 발생
2. **복잡성**: 동기화 메커니즘(세마포어, 뮤텍스 등) 추가 구현 필요

## 공유메모리 관련 시스템 콜

### 1. shmget() - 공유메모리 세그먼트 생성/접근

```c
int shmget(key_t key, size_t size, int shmflg);
```

**매개변수 분석**:
- `key`: 공유메모리를 식별하는 키 (예: 0x12345)
- `size`: 공유메모리 크기 (바이트 단위)
- `shmflg`: 플래그 (IPC_CREAT, 권한 등)

**반환값**: 성공 시 공유메모리 식별자(shmid), 실패 시 -1

### 2. shmat() - 공유메모리 연결

```c
void *shmat(int shmid, const void *shmaddr, int shmflg);
```

**매개변수 분석**:
- `shmid`: shmget()에서 반환받은 식별자
- `shmaddr`: 연결할 주소 (보통 0으로 설정하여 시스템이 자동 선택)
- `shmflg`: 플래그 (읽기 전용, 읽기/쓰기 등)

### 3. shmctl() - 공유메모리 제어

```c
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
```

**주요 명령어**:
- `IPC_RMID`: 공유메모리 삭제
- `IPC_STAT`: 공유메모리 정보 조회
- `IPC_SET`: 공유메모리 속성 설정

## shmid_ds 자료구조

공유메모리의 메타데이터를 저장하는 구조체입니다:

```c
struct shmid_ds {
    struct ipc_perm shm_perm;        /* 권한 정보 */
    size_t          shm_segsz;       /* 세그먼트 크기 (바이트) */
    __time_t        shm_atime;       /* 마지막 shmat() 호출 시간 */
    unsigned long   __glibc_reserved1;
    __time_t        shm_dtime;       /* 마지막 shmdt() 호출 시간 */
    unsigned long   __glibc_reserved2;
    __time_t        shm_ctime;       /* 마지막 shmctl() 호출 시간 */
    unsigned long   __glibc_reserved3;
    __pid_t         shm_cpid;        /* 생성 프로세스 ID */
    __pid_t         shm_lpid;        /* 마지막 shmat/shmdt 프로세스 ID */
    shmatt_t        shm_nattch;      /* 현재 연결된 프로세스 수 */
    unsigned long   __glibc_reserved4;
    unsigned long   __glibc_reserved5;
};
```

**주요 필드 설명**:
- `shm_perm`: IPC 권한 구조체 (소유자, 그룹, 권한 비트 등)
- `shm_segsz`: 공유메모리 세그먼트의 바이트 크기
- `shm_atime`: 마지막으로 프로세스가 shmat()을 호출한 시간
- `shm_dtime`: 마지막으로 프로세스가 shmdt()를 호출한 시간 
- `shm_ctime`: 마지막으로 shmctl()로 변경된 시간
- `shm_cpid`: 공유메모리 세그먼트를 생성한 프로세스 ID
- `shm_lpid`: 마지막으로 shmat() 또는 shmdt()를 호출한 프로세스 ID
- `shm_nattch`: 현재 이 세그먼트에 연결된 프로세스 개수
- `__glibc_reserved1~5`: glibc에서 미래 확장을 위해 예약된 필드들

## 실제 코드 분석

제공된 코드를 단계별로 분석해보겠습니다:

```c
#include<stdio.h>
#include<unistd.h>
#include<sys/shm.h>

#define SHM_KEY 0x12345

int main(int argc, char **argv){
    int i, pid, shmid;
    int *cVal;
    void *shmmem = (void *) 0;

    if((pid = fork()) == 0){
        // === 자식 프로세스 ===
        // 공유 메모리 공간 가져옴
        shmid = shmget((key_t)SHM_KEY, sizeof(int), 0); 
        if(shmid == -1){
            perror("shmget()");
            return -1;
        }
        
        // 공유메모리를 프로세스 주소 공간에 연결
        shmmem = shmat(shmid, (void *)0, 0666|IPC_CREAT);
        if(shmmem == (void *) -1){
            perror("shmmat()");
            return -1;
        }

        cVal = (int *)shmmem;
        *cVal = 1;  // 초기값 설정
        
        // 3번 반복하며 값 증가
        for(i=0; i<3; i++){
            *cVal +=1;
            printf("Child(%d): %d\n", i, *cVal);
            sleep(1);
        }
    }else if(pid >0){
        // === 부모 프로세스 ===
        // 공유메모리 생성 또는 기존 것 가져옴
        shmid = shmget((key_t)SHM_KEY, sizeof(int), 0666|IPC_CREAT);
        if(shmid == -1){
            perror("shmget()");
            return -1;
        }

        // 공유메모리 연결
        shmmem = shmat(shmid, (void*)0, 0);
        if(shmmem == (void*)-1){
            perror("shmat");
            return -1;
        }
        
        cVal = (int *) shmmem;
        
        // 3번 반복하며 값 읽기
        for(i=0; i<3; i++){
            sleep(1);
            printf("Parent(%d): %d\n", i, *cVal);
        }
    }

    // 공유메모리 삭제
    shmctl(shmid, IPC_RMID, 0);
    
    return 0;
}
```

### 코드 실행 흐름 분석

1. **fork()** 호출로 부모-자식 프로세스 생성
2. **자식 프로세스**: 공유메모리에 값을 쓰며 1씩 증가
3. **부모 프로세스**: 공유메모리에서 값을 읽어 출력
4. 두 프로세스가 동일한 메모리 영역을 공유하여 데이터 교환

### 예상 출력 결과
```
Child(0): 2
Parent(0): 2
Child(1): 3
Parent(1): 3
Child(2): 4
Parent(2): 4
```

## 동기화 문제와 세마포어의 필요성

### 현재 코드의 문제점

제공된 코드에서는 자식 프로세스가 값을 쓰고, 부모 프로세스가 값을 읽는 단순한 구조입니다. 하지만 다음과 같은 동기화 문제가 발생할 수 있습니다:

1. **Race Condition**: 두 프로세스가 동시에 같은 메모리 위치에 접근할 때
2. **데이터 불일치**: 한 프로세스가 쓰는 도중 다른 프로세스가 읽을 때
3. **타이밍 문제**: sleep()만으로는 완벽한 동기화 보장 불가

### 세마포어를 이용한 해결책

세마포어(Semaphore)는 공유자원에 대한 접근을 제어하는 동기화 메커니즘입니다:

```c
#include <sys/sem.h>

// 세마포어 생성
int semget(key_t key, int nsems, int semflg);

// 세마포어 연산 (P, V 연산)
int semop(int semid, struct sembuf *sops, size_t nsops);

// 세마포어 제어
int semctl(int semid, int semnum, int cmd, ...);
```

**세마포어 활용 예시**:
```c
// P 연산 (세마포어 감소, 임계영역 진입)
struct sembuf p_op = {0, -1, SEM_UNDO};
semop(semid, &p_op, 1);

// 임계영역 - 공유메모리 접근
*shared_data = new_value;

// V 연산 (세마포어 증가, 임계영역 탈출)
struct sembuf v_op = {0, +1, SEM_UNDO};
semop(semid, &v_op, 1);
```

## 실습 및 컴파일

### 컴파일 방법
```bash
gcc -o shared_memory shared_memory.c
./shared_memory
```

### 공유메모리 상태 확인
```bash
# 현재 공유메모리 목록 확인
ipcs -m

# 특정 공유메모리 삭제
ipcrm -m [shmid]
```

## 결론

공유메모리는 프로세스 간 고속 데이터 교환을 위한 강력한 도구입니다. 하지만 동시성 제어를 위해 세마포어나 뮤텍스 같은 동기화 메커니즘과 함께 사용해야 안정적인 프로그램을 작성할 수 있습니다.

## 참고자료

- Advanced Programming in the UNIX Environment
- Linux System Programming
- man pages: shmget(2), shmat(2), shmctl(2)