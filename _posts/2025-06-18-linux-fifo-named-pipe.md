---
layout: post
title: "Linux FIFO(Named Pipe)를 활용한 프로세스 간 통신"
date: 2025-06-17 22:30:00 +0900
categories: [Linux, 시스템프로그래밍]
tags: [FIFO, Named Pipe, IPC, 시스템프로그래밍, 프로세스간통신]
---

# Linux FIFO(Named Pipe)를 활용한 프로세스 간 통신

## FIFO란?

**근거**: FIFO(First In First Out)는 Named Pipe라고도 불리며, 파일시스템에 특수 파일로 존재하는 단방향 통신 메커니즘입니다.

**일반 파이프의 한계점**:
- `pipe()` 시스템 콜로 생성되는 익명 파이프는 부모-자식 프로세스 간에만 통신 가능
- 서로 관련 없는 프로세스들은 통신할 수 없음

**FIFO의 해결책**:
- 파일시스템에 이름을 가진 파일로 존재
- 같은 FIFO 파일명을 알고 있는 모든 프로세스가 통신 가능
- IPC(Inter-Process Communication) 조인 문제 해결

## 일반 파이프 vs FIFO 비교

| 구분 | 일반 파이프 | FIFO (Named Pipe) |
|------|-------------|-------------------|
| 생성 방법 | `pipe()` 시스템 콜 | `mkfifo()` 시스템 콜 |
| 파일시스템 존재 | X (메모리상에만) | O (실제 파일로 존재) |
| 통신 범위 | 부모-자식 프로세스만 | 모든 프로세스 |
| 식별 방법 | 파일 디스크립터 | 파일명 |

## FIFO 통신 플로우

```
[서버 프로세스]                    [클라이언트 프로세스]
      |                                    |
   1. unlink(FIFOFILE)                     |
      |                                    |
   2. mkfifo(FIFOFILE, 0666)               |
      |                                    |
   3. open(FIFOFILE, O_RDONLY) ---------> 4. open(FIFOFILE, O_WRONLY)
      |                                    |
   5. read(fd, buf, sizeof(buf)) <------- 6. write(fd, buf, n)
      |                                    |
   7. printf("%s", buf)                    |
      |                                    |
   8. close(fd)                        9. close(fd)
      |                                    |
```

## 주요 시스템 콜 함수 설명

### 1. mkfifo()
```c
int mkfifo(const char *pathname, mode_t mode);
```
- **기능**: FIFO 특수 파일을 생성
- **매개변수**: 
  - `pathname`: 생성할 FIFO 파일 경로
  - `mode`: 파일 권한 (예: 0666)
- **반환값**: 성공 시 0, 실패 시 -1

### 2. open() - FIFO 열기 플래그들
```c
int open(const char *pathname, int flags);
```
- **기능**: FIFO 파일을 열어 파일 디스크립터 반환
- **매개변수**:
  - `pathname`: FIFO 파일 경로
  - `flags`: 열기 모드와 추가 플래그들
- **반환값**: 성공 시 파일 디스크립터, 실패 시 -1

#### 주요 FIFO 열기 플래그들

**기본 모드 플래그**:
- `O_RDONLY`: 읽기 전용으로 열기
- `O_WRONLY`: 쓰기 전용으로 열기  
- `O_RDWR`: 읽기/쓰기 모두 가능하게 열기

**추가 동작 제어 플래그**:

**O_NDELAY (또는 O_NONBLOCK)**:
```c
fd = open(FIFOFILE, O_RDONLY | O_NDELAY);
fd = open(FIFOFILE, O_WRONLY | O_NONBLOCK);
```
- **기능**: 비블로킹 모드로 FIFO 열기
- **블로킹 모드 문제점**:
  - `O_RDONLY`로 열 때: writer가 열 때까지 무한 대기
  - `O_WRONLY`로 열 때: reader가 열 때까지 무한 대기
- **비블로킹 모드 해결**:
  - 상대방이 없어도 즉시 반환
  - reader가 없는 상태에서 `O_WRONLY|O_NDELAY`로 열면 ENXIO 에러 발생
  - writer가 없는 상태에서 `O_RDONLY|O_NDELAY`로 열면 성공하고 read() 시 0 반환

**O_CREAT**:
```c
fd = open(FIFOFILE, O_RDONLY | O_CREAT, 0666);
```
- **기능**: FIFO 파일이 없으면 생성
- **주의**: 일반 파일이 생성될 수 있으므로 FIFO에는 권장하지 않음

**O_EXCL**:
```c
fd = open(FIFOFILE, O_RDONLY | O_CREAT | O_EXCL, 0666);
```
- **기능**: 파일이 이미 존재하면 실패
- **용도**: 중복 생성 방지

### 3. read() / write()
```c
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```
- **기능**: FIFO를 통한 데이터 읽기/쓰기
- **비블로킹 모드에서의 동작**:
  - `read()`: 데이터가 없으면 0 반환 (EOF) 또는 EAGAIN 에러
  - `write()`: reader가 없으면 SIGPIPE 시그널 발생

### 4. unlink()
```c
int unlink(const char *pathname);
```
- **기능**: 파일시스템에서 파일 삭제
- **매개변수**: `pathname`: 삭제할 파일 경로
- **반환값**: 성공 시 0, 실패 시 -1

## FIFO 블로킹/비블로킹 동작 비교

