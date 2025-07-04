---
title: "Linux 멀티스레딩과 동기화: 개념부터 실제까지"
date: 2025-06-21 01:00:00 +0900
categories: [Linux, 시스템프로그래밍]
tags: [pthread, mutex, signal, 멀티스레딩, 동기화, 프로세스]
---

# Linux 멀티스레딩과 동기화: 개념부터 실제까지

## 멀티스레딩의 기본 개념

### 1. 프로세스 vs 스레드

**프로세스**는 실행 중인 프로그램의 인스턴스로, 독립된 메모리 공간을 가집니다. 반면 **스레드**는 프로세스 내에서 실행되는 경량화된 실행 단위로, 같은 프로세스의 스레드들은 메모리 공간을 공유합니다.

- **프로세스**: 독립된 주소 공간, 높은 생성 비용, IPC를 통한 통신
- **스레드**: 공유 주소 공간, 낮은 생성 비용, 직접적인 메모리 공유

### 2. 스레드의 메모리 구조

Linux에서 스레드는 다음과 같은 메모리 영역을 공유하고 분리합니다:

**공유 영역**:
- 코드 세그먼트 (Text Segment)
- 데이터 세그먼트 (Data Segment)
- 힙 (Heap)
- 열린 파일 디스크립터

**분리 영역**:
- 스택 (Stack)
- 레지스터 세트
- 프로그램 카운터 (PC)

## 동기화의 필요성과 문제점

### 1. Race Condition (경쟁 상태)

여러 스레드가 동시에 공유 자원에 접근할 때 실행 순서에 따라 결과가 달라지는 현상입니다.

```c
// 예제: 전역 변수 g_var에 대한 경쟁 상태
int g_var = 1;

// Thread 1: g_var++
// Thread 2: g_var--
// 결과: 실행 순서에 따라 0, 1, 2 중 하나
```

### 2. Critical Section (임계 구역)

**임계 구역**은 공유 자원(변수, 파일, 하드웨어 등)에 접근하는 프로그램 코드 영역입니다. 멀티스레딩 환경에서 가장 중요한 개념 중 하나로, 동시에 여러 스레드가 실행되면 데이터 불일치나 시스템 오류가 발생할 수 있는 코드 구간을 의미합니다.

#### 임계 구역의 구조

일반적으로 임계 구역은 다음과 같은 구조를 가집니다:

```c
// Entry Section (진입 구역)
pthread_mutex_lock(&mutex);

// Critical Section (임계 구역)
shared_variable++;  // 공유 자원 접근

// Exit Section (탈출 구역)  
pthread_mutex_unlock(&mutex);

// Remainder Section (나머지 구역)
```

#### 임계 구역 문제 해결의 필수 조건

**1. 상호 배제 (Mutual Exclusion)**
- 한 번에 최대 하나의 스레드만 임계 구역에서 실행
- 다른 스레드들은 대기해야 함

**2. 진행 (Progress)**
- 임계 구역에 진입하려는 스레드가 있을 때, 진입할 스레드의 선택은 유한 시간 내에 이루어져야 함
- 임계 구역에 아무도 없다면 진입을 원하는 스레드는 즉시 진입 가능해야 함

**3. 한정 대기 (Bounded Waiting)**
- 스레드가 임계 구역 진입을 요청한 후, 다른 스레드들이 임계 구역에 진입하는 횟수에는 한계가 있어야 함
- 무한정 대기(starvation) 방지

#### 임계 구역 문제의 실제 예시

```c
// 잘못된 예: 임계 구역 보호 없음
int counter = 0;

void* increment_thread(void* arg) {
    for(int i = 0; i < 1000; i++) {
        counter++;  // 임계 구역이지만 보호되지 않음
    }
    return NULL;
}

// 올바른 예: 뮤텍스로 임계 구역 보호
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
int counter = 0;

void* safe_increment_thread(void* arg) {
    for(int i = 0; i < 1000; i++) {
        pthread_mutex_lock(&mutex);    // 진입 구역
        counter++;                     // 임계 구역
        pthread_mutex_unlock(&mutex);  // 탈출 구역
    }
    return NULL;
}
```

#### 임계 구역 식별 방법

**공유 자원 접근 패턴**:
- 전역 변수 읽기/쓰기
- 동적 메모리 할당/해제
- 파일 I/O 작업
- 하드웨어 자원 접근
- 공유 자료구조 조작

**임계 구역이 아닌 경우**:
- 지역 변수만 사용하는 코드
- 읽기 전용 데이터 접근 (상수)
- 스레드별 독립적인 자원 사용

## POSIX 스레드 (pthread) 개념

### 1. 스레드 생명주기

```
생성 → 준비 → 실행 → 대기/블록 → 종료
```

