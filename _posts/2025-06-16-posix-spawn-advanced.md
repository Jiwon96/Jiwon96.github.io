---
layout: post
title: "posix_spawn() 고급 기능 - 파일 액션과 속성 제어"
date: 2025-06-16 13:00:00 +0900
categories: [Linux, System Programming]
tags: [posix_spawn, file_actions, spawnattr, pipe, redirect]
---

# posix_spawn() 고급 기능 - 파일 액션과 속성 제어

## 고급 posix_spawn() 활용

### 파일 액션과 속성 설정
```c
int system2(char *cmd) {
    pid_t pid;
    int status;
    posix_spawn_file_actions_t actions;
    posix_spawnattr_t attrs;

    char *argv[] = {"sh", "-c", cmd, NULL};
    
    // 초기화
    posix_spawn_file_actions_init(&actions);
    posix_spawnattr_init(&attrs);
    
    // 스케줄러 설정
    posix_spawnattr_setflags(&attrs, POSIX_SPAWN_SETSCHEDULER);
    
    posix_spawn(&pid, "/bin/sh", &actions, &attrs, argv, environ);

    waitpid(pid, &status, 0);
    
    // 정리
    posix_spawn_file_actions_destroy(&actions);
    posix_spawnattr_destroy(&attrs);
    
    return status;
}
```

## 파일 액션 (File Actions)

### 파일 리다이렉션
```c
posix_spawn_file_actions_t actions;
posix_spawn_file_actions_init(&actions);

// stdin을 파일로 리다이렉트
posix_spawn_file_actions_addopen(&actions, 0, "/input.txt", O_RDONLY, 0);

// stdout을 파일로 리다이렉트
posix_spawn_file_actions_addopen(&actions, 1, "/output.txt", O_WRONLY | O_CREAT, 0644);

// 파일 디스크립터 복제
posix_spawn_file_actions_adddup2(&actions, pipefd[1], STDOUT_FILENO);

// 파일 디스크립터 닫기
posix_spawn_file_actions_addclose(&actions, 3);
```

### 파이프라인 구현 예제
```c
// "cat file.txt | grep pattern" 구현
void create_pipeline() {
    int pipefd[2];
    pipe(pipefd);
    
    // cat 프로세스
    posix_spawn_file_actions_t cat_actions;
    posix_spawn_file_actions_init(&cat_actions);
    posix_spawn_file_actions_adddup2(&cat_actions, pipefd[1], STDOUT_FILENO);
    posix_spawn_file_actions_addclose(&cat_actions, pipefd[0]);
    posix_spawn_file_actions_addclose(&cat_actions, pipefd[1]);
    
    char *cat_argv[] = {"cat", "file.txt", NULL};
    pid_t cat_pid;
    posix_spawn(&cat_pid, "/bin/cat", &cat_actions, NULL, cat_argv, environ);
    
    // grep 프로세스
    posix_spawn_file_actions_t grep_actions;
    posix_spawn_file_actions_init(&grep_actions);
    posix_spawn_file_actions_adddup2(&grep_actions, pipefd[0], STDIN_FILENO);
    posix_spawn_file_actions_addclose(&grep_actions, pipefd[0]);
    posix_spawn_file_actions_addclose(&grep_actions, pipefd[1]);
    
    char *grep_argv[] = {"grep", "pattern", NULL};
    pid_t grep_pid;
    posix_spawn(&grep_pid, "/bin/grep", &grep_actions, NULL, grep_argv, environ);
    
    close(pipefd[0]);
    close(pipefd[1]);
    
    waitpid(cat_pid, NULL, 0);
    waitpid(grep_pid, NULL, 0);
    
    posix_spawn_file_actions_destroy(&cat_actions);
    posix_spawn_file_actions_destroy(&grep_actions);
}
```

## 프로세스 속성 (Spawn Attributes)

### 사용 가능한 플래그들
```c
posix_spawnattr_t attrs;
posix_spawnattr_init(&attrs);

// 시그널 마스크 설정
sigset_t sigmask;
sigemptyset(&sigmask);
sigaddset(&sigmask, SIGCHLD);
posix_spawnattr_setsigmask(&attrs, &sigmask);

// 프로세스 그룹 설정
posix_spawnattr_setpgroup(&attrs, 0);

// 플래그 조합
int flags = POSIX_SPAWN_SETPGROUP | 
           POSIX_SPAWN_SETSIGMASK | 
           POSIX_SPAWN_SETSCHEDULER;
posix_spawnattr_setflags(&attrs, flags);
```

