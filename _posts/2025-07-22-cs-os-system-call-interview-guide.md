---
title: "운영체제 면접 완전 정복 - 시스템 콜부터 Dual Mode까지"
date: 2025-07-22 23:00:00 +0900
categories: [CS, OS]
tags: [operating-system, system-call, dual-mode, cpu-ring, interview, cs-interview, kernel-mode, user-mode]
author: jiwon
toc: true
toc_sticky: true
---

## 간략버전 (면접 대비용)

### 1. 시스템 콜이 무엇인지 설명해주세요.

**A: 시스템 콜은 사용자 프로그램이 운영체제의 커널 서비스에 접근하기 위해 사용하는 인터페이스입니다.**

**핵심 키워드:**
- **보호 모드(Protection Mode)**: 현대 운영체제의 보안 메커니즘
- **커널 모드(Kernel Mode)**: 모든 하드웨어 접근이 가능한 실행 모드 (Ring 0)
- **사용자 모드(User Mode)**: 제한된 권한으로 실행되는 모드 (Ring 3)
- **모드 전환(Mode Switch)**: 사용자 모드 ↔ 커널 모드 간 전환
- **소프트웨어 인터럽트**: 시스템 콜 호출 메커니즘

### 2. 시스템 콜의 예시

**5가지 유형별 주요 예시:**
- **프로세스 제어**: `fork()`, `exec()`, `exit()`, `wait()`
- **파일 관리**: `open()`, `read()`, `write()`, `close()`
- **장치 관리**: `ioctl()` (터미널 제어, 네트워크 설정)
- **정보 유지**: `getpid()`, `time()`, `getuid()`
- **통신**: `pipe()`, `socket()`, `shmget()`, `mmap()`

### 3. 시스템 콜 실행 과정 (7단계)

1. **사용자 프로그램**에서 시스템 콜 호출
2. **라이브러리 래퍼 함수**에서 시스템 콜 번호와 매개변수 준비
3. **syscall 명령어** 실행으로 하드웨어 인터럽트 발생
4. **모드 전환**: Ring 3 (사용자) → Ring 0 (커널)
5. **커널 처리**: 시스템 콜 번호 검증 → 테이블 조회 → 함수 실행
6. **결과 반환**: 레지스터에 결과값 저장
7. **모드 복귀**: Ring 0 → Ring 3, 사용자 프로그램 재개

### 4. Dual Mode 개념

**Dual Mode**: CPU를 **사용자 모드(Ring 3)**와 **커널 모드(Ring 0)**로 구분

**구분 이유:**
- **시스템 안정성**: 사용자 프로그램 오류가 전체 시스템에 영향 방지
- **보안**: 악의적 코드로부터 시스템 보호
- **리소스 관리**: 공정한 자원 배분과 제어
- **하드웨어 추상화**: 일관된 인터페이스 제공

### 5. 시스템 콜 구분 방법

**시스템 콜 번호 + 시스템 콜 테이블:**
```c
#define __NR_read    0    // read() 시스템 콜
#define __NR_write   1    // write() 시스템 콜
#define __NR_open    2    // open() 시스템 콜

sys_call_table[1] = sys_write;  // 번호 1번은 write 함수 호출
```

**레지스터 규약 (x86_64):**
- `rax`: 시스템 콜 번호
- `rdi, rsi, rdx`: 첫 번째, 두 번째, 세 번째 매개변수

---

## 이해용 상세 설명

### 시스템 콜의 근본적 필요성

현대 운영체제는 **보호 모드(Protection Mode)**로 동작합니다. 이는 **CPU Ring 구조**를 활용한 권한 분리 메커니즘입니다.

#### CPU Ring 구조 (x86 아키텍처)
```
┌─────────────────────────────────┐
│         Ring 0 (커널)            │  ← 최고 권한 (Supervisor Mode)
│    - 운영체제 커널               │    모든 하드웨어 접근 가능
│    - 디바이스 드라이버           │    특권 명령어 실행 가능
├─────────────────────────────────┤
│         Ring 1 (드라이버)        │  ← 중간 권한 (현재 미사용)
│    - 디바이스 드라이버           │    가상화 환경에서만 활용
│    - 시스템 서비스               │
├─────────────────────────────────┤
│         Ring 2 (서비스)          │  ← 중간 권한 (현재 미사용)
│    - 시스템 서비스               │    복잡성 대비 이익 부족
│    - 특권 애플리케이션           │
├─────────────────────────────────┤
│         Ring 3 (사용자)          │  ← 최저 권한 (User Mode)
│    - 일반 애플리케이션           │    제한된 권한
│    - 사용자 프로그램             │    하드웨어 직접 접근 불가
└─────────────────────────────────┘
```

#### 왜 Ring 1, 2는 사용하지 않을까?