- **생성**: `pthread_create()`로 새 스레드 생성
- **준비**: 스케줄러에 의해 실행 대기
- **실행**: CPU를 할당받아 실행
- **대기**: I/O, 동기화 객체 등으로 인한 블록
- **종료**: `pthread_exit()` 호출 또는 함수 반환

### 2. 스레드 식별과 관리

**pthread_t**: 스레드 식별자
- 불투명한 타입으로 내부 구조는 구현체마다 다름
- 일반적으로 스레드 제어 블록(TCB)에 대한 포인터나 고유 ID

**주요 함수들의 개념**:

#### pthread_create()
- **개념**: 새로운 실행 흐름 생성
- **커널 동작**: clone() 시스템 콜을 통해 새 태스크 생성
- **메모리**: 새 스택 영역 할당, 나머지는 부모와 공유

#### pthread_join()
- **개념**: 스레드 종료 동기화
- **커널 동작**: 대상 스레드가 종료될 때까지 호출 스레드를 sleep 상태로 전환
- **자원 회수**: 종료된 스레드의 자원을 회수

#### pthread_exit()
- **개념**: 현재 스레드 종료
- **특징**: main()에서 호출 시 프로세스는 종료되지 않고 다른 스레드들 계속 실행

#### pthread_cancel()
- **개념**: 다른 스레드에게 종료 요청
- **동작**: 즉시 종료가 아닌, 취소 포인트에서 종료 처리

#### pthread_self()
- **개념**: 현재 스레드의 식별자 반환
- **용도**: 스레드가 자신을 식별해야 할 때 사용

## 뮤텍스 (Mutex) 개념과 동작

### 1. 뮤텍스의 개념

**Mutex (Mutual Exclusion)**는 상호 배제를 구현하는 동기화 객체입니다.

**동작 원리**:
- 이진 세마포어와 유사하지만 소유권 개념 존재
- 락을 획득한 스레드만이 언락 가능
- 우선순위 상속 (Priority Inheritance) 지원

### 2. 뮤텍스 상태와 전이

```
UNLOCKED → LOCKED → UNLOCKED
    ↑         ↓
  unlock    lock
```

### 3. 뮤텍스 함수들의 개념

#### pthread_mutex_init() / pthread_mutex_destroy()
- **개념**: 뮤텍스 생명주기 관리
- **초기화**: 뮤텍스를 UNLOCKED 상태로 설정
- **해제**: 시스템 자원 반환

#### pthread_mutex_lock()
- **개념**: 블로킹 락 획득
- **동작**: 락이 사용 가능할 때까지 스레드를 블록

#### pthread_mutex_trylock()
- **개념**: 논블로킹 락 시도
- **장점**: 데드락 회피, 성능 최적화 가능

#### pthread_mutex_unlock()
- **개념**: 락 해제 및 대기 스레드 깨우기
- **스케줄링**: 대기 중인 스레드 중 하나를 실행 가능 상태로 전환

#### pthread_mutex_timedlock()
- **개념**: 시간 제한이 있는 락 획득
- **용도**: 무한 대기 방지, 반응성 향상

## Reader-Writer Lock 개념

### 1. 읽기-쓰기 락의 필요성

데이터를 주로 읽는 작업이 많은 경우, 여러 읽기 작업은 동시에 수행해도 안전합니다.

**동시성 규칙**:
- 여러 읽기 작업: 동시 허용
- 하나의 쓰기 작업: 다른 모든 작업과 상호 배제
- 읽기와 쓰기: 상호 배제

### 2. RWLock 함수들의 개념

#### pthread_rwlock_rdlock()
- **개념**: 공유 읽기 락 획득
- **동작**: 다른 읽기 락과 동시 획득 가능

#### pthread_rwlock_wrlock()
- **개념**: 독점 쓰기 락 획득
- **동작**: 모든 다른 락과 상호 배제

#### pthread_rwlock_unlock()
- **개념**: 락 타입에 관계없이 해제
- **스케줄링**: 대기 중인 스레드들 중 적절한 스레드 선택

## 스레드와 시그널

### 1. 시그널의 기본 개념

**시그널**은 Unix/Linux에서 프로세스 간 비동기 통신을 위한 메커니즘입니다.

**시그널의 특성**:
- 비동기적 이벤트 알림
- 소프트웨어 인터럽트
- 프로세스 제어 및 상태 통지

### 2. 프로세스 vs 스레드: 시그널 처리의 공통점과 차이점

#### 공통점

**1. 시그널 기본 메커니즘**
- 동일한 시그널 번호와 의미 사용 (SIGINT, SIGTERM, SIGSEGV 등)
- signal() 및 sigaction() 함수로 핸들러 등록 가능
- 시그널 마스킹 메커니즘 지원 (SIG_BLOCK, SIG_UNBLOCK, SIG_SETMASK)
- 동기적/비동기적 시그널 분류 동일

