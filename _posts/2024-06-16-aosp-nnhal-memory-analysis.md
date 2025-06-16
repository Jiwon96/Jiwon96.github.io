---
layout: post
title: "AOSP 14 Neural Networks HAL 메모리 구조 분석"
date: 2024-06-16 21:00:00 +0900
categories: [Android, NNHAL]
tags: [AOSP, Neural Networks, HAL, HIDL, NPU, Memory]
---

# AOSP 14 Neural Networks HAL 메모리 구조 분석

## 개요

AOSP 14에서 Neural Networks HAL의 `hidl_vec<bool>& supportedOperations` 매개변수가 어디에 저장되고 어떻게 할당되는지에 대한 심층 분석입니다. 이 글에서는 HAL 서버 프로세스의 메모리 구조와 NPU 하드웨어와의 관계를 중심으로 분석합니다.

## 메모리 구조 분석

### 1. hidl_vec<bool>& 메모리 위치

**결론**: NPU HAL 서버 프로세스의 유저스페이스 힙 메모리에 저장

#### 근거:
- **프로세스 분리**: Binderized HAL은 독립적인 유저스페이스 프로세스로 실행
- **동적 할당**: `hidl_vec`는 C++ STL vector와 유사하게 동적 크기 지원
- **힙 할당**: 가변 크기 컨테이너는 힙 메모리에 할당

### 2. 시스템 아키텍처

```
Android Framework (system_server)
        ↓ Binder IPC
NPU HAL 서버 프로세스 (유저스페이스)
        ↓ 시스템콜 (/dev/npu)
NPU 커널 드라이버 (커널스페이스)
        ↓ 하드웨어 I/O
NPU 하드웨어 (물리적 칩)
```

## NPU HAL 서버와 NPU 하드웨어의 관계

### 1. 독립적인 실행 단위
- **NPU 하드웨어**: AI/ML 작업을 위한 전용 하드웨어 칩 (물리적 프로세서)
- **NPU HAL 서버**: 유저스페이스에서 실행되는 소프트웨어 프로세스 (독립 프로세스)
- **NPU 커널 드라이버**: 커널스페이스에서 NPU 하드웨어 제어하는 드라이버 모듈

### 2. 통신 경로
```
Android Framework 
    ↓ Binder IPC
NPU HAL 서버 프로세스 (유저스페이스)
    ↓ 시스템콜 (/dev/npu)
NPU 커널 드라이버 (커널스페이스)
    ↓ 하드웨어 인터페이스
NPU 하드웨어 (물리적 칩)
```

### 3. 실제 통신 예시
```cpp
// HAL 서버에서 NPU 커널 드라이버와 직접 통신
fd = open("/dev/npu", O_RDWR);
ioctl(fd, NPU_EXECUTE_MODEL, &model_data);
```

## Binderized HAL 서버-클라이언트 구조의 이유

### 단일 프로세스 구조의 문제점
1. **보안 위험**: 벤더 코드가 system_server에서 실행시 시스템 전체 손상 가능
2. **안정성 문제**: HAL 코드 크래시가 Android 시스템 전체 다운 유발
3. **업데이트 제약**: 벤더 코드와 AOSP 코드 결합으로 독립 업데이트 불가

### 서버-클라이언트 구조의 장점
1. **프로세스 격리**: HAL 서버 문제가 클라이언트(시스템 서버)에 영향을 주지 않음
2. **리소스 독점성**: HAL 서버가 하드웨어 독점 접근하면서 다중 클라이언트 지원 동시 제공
3. **유지보수성**: 서버(벤더 HAL)와 클라이언트(AOSP 프레임워크) 독립적 개발 가능
4. **보안 경계**: Binder IPC를 통한 명확한 프로세스 간 통신으로 추가 보안 계층 제공
5. **확장성**: 여러 클라이언트가 동시에 같은 HAL 서버에 접근 가능
6. **독립적 재시작**: HAL 서버 크래시 시 해당 서비스만 재시작하고 시스템 전체는 정상 동작 유지

## 결론

AOSP 14의 Neural Networks HAL에서 `hidl_vec<bool>& supportedOperations`는:

1. **메모리 위치**: NPU HAL 서버 프로세스의 유저스페이스 힙 메모리에 저장
2. **프로세스 독립성**: NPU 하드웨어와 완전히 별개인 독립적인 소프트웨어 프로세스
3. **통신 구조**: HAL 서버가 시스템콜을 통해 NPU 커널 드라이버와 직접 통신
4. **설계 목적**: 보안과 안정성을 위한 프로세스 분리 아키텍처

이러한 메모리 구조는 성능 오버헤드가 있지만, 현대 모바일 시스템에서 요구되는 보안성과 안정성을 확보하는 핵심 설계입니다.

---

**참고 자료**:
- [AOSP Neural Networks HAL Documentation](https://source.android.com/docs/core/interaction/neural-networks)
- [Android Hardware Interfaces](https://android.googlesource.com/platform/hardware/interfaces/)
- [Binderized HAL Architecture](https://www.embien.com/blog/a-deep-dive-into-aosp-binderized-hal)
