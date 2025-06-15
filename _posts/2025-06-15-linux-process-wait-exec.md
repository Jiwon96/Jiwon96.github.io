---
layout: post
title: "리눅스 프로세스 관리: wait()와 exec() 함수 완전 분석"
date: 2025-06-15 21:00:00 +0900
categories: [Linux, 시스템프로그래밍]
tags: [wait, exec, process, zombie, fork]
---

# 리눅스 프로세스 관리: wait()와 exec() 함수 완전 분석

## 목차
1. wait() 함수 계열 개요
2. exec() 함수 계열 개요  
3. exec()의 실행 파일 처리 메커니즘
4. 프로세스 그룹과 세션 관리
5. 실제 사용 예제

---

## 1. wait() 함수 계열 - 좀비 프로세스의 비밀

### 기본 동작 원리
**근거**: POSIX 표준과 Linux man pages
- `wait()`: 임의의 자식 프로세스 종료 대기
- `waitpid()`: 특정 자식 프로세스 지정 대기
- `waitid()`: 더 세밀한 제어 옵션 제공

```c
#include <sys/wait.h>

pid_t wait(int *status);
pid_t waitpid(pid_t pid, int *status, int options);
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
```

### 좀비 프로세스가 발생하는 진짜 이유
자식 프로세스가 종료되어도 부모가 wait()를 호출하지 않으면 좀비가 됩니다. 하지만 **왜 wait() 처리가 끝나지 않을까요?**

**주요 원인들**:
1. **부모가 wait() 호출을 안함**: 가장 흔한 실수
2. **부모가 무한 대기 상태**: I/O 블로킹으로 wait()에 도달 못함
3. **잘못된 시그널 처리**: SIGCHLD 무시하거나 잘못 처리

### 핵심 동작 과정과 EINTR 처리
```c
// 부모 프로세스에서 안전한 자식 대기
while(waitpid(pid, &status, 0) < 0) {
    if(errno != EINTR) {  // EINTR이 뭔가요?
        status = -1;
        break;
    }
}
```

**EINTR이 뭔가요?**: 시그널로 인해 시스템 호출이 중단되었다는 의미입니다. 이때는 재시도해야 합니다.

---

## 2. exec() 함수 계열

### argv[0]는 왜 "sh"인가요?
많이 헷갈리는 부분입니다!

```c
execl("/bin/sh", "sh", "-c", cmd, (char *)0);
//     ^실행할    ^argv[0]  
//     바이너리    (프로그램 이름)
```

**오해**: argv[0]가 "sh"면 바이너리가 아니라 쉘을 실행한다?
**진실**: `/bin/sh`는 **바이너리 프로그램**입니다! argv[0]는 단지 프로그램이 자신의 이름을 알기 위한 용도입니다.

---

## 3. 실제 사용 예제

### system() 함수 구현하며 이해하기
```c
int system(const char *cmd) {
    pid_t pid;
    int status;

    if((pid = fork()) < 0) {
        status = -1;
    } else if(pid == 0) {
        // 자식: 쉘을 통해 명령어 실행
        execl("/bin/sh", "sh", "-c", cmd, (char *)0);
        _exit(127);  // exec 실패 시
    } else {
        // 부모: 자식 종료 대기 (좀비 방지)
        while(waitpid(pid, &status, 0) < 0) {
            if(errno != EINTR) {
                status = -1;
                break;
            }
        }
    }
    return status;
}
```

### ~/.bashrc와 envp[]의 관계

**의문**: 환경변수가 프로그램 실행에 어떤 도움을 주나요?

**답**: 가장 중요한 것은 **PATH** 환경변수입니다!

```bash
# PATH가 있어야 이렇게 실행 가능
ls
# PATH에서 /bin/ls를 찾아서 실행

# PATH가 없다면
unset PATH
ls  # command not found!
/bin/ls  # 전체 경로로만 실행 가능
```

### *envp++ 연산자 우선순위 분석
```c
printf("%s\n", *envp++);
```

**헷갈리는 포인트**: 이게 `*(envp++)`인가요, `(*envp)++`인가요?

**정답**: `*(envp++)`입니다!

```c
// 실제 동작
char *current = envp;     // 현재 envp 저장
envp = envp + 1;          // envp를 다음으로 이동
return *current;          // 원래 값 반환
```

---

## 마무리

wait()와 exec() 함수는 유닉스/리눅스 시스템에서 프로세스 관리의 핵심입니다. exec()는 ELF 바이너리는 직접 실행하고, 스크립트는 적절한 인터프리터를 통해 실행하며, 프로세스 그룹과 세션을 통해 터미널과의 상호작용을 제어합니다.