### 주요 플래그 옵션
- **POSIX_SPAWN_SETPGROUP**: 프로세스 그룹 설정
- **POSIX_SPAWN_SETSIGMASK**: 시그널 마스크 설정
- **POSIX_SPAWN_SETSCHEDULER**: 스케줄링 정책 설정
- **POSIX_SPAWN_RESETIDS**: 사용자/그룹 ID 재설정
- **POSIX_SPAWN_SETSIGDEF**: 시그널 핸들러 기본값으로 재설정

## 고급 활용 사례

### 로그 파일로 출력 리다이렉트
```c
void run_with_logging(char *command) {
    posix_spawn_file_actions_t actions;
    posix_spawn_file_actions_init(&actions);
    
    // stdout과 stderr를 로그 파일로 리다이렉트
    posix_spawn_file_actions_addopen(&actions, 1, "/var/log/output.log", 
                                    O_WRONLY | O_CREAT | O_APPEND, 0644);
    posix_spawn_file_actions_adddup2(&actions, 1, 2);  // stderr = stdout
    
    char *argv[] = {"sh", "-c", command, NULL};
    pid_t pid;
    
    posix_spawn(&pid, "/bin/sh", &actions, NULL, argv, environ);
    waitpid(pid, NULL, 0);
    
    posix_spawn_file_actions_destroy(&actions);
}
```

### 보안이 강화된 프로세스 실행
```c
void secure_execute(char *program, char **args) {
    posix_spawnattr_t attrs;
    posix_spawnattr_init(&attrs);
    
    // 모든 시그널을 기본 처리로 재설정
    sigset_t default_signals;
    sigfillset(&default_signals);
    posix_spawnattr_setsigdefault(&attrs, &default_signals);
    
    // 새로운 프로세스 그룹 생성
    posix_spawnattr_setpgroup(&attrs, 0);
    
    // 권한 재설정
    posix_spawnattr_setflags(&attrs, 
                            POSIX_SPAWN_SETSIGDEF | 
                            POSIX_SPAWN_SETPGROUP | 
                            POSIX_SPAWN_RESETIDS);
    
    pid_t pid;
    posix_spawn(&pid, program, NULL, &attrs, args, environ);
    
    waitpid(pid, NULL, 0);
    posix_spawnattr_destroy(&attrs);
}
```

## 실무 활용 팁

### 메모리 관리
```c
// 항상 init/destroy 쌍으로 사용
posix_spawn_file_actions_t actions;
posix_spawnattr_t attrs;

posix_spawn_file_actions_init(&actions);  // 초기화
posix_spawnattr_init(&attrs);

// ... 사용 ...

posix_spawn_file_actions_destroy(&actions);  // 정리 필수!
posix_spawnattr_destroy(&attrs);
```

### 오류 처리 패턴
```c
int spawn_with_error_check(char *program, char **argv) {
    posix_spawn_file_actions_t actions;
    posix_spawnattr_t attrs;
    pid_t pid;
    int result;
    
    // 초기화
    if ((result = posix_spawn_file_actions_init(&actions)) != 0) {
        fprintf(stderr, "file_actions_init failed: %s\n", strerror(result));
        return -1;
    }
    
    if ((result = posix_spawnattr_init(&attrs)) != 0) {
        fprintf(stderr, "spawnattr_init failed: %s\n", strerror(result));
        posix_spawn_file_actions_destroy(&actions);
        return -1;
    }
    
    // 실행
    result = posix_spawn(&pid, program, &actions, &attrs, argv, environ);
    
    // 정리
    posix_spawn_file_actions_destroy(&actions);
    posix_spawnattr_destroy(&attrs);
    
    if (result != 0) {
        fprintf(stderr, "posix_spawn failed: %s\n", strerror(result));
        return -1;
    }
    
    return pid;
}
```

## 마무리

posix_spawn()의 고급 기능들을 활용하면 fork()+exec()로는 복잡하게 구현해야 할 기능들을 간단하고 효율적으로 처리할 수 있습니다. 특히 파일 리다이렉션, 파이프라인, 프로세스 속성 제어 등이 필요한 시스템 프로그래밍에서 매우 유용합니다.

파일 액션과 속성 설정을 통해 복잡한 프로세스 제어도 가능하지만, 너무 복잡한 경우에는 여전히 fork()+exec() 패턴이 더 적합할 수 있으니 상황에 맞게 선택하여 사용하시기 바랍니다.