**2. 시그널 전달 방식**
- 커널에 의한 시그널 생성 및 전달
- 시그널 대기열(pending signal queue) 관리
- 실시간 시그널과 표준 시그널 구분

#### 차이점

| 항목 | 프로세스 | 스레드 |
|------|----------|--------|
| **시그널 대상** | 전체 프로세스 | 개별 스레드 또는 프로세스 |
| **마스크 상속** | fork()시 자식이 상속 | 새 스레드가 생성 스레드 마스크 상속 |
| **핸들러 공유** | - | 모든 스레드가 동일한 핸들러 공유 |
| **전달 함수** | kill() 사용 | pthread_kill() 사용 |
| **마스크 설정** | sigprocmask() 사용 | pthread_sigmask() 사용 |

#### 멀티스레딩 환경에서의 시그널 복잡성

**1. 시그널 전달 대상 결정**
```c
// 프로세스 전체에 시그널 전송
kill(pid, SIGTERM);

// 특정 스레드에 시그널 전송  
pthread_kill(thread_id, SIGUSR1);
```

**2. 시그널 처리 스레드 선택 규칙**
- **동기적 시그널**: 시그널을 발생시킨 스레드에서 처리
- **비동기적 시그널**: 해당 시그널을 차단하지 않은 임의의 스레드에서 처리
- **모든 스레드가 차단**: 시그널이 pending 상태로 대기

**3. 시그널 마스크 관리**
```c
// 잘못된 예: 멀티스레드에서 sigprocmask() 사용
sigprocmask(SIG_BLOCK, &set, NULL);  // 동작 정의되지 않음

// 올바른 예: pthread_sigmask() 사용
pthread_sigmask(SIG_BLOCK, &set, NULL);  // 현재 스레드만 적용
```

#### 시그널 처리 방식의 차이

**1. 동기적 시그널 (SIGSEGV, SIGFPE, SIGILL 등)**
- **프로세스**: 오류 발생 시 전체 프로세스 종료
- **스레드**: 오류 발생 스레드에서 처리, 다른 스레드는 계속 실행

**2. 비동기적 시그널 (SIGINT, SIGTERM, SIGUSR1 등)**
- **프로세스**: 전체 프로세스에 영향
- **스레드**: 시그널을 차단하지 않은 임의의 스레드가 처리

### 3. 스레드 시그널 함수들

#### pthread_sigmask()
```c
int pthread_sigmask(int how, const sigset_t *set, sigset_t *oldset);
```
- **개념**: 스레드별 시그널 마스크 설정
- **용도**: 특정 스레드에서 시그널 차단

#### pthread_kill()
```c
int pthread_kill(pthread_t thread, int sig);
```
- **개념**: 특정 스레드에게 시그널 전송
- **차이점**: kill()은 프로세스 전체, pthread_kill()은 특정 스레드

#### sigwait()
```c
int sigwait(const sigset_t *set, int *sig);
```
- **개념**: 지정된 시그널을 동기적으로 대기
- **패턴**: 전용 시그널 처리 스레드 구현에 사용

### 4. 시그널 처리 설계 패턴

#### 전용 시그널 스레드 패턴
```c
// 모든 스레드에서 시그널 차단
pthread_sigmask(SIG_BLOCK, &set, NULL);

// 전용 스레드에서만 시그널 처리
void* signal_thread(void* arg) {
    int sig;
    while (1) {
        sigwait(&set, &sig);
        // 시그널 처리
    }
}
```

## 동기화 설계 원칙

### 1. 데드락 방지
- **락 순서 일관성**: 모든 스레드가 동일한 순서로 락 획득
- **타임아웃 사용**: `pthread_mutex_timedlock()` 활용
- **락 계층 구조**: 상위 레벨에서 하위 레벨 순으로 락 획득

### 2. 성능 최적화
- **임계 구역 최소화**: 락 보유 시간 단축
- **적절한 동기화 기법 선택**: 읽기 위주 → RWLock, 단순 보호 → Mutex
- **락 경합 분산**: 여러 개의 세밀한 락 사용

### 3. 올바른 시그널 처리
- **시그널 마스킹**: 대부분의 스레드에서 시그널 차단
- **전용 처리 스레드**: 시그널 처리를 위한 별도 스레드
- **시그널 안전성**: 시그널 핸들러에서는 async-signal-safe 함수만 사용

## 결론

Linux 멀티스레딩 환경에서는 스레드 간 공유 자원에 대한 안전한 접근과 효율적인 동기화가 핵심입니다. 뮤텍스, 읽기-쓰기 락 등의 동기화 기법과 시그널 처리 메커니즘을 올바르게 이해하고 적용하는 것이 안정적이고 성능이 우수한 멀티스레드 애플리케이션 개발의 기초가 됩니다. 특히 각 동기화 기법의 특성을 파악하고 상황에 맞는 적절한 선택을 하는 것이 중요합니다.