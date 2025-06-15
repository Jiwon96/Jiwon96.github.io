---
layout: post
title: "posix_spawn() - 차세대 프로세스 생성 함수"
date: 2025-06-15 22:00:00 +0900
categories: [Linux, 시스템프로그래밍]
tags: [posix_spawn, fork, exec, process, performance]
---

# posix_spawn() - 차세대 프로세스 생성 함수

## 왜 posix_spawn()인가?

### fork()+exec()의 한계
- 불필요한 메모리 복사 오버헤드
- 임베디드 시스템에서의 성능 문제
- 복잡한 오류 처리

### posix_spawn()의 장점
- 원자적 프로세스 생성
- 메모리 효율성
- 표준화된 인터페이스

## 기본 사용법

### extern char **environ; // 선언만, 정의는 어디에?

```c
#include <spawn.h>

extern char **environ;  // 이것도 extern!
```

**의문**: 이것도 선언 없이 사용하네요?

**답**: glibc에서 제공하는 전역 변수입니다. 현재 프로세스의 환경변수 배열을 가리킵니다.

```c
int system(char *cmd) {
    pid_t pid;
    int status;
    
    char *argv[] = {"sh", "-c", cmd, NULL};
    posix_spawn(&pid, "/bin/sh", NULL, NULL, argv, environ);

    waitpid(pid, &status, 0);
    return status;
}
```

### fork()+exec() vs posix_spawn() 비교
```c
// 전통적인 방법
pid_t pid = fork();
if (pid == 0) {
    execl("/bin/sh", "sh", "-c", cmd, NULL);
    exit(127);
}

// posix_spawn() 방법  
char *argv[] = {"sh", "-c", cmd, NULL};
posix_spawn(&pid, "/bin/sh", NULL, NULL, argv, environ);
```

**posix_spawn()의 이점**:
- 한 번의 함수 호출로 프로세스 생성과 실행
- 커널 레벨 최적화 가능
- 메모리 효율성 향상

## 실제 사용 예제
```c
int main(int argc, char **argv, char **envp) {
    while (*envp) {
        printf("%s\n", *envp++);
    }
    
    system("who");
    system("nocommand");  // 오류 테스트
    system("cal");
    return 0;
}
```

## 언제 사용할까?

### 권장 상황
- **단순한 프로그램 실행**: posix_spawn() 권장
- **성능이 중요한 환경**: posix_spawn() 필수
- **임베디드 시스템**: posix_spawn() 강력 추천

### fork()+exec() 권장 상황
- **복잡한 설정 필요**: fork()+exec() 사용
- **자식 프로세스에서 복잡한 초기화 작업**

## 오류 처리의 차이점

```c
// fork()+exec(): errno 사용
if (fork() == -1) {
    perror("fork failed");
}

// posix_spawn(): 반환값이 오류 코드
int result = posix_spawn(&pid, "/bin/ls", NULL, NULL, argv, environ);
if (result != 0) {
    fprintf(stderr, "posix_spawn failed: %s\n", strerror(result));
}
```

posix_spawn()은 errno 대신 반환값으로 오류 코드를 제공합니다.

## 마무리

posix_spawn()은 단순한 프로그램 실행에서 fork()+exec()보다 효율적이고 사용하기 쉬운 대안입니다. 특히 성능이 중요한 환경에서는 필수적인 도구가 되고 있습니다.
