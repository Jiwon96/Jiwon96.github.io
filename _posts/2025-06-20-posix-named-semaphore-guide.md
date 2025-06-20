---
layout: post
title: "POSIX 네임드 세마포어 (Named Semaphore) 완벽 가이드"
date: 2025-06-20 23:59:00 +0900
categories: [Linux, 시스템프로그래밍]
tags: [semaphore, posix, ipc, 동기화, 시스템프로그래밍]
---

## 개요

네임드 세마포어(Named Semaphore)는 POSIX IPC 메커니즘 중 하나로, 프로세스 간 동기화를 위해 사용되는 중요한 도구입니다. 이름을 가진 세마포어로 여러 프로세스가 공유할 수 있으며, 파일 시스템에 영구적으로 존재할 수 있습니다.

## 네임드 세마포어란?

### 정의와 특징
- **네임드 세마포어**: 이름을 가진 세마포어로 프로세스 간에 공유 가능
- **영구성**: 프로세스가 종료되어도 명시적으로 제거하지 않으면 시스템에 남아있음
- **접근성**: 같은 이름을 사용하는 모든 프로세스가 접근 가능

### 익명 세마포어와의 차이점
| 구분 | 네임드 세마포어 | 익명 세마포어 |
|------|----------------|---------------|
| 이름 | 파일 시스템 이름 보유 | 이름 없음 |
| 공유 범위 | 모든 프로세스 | 관련 프로세스만 |
| 영구성 | 명시적 제거 전까지 유지 | 프로세스 종료 시 자동 제거 |

## 주요 함수 분석

### 1. sem_open() - 세마포어 열기/생성/초기화

```c
sem_t *sem_open(const char *name, int oflag, mode_t mode, unsigned int value);
```

**근거**: POSIX.1-2001 표준에 정의된 함수

**역할**: 네임드 세마포어는 사용 전에 반드시 초기화가 필요하며, `sem_open()`이 이 역할을 담당합니다.

**매개변수 분석**:
- `name`: 세마포어 이름 (일반적으로 "/이름" 형태)
- `oflag`: 플래그 옵션
  - `O_CREAT`: 세마포어가 없으면 생성
  - `O_EXCL`: O_CREAT와 함께 사용 시 이미 존재하면 실패
- `mode`: 권한 설정 (S_IRUSR | S_IWUSR = 소유자 읽기/쓰기)
- `value`: 초기 세마포어 값

**코드에서의 사용**:
```c
sem = sem_open(name, O_CREAT, S_IRUSR | S_IWUSR, value);
```

### 2. sem_wait() - 세마포어 획득 (P 연산)

```c
int sem_wait(sem_t *sem);
```

**동작 원리**:
1. 세마포어 값이 0보다 크면 1 감소 후 진행
2. 세마포어 값이 0이면 블록되어 대기
3. 성공 시 0 반환, 실패 시 -1 반환

### 3. sem_post() - 세마포어 해제 (V 연산)

```c
int sem_post(sem_t *sem);
```

**동작 원리**:
1. 세마포어 값을 1 증가
2. 대기 중인 프로세스가 있으면 깨움
3. 성공 시 0 반환, 실패 시 -1 반환

### 4. sem_close() - 세마포어 닫기

```c
int sem_close(sem_t *sem);
```

**근거**: 프로세스가 세마포어 사용을 완료했음을 시스템에 알림
- 세마포어 자체는 삭제되지 않음
- 단순히 현재 프로세스와의 연결만 해제

### 5. sem_unlink() - 세마포어 제거

```c
int sem_unlink(const char *name);
```

**근거**: 파일 시스템에서 세마포어를 완전히 제거
- 마지막 참조가 해제될 때 실제로 삭제됨
- 이후 같은 이름으로 새 세마포어 생성 가능

### sem_destroy() 사용 권장사항

네임드 세마포어에서는 `sem_close()`와 `sem_unlink()`가 있어서 `sem_destroy()`를 쓸 수는 있지만, 환경에 따라 에러가 날 수 있어 안 쓰는 게 좋을 것 같습니다.

## 코드의 P/V 함수 분석

### 제공된 코드의 P/V 함수:
```c
void p(){
    cnt--;
    sem_post(sem);
}

void v(){
    cnt++;
    sem_wait(sem);
}
```

**근거**: 전통적인 P/V 연산의 정의와 다름
- **P 연산**: 세마포어 값 감소 (wait)
- **V 연산**: 세마포어 값 증가 (post)

**올바른 구현**:
```c
void p(){
    sem_wait(sem);   // 먼저 세마포어 대기
    cnt--;           // 그 다음 카운터 감소
}

void v(){
    cnt++;           // 먼저 카운터 증가
    sem_post(sem);   // 그 다음 세마포어 해제
}
```

## 실행 흐름 분석

### 추론 과정:
1. **초기 상태**: 세마포어 값 = 8, cnt = 0
2. **루프 진입**: cnt < 8이므로 v() 함수 호출
3. **v() 실행**: cnt 증가 후 sem_wait() 호출
4. **sem_wait() 효과**: 세마포어 값 감소 (8→7→6...)
5. **종료 조건**: cnt >= 8이 되면 p() 호출 후 종료

### 예상 출력:
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

## 실제 사용 예제

### 올바른 네임드 세마포어 구현:

```c
#include <fcntl.h>
#include <sys/stat.h>
#include <semaphore.h>
#include <stdio.h>
#include <unistd.h>

sem_t *sem;
static int cnt = 0;

void p(){
    sem_wait(sem);   // 먼저 세마포어 대기
    cnt--;           // 그 다음 카운터 감소
}

void v(){
    cnt++;           // 먼저 카운터 증가
    sem_post(sem);   // 그 다음 세마포어 해제
}

int main() {
    const char *name = "/posix_sem";  // '/' 접두사 권장
    unsigned int value = 8;           // 세마포어 초기값
    
    // 세마포어 생성/초기화
    sem = sem_open(name, O_CREAT, S_IRUSR | S_IWUSR, value);
    if (sem == SEM_FAILED) {
        perror("sem_open");
        return 1;
    }
    
    while(1){
        if(cnt >= 8){
            p();
            printf("decrease: %d\n", cnt);
            break;
        }
        else{
            v();
            printf("increase: %d\n", cnt);
            usleep(100);
        }
    }
    
    // 정리
    sem_close(sem);
    sem_unlink(name);
    
    return 0;
}
```

## 결론

네임드 세마포어는 프로세스 간 동기화를 위한 강력한 도구이지만, 올바른 사용법을 이해하고 적용하는 것이 중요합니다. 특히 P/V 연산의 순서와 네임드 세마포어 전용 함수들을 정확히 사용해야 합니다.

## 참고자료

- POSIX.1-2001 표준 문서
- Linux Manual Pages (man 3 sem_open)
- Advanced Programming in the UNIX Environment