**1. 복잡성 증가**
```c
// 4단계 Ring을 모두 사용할 경우
Ring 3 → Ring 2 → Ring 1 → Ring 0
// 각 단계마다 권한 검사와 모드 전환 필요
// 성능 오버헤드가 기하급수적으로 증가
```

**2. 실용성 부족**
- 대부분의 작업은 **"일반 작업"** 또는 **"특권 작업"**으로 구분 가능
- 중간 단계의 필요성이 실제로는 크지 않음
- **Dual Mode (Ring 0 + Ring 3)**만으로도 충분한 보안과 기능 제공

**3. 예외적 사용 사례: 가상화**
```c
// 가상화 환경에서의 Ring 활용
┌─────────────────────────────────┐
│         Ring 0 (하이퍼바이저)     │  ← VMware, VirtualBox
├─────────────────────────────────┤
│         Ring 1 (게스트 커널)      │  ← 가상머신의 운영체제
├─────────────────────────────────┤
│         Ring 3 (게스트 사용자)    │  ← 가상머신의 애플리케이션
└─────────────────────────────────┘
```

### 시스템 콜 실행 과정 상세 분석

#### 1. 사용자 공간에서의 준비
```c
// 사용자 프로그램
int fd = open("file.txt", O_RDONLY);

// glibc 래퍼 함수 (간략화)
ssize_t open(const char *pathname, int flags) {
    return syscall(__NR_open, pathname, flags);
}
```

#### 2. 시스템 콜 번호와 매개변수 준비
```assembly
# glibc의 syscall 함수
mov $2, %rax        # __NR_open (시스템 콜 번호 2)
mov %rdi, %rdi      # 첫 번째 인자 (파일명)
mov %rsi, %rsi      # 두 번째 인자 (플래그)
syscall             # 커널로 제어권 이전
```

#### 3. 하드웨어 레벨 모드 전환
```c
// CPU가 자동으로 수행하는 작업
1. 현재 실행 주소 (RIP) 저장
2. 플래그 레지스터 (RFLAGS) 저장
3. 사용자 스택 → 커널 스택 전환
4. Ring 3 → Ring 0 전환
```

#### 4. 커널의 시스템 콜 처리
```c
// 커널 내부 디스패처
long do_syscall_64(unsigned long nr, struct pt_regs *regs) {
    // 1. 시스템 콜 번호 유효성 검사
    if (unlikely(nr >= NR_syscalls))
        return -ENOSYS;
    
    // 2. 시스템 콜 테이블에서 함수 호출
    return sys_call_table[nr](regs);
}

// 시스템 콜 테이블
const sys_call_ptr_t sys_call_table[] = {
    [0] = sys_read,
    [1] = sys_write,
    [2] = sys_open,      // 우리가 호출한 open
    [3] = sys_close,
    // ... 수백 개의 시스템 콜들
};
```

#### 5. 실제 시스템 콜 함수 실행
```c
// sys_open 함수의 실제 구현
SYSCALL_DEFINE3(open, const char __user *, filename, 
                int, flags, umode_t, mode) {
    // 1. 사용자 공간 매개변수 검증
    // 2. 파일 경로 해석 (/home/user/file.txt → inode)
    // 3. 권한 검사 (읽기 권한 확인)
    // 4. 파일 디스크립터 할당
    // 5. 파일 시스템 호출 (ext4, NTFS 등)
    
    return do_sys_open(AT_FDCWD, filename, flags, mode);
}
```

### 시스템 콜 유형별 상세 분석

#### 1. 프로세스 제어 시스템 콜

**fork() - 프로세스 복제**
```c
#include <unistd.h>

pid_t pid = fork();
if (pid == 0) {
    // 자식 프로세스: 부모의 완전한 복사본
    printf("자식 PID: %d\n", getpid());
    execl("/bin/ls", "ls", "-l", NULL);  // 새 프로그램 실행
} else if (pid > 0) {
    // 부모 프로세스
    int status;
    wait(&status);  // 자식 프로세스 완료 대기
    printf("자식 프로세스 완료\n");
}
```

**커널 내부 동작:**
- **프로세스 제어 블록(PCB)** 복제
- **메모리 공간** Copy-on-Write로 복사
- **파일 디스크립터 테이블** 상속
- **새로운 PID** 할당

#### 2. 파일 관리 시스템 콜

