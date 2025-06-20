---
layout: post
title: "POSIX IPC 메시지큐 완전 가이드"
date: 2025-06-21 11:20:00 +0900
categories: [Linux, 시스템프로그래밍]
tags: [POSIX, IPC, Message Queue, mqueue, System Programming, C]
---

# POSIX IPC 메시지큐 완전 가이드

## 목차
1. [POSIX IPC의 탄생 배경](#posix-ipc의-탄생-배경)
2. [메시지큐란 무엇인가?](#메시지큐란-무엇인가)
3. [POSIX 메시지큐 vs System V 메시지큐](#posix-메시지큐-vs-system-v-메시지큐)
4. [핵심 구조체와 함수들](#핵심-구조체와-함수들)
5. [실제 구현 예제](#실제-구현-예제)
6. [메시지큐 동작 원리](#메시지큐-동작-원리)

## POSIX IPC의 탄생 배경

### System V IPC의 한계점
초기 Unix 시스템에서는 System V IPC(Inter-Process Communication)가 프로세스 간 통신의 주요 방법이었습니다. 하지만 몇 가지 문제점이 있었습니다:

- **복잡한 키 관리**: `ftok()` 함수를 통한 키 생성 방식의 복잡성
- **이식성 부족**: 시스템마다 다른 구현으로 인한 호환성 문제
- **제한된 실시간 지원**: 실시간 시스템 요구사항 미충족
- **네임스페이스 문제**: 전역 네임스페이스로 인한 충돌 가능성

### POSIX.1b 표준의 등장
1993년 IEEE에서 발표한 POSIX.1b(실시간 확장) 표준은 이러한 문제들을 해결하기 위해 새로운 IPC 메커니즘을 도입했습니다:

- **파일시스템 기반 네이밍**: `/name` 형태의 직관적인 이름 체계
- **표준화된 인터페이스**: 모든 POSIX 호환 시스템에서 동일한 동작
- **실시간 지원**: 우선순위 기반 메시지 처리
- **향상된 보안**: 파일 권한 모델 적용

## 메시지큐란 무엇인가?

메시지큐(Message Queue)는 **프로세스 간에 구조화된 데이터를 비동기적으로 주고받을 수 있는 IPC 메커니즘**입니다.

### 핵심 특징
- **비동기 통신**: 송신자와 수신자가 동시에 실행될 필요 없음
- **구조화된 메시지**: 임의의 크기와 타입의 데이터 전송 가능
- **우선순위 지원**: 메시지마다 우선순위 설정 가능
- **퍼시스턴트**: 프로세스 종료 후에도 커널에 남아있음

## POSIX 메시지큐 vs System V 메시지큐

| 특징 | POSIX MQ | System V MQ |
|------|----------|-------------|
| 네이밍 | `/name` (파일시스템 스타일) | 숫자 키 (ftok() 사용) |
| 헤더 | `<mqueue.h>` | `<sys/msg.h>` |
| 생성함수 | `mq_open()` | `msgget()` |
| 송신함수 | `mq_send()` | `msgsnd()` |
| 수신함수 | `mq_receive()` | `msgrcv()` |
| 우선순위 | 0-31 (숫자가 클수록 높음) | 타입 기반 |
| 비동기 알림 | 지원 (`mq_notify()`) | 미지원 |

## 핵심 구조체와 함수들

### mq_attr 구조체
```c
struct mq_attr {
    long mq_flags;     // 메시지큐 플래그 (O_NONBLOCK 등)
    long mq_maxmsg;    // 큐에 저장 가능한 최대 메시지 수
    long mq_msgsize;   // 각 메시지의 최대 크기 (바이트)
    long mq_curmsgs;   // 현재 큐에 있는 메시지 수 (읽기 전용)
};
```

### 주요 함수들

#### mq_open()
```c
mqd_t mq_open(const char *name, int oflag, mode_t mode, struct mq_attr *attr);
```
- **기능**: 메시지큐 생성 또는 기존 큐 열기
- **플래그**: 
  - `O_CREAT`: 큐가 없으면 생성
  - `O_EXCL`: 이미 존재하면 실패
  - `O_RDONLY`, `O_WRONLY`, `O_RDWR`: 접근 모드

#### mq_send()
```c
int mq_send(mqd_t mqdes, const char *msg_ptr, size_t msg_len, unsigned int msg_prio);
```
- **기능**: 메시지큐에 메시지 전송
- **우선순위**: 0-31 (31이 가장 높은 우선순위)

#### mq_receive()
```c
ssize_t mq_receive(mqd_t mqdes, char *msg_ptr, size_t msg_len, unsigned int *msg_prio);
```
- **기능**: 메시지큐에서 가장 높은 우선순위 메시지 수신
- **반환값**: 받은 메시지의 길이

#### mq_close() / mq_unlink()
```c
int mq_close(mqd_t mqdes);          // 디스크립터 닫기
int mq_unlink(const char *name);    // 메시지큐 삭제
```

## 실제 구현 예제

### 메시지 수신자 (Reader)
```c
#include<stdio.h>
#include<unistd.h>
#include<mqueue.h>

int main(int argc, char **argv){
    mqd_t mq;
    struct mq_attr attr;
    const char* name = "/posix_msq"; // 메시지 큐 이름
    char buf[BUFSIZ];
    int n;

    // 메시지 큐 속성 초기화
    attr.mq_flags = 0;           // 블로킹 모드
    attr.mq_maxmsg = 10;         // 최대 10개 메시지
    attr.mq_msgsize = BUFSIZ;    // 메시지 최대 크기
    attr.mq_curmsgs = 0;         // 현재 메시지 수 (자동 설정)

    // 메시지큐 생성 및 읽기 전용으로 열기
    mq = mq_open(name, O_CREAT | O_RDONLY, 0644, &attr);

    // 메시지 수신 루프
    while(1){
        n = mq_receive(mq, buf, sizeof(buf), NULL);
        switch(buf[0]){
            case 'q':  // 종료 신호
                goto END;
            default:
                write(1, buf, n);  // 표준출력으로 메시지 출력
                break;
        }
    }

    END:
    mq_close(mq);      // 디스크립터 닫기
    mq_unlink(name);   // 메시지큐 삭제

    return 0;
}
```

### 메시지 송신자 (Writer)
```c
#include<stdio.h>
#include<string.h>
#include<unistd.h>
#include<mqueue.h>

int main(int argc, char **argv){
    mqd_t mq;
    const char * name = "/posix_msq";
    char buf[BUFSIZ];

    // 기존 메시지큐를 쓰기 전용으로 열기
    mq = mq_open(name, O_WRONLY);

    // 첫 번째 메시지 전송
    strcpy(buf, "Hello, World\n");
    mq_send(mq, buf, strlen(buf), 0);  // 우선순위 0

    // 종료 신호 전송
    strcpy(buf, "q");
    mq_send(mq, buf, strlen(buf), 0);

    mq_close(mq);

    return 0;
}
```

## 메시지큐 동작 원리

<svg viewBox="0 0 800 600" xmlns="http://www.w3.org/2000/svg">
  <!-- 배경 -->
  <rect width="800" height="600" fill="#f8f9fa"/>
  
  <!-- 제목 -->
  <text x="400" y="30" text-anchor="middle" font-family="Arial, sans-serif" font-size="20" font-weight="bold" fill="#2c3e50">POSIX IPC 메시지큐 (Message Queue)</text>
  
  <!-- 커널 공간 박스 -->
  <rect x="200" y="80" width="400" height="300" fill="#e8f4f8" stroke="#3498db" stroke-width="2" rx="10"/>
  <text x="400" y="100" text-anchor="middle" font-family="Arial, sans-serif" font-size="14" font-weight="bold" fill="#2980b9">커널 공간 (Kernel Space)</text>
  
  <!-- 메시지큐 -->
  <rect x="250" y="130" width="300" height="200" fill="#ffffff" stroke="#34495e" stroke-width="2" rx="5"/>
  <text x="400" y="150" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" font-weight="bold" fill="#2c3e50">메시지큐 (/posix_msq)</text>
  
  <!-- 메시지들 -->
  <rect x="270" y="170" width="260" height="30" fill="#3498db" stroke="#2980b9" stroke-width="1" rx="3"/>
  <text x="400" y="190" text-anchor="middle" font-family="Arial, sans-serif" font-size="10" fill="white">"Hello, World" (우선순위: 0)</text>
  
  <rect x="270" y="210" width="260" height="30" fill="#e74c3c" stroke="#c0392b" stroke-width="1" rx="3"/>
  <text x="400" y="230" text-anchor="middle" font-family="Arial, sans-serif" font-size="10" fill="white">"q" (우선순위: 0)</text>
  
  <rect x="270" y="250" width="260" height="20" fill="#ecf0f1" stroke="#bdc3c7" stroke-width="1" rx="3" stroke-dasharray="5,5"/>
  <text x="400" y="265" text-anchor="middle" font-family="Arial, sans-serif" font-size="10" fill="#7f8c8d">...</text>
  
  <!-- 프로세스 A (송신자) -->
  <rect x="50" y="200" width="120" height="80" fill="#2ecc71" stroke="#27ae60" stroke-width="2" rx="5"/>
  <text x="110" y="220" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" font-weight="bold" fill="white">프로세스 A</text>
  <text x="110" y="240" text-anchor="middle" font-family="Arial, sans-serif" font-size="10" fill="white">(송신자)</text>
  <text x="110" y="260" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="white">mq_send()</text>
  
  <!-- 프로세스 B (수신자) -->
  <rect x="630" y="200" width="120" height="80" fill="#9b59b6" stroke="#8e44ad" stroke-width="2" rx="5"/>
  <text x="690" y="220" text-anchor="middle" font-family="Arial, sans-serif" font-size="12" font-weight="bold" fill="white">프로세스 B</text>
  <text x="690" y="240" text-anchor="middle" font-family="Arial, sans-serif" font-size="10" fill="white">(수신자)</text>
  <text x="690" y="260" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="white">mq_receive()</text>
  
  <!-- 화살표 (송신) -->
  <defs>
    <marker id="arrowhead-send" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#27ae60"/>
    </marker>
    <marker id="arrowhead-receive" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#8e44ad"/>
    </marker>
  </defs>
  
  <path d="M 170 240 Q 210 210 250 240" stroke="#27ae60" stroke-width="2" fill="none" marker-end="url(#arrowhead-send)"/>
  <text x="210" y="220" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#27ae60">메시지 전송</text>
  
  <!-- 화살표 (수신) -->
  <path d="M 550 240 Q 590 210 630 240" stroke="#8e44ad" stroke-width="2" fill="none" marker-end="url(#arrowhead-receive)"/>
  <text x="590" y="220" text-anchor="middle" font-family="Arial, sans-serif" font-size="9" fill="#8e44ad">메시지 수신</text>
  
  <!-- 주요 함수들 -->
  <rect x="50" y="420" width="700" height="150" fill="#ffffff" stroke="#34495e" stroke-width="1" rx="5"/>
  <text x="400" y="440" text-anchor="middle" font-family="Arial, sans-serif" font-size="14" font-weight="bold" fill="#2c3e50">주요 POSIX 메시지큐 함수</text>
  
  <!-- 함수 설명 -->
  <text x="70" y="465" font-family="Arial, sans-serif" font-size="11" fill="#2c3e50">• <tspan font-weight="bold">mq_open()</tspan>: 메시지큐 생성/열기</text>
  <text x="70" y="485" font-family="Arial, sans-serif" font-size="11" fill="#2c3e50">• <tspan font-weight="bold">mq_send()</tspan>: 메시지 전송 (우선순위 지정 가능)</text>
  <text x="70" y="505" font-family="Arial, sans-serif" font-size="11" fill="#2c3e50">• <tspan font-weight="bold">mq_receive()</tspan>: 메시지 수신 (높은 우선순위부터)</text>
  <text x="70" y="525" font-family="Arial, sans-serif" font-size="11" fill="#2c3e50">• <tspan font-weight="bold">mq_close()</tspan>: 메시지큐 닫기</text>
  <text x="70" y="545" font-family="Arial, sans-serif" font-size="11" fill="#2c3e50">• <tspan font-weight="bold">mq_unlink()</tspan>: 메시지큐 삭제</text>
  
  <!-- 특징 -->
  <text x="400" y="465" font-family="Arial, sans-serif" font-size="11" fill="#2c3e50">• <tspan font-weight="bold">우선순위 기반</tspan>: 높은 우선순위 먼저 처리</text>
  <text x="400" y="485" font-family="Arial, sans-serif" font-size="11" fill="#2c3e50">• <tspan font-weight="bold">비동기 통신</tspan>: 송신자와 수신자 독립적</text>
  <text x="400" y="505" font-family="Arial, sans-serif" font-size="11" fill="#2c3e50">• <tspan font-weight="bold">커널 관리</tspan>: 프로세스 종료 후에도 유지</text>
  <text x="400" y="525" font-family="Arial, sans-serif" font-size="11" fill="#2c3e50">• <tspan font-weight="bold">FIFO + 우선순위</tspan>: 동일 우선순위는 FIFO</text>
  <text x="400" y="545" font-family="Arial, sans-serif" font-size="11" fill="#2c3e50">• <tspan font-weight="bold">최대 크기 제한</tspan>: 시스템 설정으로 제한</text>
</svg>

### 동작 순서
1. **큐 생성**: Reader가 `mq_open()`으로 메시지큐 생성
2. **메시지 전송**: Writer가 `mq_send()`로 메시지 전송
3. **우선순위 정렬**: 커널이 우선순위에 따라 메시지 정렬
4. **메시지 수신**: Reader가 `mq_receive()`로 높은 우선순위 메시지부터 수신
5. **큐 정리**: `mq_unlink()`로 메시지큐 삭제

### 주요 특징 분석

**코드에서 확인할 수 있는 특징들:**

1. **mq_attr 구조체 사용**:
   - `mq_flags = 0`: 블로킹 모드로 설정
   - `mq_maxmsg = 10`: 최대 10개 메시지 저장 가능
   - `mq_msgsize = BUFSIZ`: 메시지 최대 크기 설정
   - `mq_curmsgs = 0`: 현재 메시지 수 (커널이 자동 관리)

2. **파일시스템 네이밍**:
   - `/posix_msq`: 파일시스템 스타일의 직관적인 이름
   - 모든 프로세스가 동일한 이름으로 접근

3. **우선순위 처리**:
   - 코드에서는 우선순위 0으로 설정 (낮은 우선순위)
   - 동일 우선순위 내에서는 FIFO 방식으로 처리

### 주의사항
- **링크 라이브러리**: 컴파일 시 `-lrt` 옵션 필요
- **권한 설정**: 메시지큐 생성 시 적절한 권한 설정 중요
- **에러 처리**: 모든 함수 호출 후 반환값 확인 필수
- **리소스 정리**: 사용 후 반드시 `mq_close()`와 `mq_unlink()` 호출

## 컴파일 및 실행
```bash
# 컴파일 (실시간 라이브러리 링크 필요)
gcc -o reader reader.c -lrt
gcc -o writer writer.c -lrt

# 실행 (reader를 먼저 실행)
./reader &
./writer
```

POSIX 메시지큐는 System V IPC의 복잡성을 해결하고 실시간 시스템의 요구사항을 만족하는 강력한 IPC 메커니즘입니다. 파일시스템 기반의 직관적인 인터페이스와 우선순위 지원으로 현대적인 프로세스 간 통신에 적합한 선택입니다.