### 블로킹 모드 (기본)
```c
// 서버 - 클라이언트가 열 때까지 대기
fd = open(FIFOFILE, O_RDONLY);        // 블로킹

// 클라이언트 - 서버가 열 때까지 대기  
fd = open(FIFOFILE, O_WRONLY);        // 블로킹
```

### 비블로킹 모드
```c
// 서버 - 즉시 반환
fd = open(FIFOFILE, O_RDONLY | O_NDELAY);     // 비블로킹

// 클라이언트 - reader가 없으면 ENXIO 에러
fd = open(FIFOFILE, O_WRONLY | O_NDELAY);     // 비블로킹
if (fd < 0 && errno == ENXIO) {
    printf("No reader available\n");
}
```

### 비블로킹 모드 예제
```c
// 비블로킹 서버 예제
#include <errno.h>

int fd;
char buf[BUFSIZ];
ssize_t n;

fd = open(FIFOFILE, O_RDONLY | O_NDELAY);
if (fd < 0) {
    perror("open");
    return -1;
}

while (1) {
    n = read(fd, buf, sizeof(buf));
    if (n > 0) {
        printf("Received: %s", buf);
    } else if (n == 0) {
        printf("No data available\n");
        sleep(1);  // 잠시 대기 후 재시도
    } else {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            printf("No data, try again later\n");
            sleep(1);
        } else {
            perror("read");
            break;
        }
    }
}
```

## 실습 코드 분석

### 서버 코드 (fifo_server.c)

```c
#include<sys/types.h>
#include<sys/stat.h>
#include<fcntl.h>
#include<unistd.h>
#include<stdio.h>
#define FIFOFILE "fifo"

int main(int argc, char **argv){
    int n, fd;
    char buf[BUFSIZ]; // BUFSIZ는 stdio.h에 정의된 상수

    unlink(FIFOFILE); // 기존의 FIFO 파일 삭제

    if(mkfifo(FIFOFILE, 0666)<0){ // FIFO 파일 생성
        perror("mkfifo()");
        return -1;
    }

    if((fd=open(FIFOFILE, O_RDONLY)) <0){ // FIFO 오픈 (블로킹 모드)
        perror("open()");
        return -1;
    }

    while((n = read(fd, buf, sizeof(buf))) > 0){
        printf("%s", buf);
    }

    close(fd);
    return 0;
}
```

**동작 과정**:
1. 기존 FIFO 파일이 있다면 `unlink()`로 삭제
2. `mkfifo()`로 새로운 FIFO 파일 생성 (권한: 0666)
3. `O_RDONLY` 모드로 FIFO 파일 열기 (읽기 전용)
4. 클라이언트로부터 데이터가 올 때까지 `read()` 대기
5. 받은 데이터를 화면에 출력

### 클라이언트 코드 (fifo_client.c)

```c
#include<stdio.h>
#include<fcntl.h>
#include<unistd.h>

#define FIFOFILE "fifo"

int main(int argc, char **argv){
    int n, fd;
    char buf[BUFSIZ];

    if((fd = open(FIFOFILE, O_WRONLY))<0){    // FIFO 를 연다 (블로킹 모드)
        perror("open()");
        return -1;
    }

    while((n = read(0, buf, sizeof(buf)))>0){ // 키보드로부터 데이터를 입력받는다.
        write(fd, buf, n);                    // FIFO로 데이터를 보낸다.
    }
    close(fd);

    return 0;
}
```

**동작 과정**:
1. 서버가 생성한 FIFO 파일을 `O_WRONLY` 모드로 열기 (쓰기 전용)
2. `read(0, ...)`: 표준 입력(키보드)에서 데이터 읽기
3. `write(fd, ...)`: FIFO를 통해 서버로 데이터 전송
4. EOF(Ctrl+D) 입력 시까지 반복

## 실행 방법

```bash
# 컴파일
gcc -o fifo_server fifo_server.c
gcc -o fifo_client fifo_client.c

# 실행 (터미널 2개 필요)
# 터미널 1: 서버 실행
./fifo_server

# 터미널 2: 클라이언트 실행
./fifo_client
```

## FIFO 파일 삭제

실행 후 남아있는 FIFO 파일은 다음 명령으로 삭제:
```bash
rm fifo
# 또는
unlink fifo
# 또는 프로그램에서
unlink(FIFOFILE);
```

## 주의사항

1. **블로킹 특성**: 
   - 서버가 `open(O_RDONLY)` 시 클라이언트가 열 때까지 대기
   - 클라이언트가 `open(O_WRONLY)` 시 서버가 열 때까지 대기
   - `O_NDELAY` 플래그로 해결 가능

2. **단방향 통신**: FIFO는 단방향이므로 양방향 통신 시 FIFO 2개 필요

3. **파일 권한**: `mkfifo()` 시 적절한 권한 설정 필요

4. **에러 처리**: 비블로킹 모드에서는 EAGAIN, ENXIO 등의 에러 처리 필요

5. **시그널 처리**: writer가 없는 상태에서 write() 시 SIGPIPE 발생 가능

## 정리

FIFO는 파일시스템 기반의 Named Pipe로, 서로 관련 없는 프로세스들 간의 통신을 가능하게 하는 효과적인 IPC 메커니즘입니다. O_NDELAY 같은 플래그를 활용하면 블로킹 문제를 해결하고 더 유연한 통신이 가능합니다. 일반 파이프의 제약사항을 해결하며, 파일 I/O 함수들을 그대로 사용할 수 있어 구현이 간단합니다.