**완전한 파일 I/O 예시**
```c
#include <fcntl.h>
#include <unistd.h>

int main() {
    int fd;
    char buffer[1024];
    ssize_t bytes_read, bytes_written;
    
    // 파일 열기
    fd = open("data.txt", O_RDWR | O_CREAT, 0644);
    if (fd == -1) {
        perror("파일 열기 실패");
        return 1;
    }
    
    // 파일 읽기
    bytes_read = read(fd, buffer, sizeof(buffer) - 1);
    if (bytes_read > 0) {
        buffer[bytes_read] = '\0';
        printf("읽은 내용: %s\n", buffer);
    }
    
    // 파일 포인터 이동
    lseek(fd, 0, SEEK_END);  // 파일 끝으로 이동
    
    // 파일 쓰기
    bytes_written = write(fd, "\n새로운 데이터", 13);
    
    // 파일 닫기
    close(fd);
    return 0;
}
```

#### 3. 통신 시스템 콜

**파이프를 통한 프로세스 간 통신**
```c
#include <unistd.h>
#include <string.h>

int main() {
    int pipefd[2];
    pid_t pid;
    char write_msg[] = "부모에서 자식으로 메시지";
    char read_msg[100];
    
    // 파이프 생성
    if (pipe(pipefd) == -1) {
        perror("파이프 생성 실패");
        return 1;
    }
    
    pid = fork();
    if (pid == 0) {
        // 자식 프로세스: 읽기
        close(pipefd[1]);  // 쓰기 파이프 닫기
        read(pipefd[0], read_msg, sizeof(read_msg));
        printf("자식이 받은 메시지: %s\n", read_msg);
        close(pipefd[0]);
    } else {
        // 부모 프로세스: 쓰기
        close(pipefd[0]);  // 읽기 파이프 닫기
        write(pipefd[1], write_msg, strlen(write_msg) + 1);
        close(pipefd[1]);
        wait(NULL);  // 자식 프로세스 대기
    }
    return 0;
}
```

### 성능 최적화 기법

#### 1. 버퍼링을 통한 시스템 콜 최소화
```c
// 비효율적: 여러 번의 시스템 콜
for (int i = 0; i < 1000; i++) {
    write(fd, &data[i], sizeof(int));  // 1000번의 시스템 콜
}

// 효율적: 한 번의 시스템 콜
write(fd, data, 1000 * sizeof(int));  // 1번의 시스템 콜
```

#### 2. 메모리 매핑 활용
```c
#include <sys/mman.h>

// 대용량 파일 처리 시 메모리 매핑 사용
int fd = open("large_file.dat", O_RDWR);
struct stat sb;
fstat(fd, &sb);

// 파일을 메모리에 매핑
void *mapped = mmap(NULL, sb.st_size, PROT_READ | PROT_WRITE, 
                    MAP_SHARED, fd, 0);

// 이제 파일 내용을 메모리처럼 접근 가능
((int*)mapped)[0] = 42;  // read/write 시스템 콜 없이 직접 수정

munmap(mapped, sb.st_size);
close(fd);
```

### 다른 아키텍처의 권한 모델

#### ARM 아키텍처
```c
// ARM의 Exception Levels
EL0: User Mode        // Ring 3과 유사
EL1: Kernel Mode      // Ring 0과 유사
EL2: Hypervisor Mode  // 가상화 지원
EL3: Secure Monitor   // 보안 확장 (TrustZone)
```

#### RISC-V 아키텍처
```c
// RISC-V의 권한 모드
U-Mode: User Mode        // 사용자 모드
S-Mode: Supervisor Mode  // 커널 모드 (Linux 실행)
M-Mode: Machine Mode     // 최고 권한 (부트로더, 펌웨어)
```

### 면접에서 자주 나오는 심화 질문들

1. **"printf()는 시스템 콜인가요?"**
   - **답**: `printf()`는 라이브러리 함수이고, 내부적으로 `write()` 시스템 콜을 호출합니다.

2. **"fork() 후에 파일 디스크립터는 어떻게 되나요?"**
   - **답**: 자식 프로세스가 부모의 파일 디스크립터 테이블을 상속받아 같은 파일을 가리킵니다.

3. **"시스템 콜의 성능 오버헤드는 얼마나 되나요?"**
   - **답**: 일반적으로 100-300 CPU 사이클 정도로, 함수 호출 대비 10-100배 느립니다.

4. **"Ring 1, 2가 있는데 왜 사용하지 않나요?"**
   - **답**: 복잡성 대비 실질적 이익이 부족하며, Dual Mode만으로도 충분한 보안을 제공하기 때문입니다.

5. **"가상화에서는 Ring 구조가 어떻게 변하나요?"**
   - **답**: 하이퍼바이저가 Ring 0, 게스트 OS가 Ring 1에서 실행되어 권한을 제한합니다.

### 결론

**시스템 콜**은 **Dual Mode 구조**를 기반으로 한 **안전하고 통제된 인터페이스**입니다. **Ring 0 (커널 모드)**와 **Ring 3 (사용자 모드)** 간의 전환을 통해 **시스템 안정성, 보안, 리소스 관리**를 모두 보장하면서도 **성능 최적화**가 가능한 현실적인 해결책입니다.
