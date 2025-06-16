---
layout: post
title: "AOSP에서 콜백 방식 IPC 데이터 동기화의 설계 이유"
date: 2024-06-15 21:00:00 +0900
categories: [Android, NNHAL]
tags: [AOSP, HIDL, IPC, HAL, Callback, Memory]
---

# AOSP에서 콜백 방식 IPC 데이터 동기화의 설계 이유

## 목차
1. [개요](#개요)
2. [메모리 소유권 문제 해결](#메모리-소유권-문제-해결)
3. [IPC 성능 최적화](#ipc-성능-최적화)
4. [동기/비동기 콜백 구분](#동기비동기-콜백-구분)
5. [프로세스 격리 보안](#프로세스-격리-보안)
6. [실제 구현 예시](#실제-구현-예시)
7. [결론](#결론)

## 개요

Android HIDL(HAL Interface Definition Language)에서 IPC(Inter-Process Communication) 시 콜백 함수를 사용하는 이유는 **메모리 소유권 관리**와 **성능 최적화**를 위한 핵심 설계 결정입니다.

## 메모리 소유권 문제 해결

### 설계 원리
HIDL은 복잡한 메모리 소유권 문제를 피하기 위해 다음과 같은 원칙을 따릅니다:

- **입력 매개변수만 사용**: RPC에서 in 파라미터만 허용
- **콜백을 통한 반환**: 효율적으로 반환할 수 없는 값은 콜백 함수로 반환
- **소유권 유지**: 데이터 소유권이 항상 호출 함수에 유지됨

### 메모리 관리 규칙
```cpp
// 호출자가 데이터 소유권 유지
hidl_vec<hidl_string> data;
service->processData(data, [&](Result result) {
    // 콜백에서 결과 처리
    // 원본 data는 호출자가 계속 소유
});
```

## IPC 성능 최적화

### 불필요한 복사 제거
전통적인 IPC 방식의 문제점:
- **4번의 데이터 복사**: 입력파일 → 커널 → IPC 채널 → 클라이언트 버퍼 → 출력파일
- **공유 메모리 방식**: 입력파일 → 공유메모리 → 출력파일 (2번만 복사)

### HIDL의 최적화
- **자동 직렬화/역직렬화**: 개발자가 명시적 처리 불필요
- **데이터 지속성**: 함수 호출 기간 동안만 유지
- **Zero-copy 지향**: 가능한 한 데이터 복사 최소화

## 동기/비동기 콜백 구분

### 동기 콜백 (Synchronous Callback)
여러 값을 반환하거나 비원시 타입을 반환할 때 사용:

```cpp
// 원시 타입 하나 - 콜백 불필요
Return<int32_t> getValue();

// 복수 값 - 콜백 필요
Return<void> getValues(getValues_cb callback);
// 구현: callback(value1, value2, complexData);
```

### 비동기 콜백 (Asynchronous Callback)
HAL에서 프레임워크로의 비동기 알림:

```cpp
// 센서 데이터 스트리밍
interface ISensorCallback {
    oneway onSensorDataChanged(SensorData data);
};
```

## 프로세스 격리 보안

### Binderized HAL의 보안 이점
- **프로세스 분리**: HAL 서비스가 별도 프로세스에서 실행
- **장애 격리**: HAL 크래시가 시스템 전체에 영향 주지 않음
- **접근 제어**: Binder IPC를 통한 엄격한 권한 관리

### 보안 설계
```cpp
// 서비스 등록 시 권한 검증
service->registerAsService();  // 권한 확인 후 등록
// 클라이언트 접근 시 SELinux 정책 적용
```

## 실제 구현 예시

### 오디오 HAL 예시
```cpp
// 오디오 데이터 스트리밍
Return<void> readFrames(readFrames_cb callback) {
    // 하드웨어에서 오디오 데이터 읽기
    AudioBuffer buffer = readFromHardware();
    
    // 콜백으로 데이터 반환 (소유권은 호출자 유지)
    callback(Result::OK, buffer);
    return Void();
}
```

### 센서 HAL 예시
```cpp
// 센서 정보 조회
Return<void> getSensorsList(getSensorsList_cb callback) {
    hidl_vec<SensorInfo> sensors = enumerateSensors();
    callback(sensors);  // 벡터 데이터를 콜백으로 반환
    return Void();
}
```

## 결론

AOSP의 콜백 기반 IPC 설계는 다음과 같은 핵심 이점을 제공합니다:

1. **메모리 안전성**: 명확한 소유권 모델로 메모리 누수/충돌 방지
2. **성능 향상**: 불필요한 데이터 복사 최소화
3. **보안 강화**: 프로세스 격리를 통한 시스템 안정성 확보
4. **개발 편의성**: 자동 직렬화로 개발자 부담 감소

이러한 설계는 Android의 **Project Treble** 아키텍처에서 vendor/framework 분리를 지원하는 핵심 기술로 활용되고 있습니다.

---

### 참고 자료
- [Android HIDL 공식 문서](https://source.android.com/docs/core/architecture/hidl)
- [HIDL 서비스 & 데이터 전송](https://source.android.com/devices/architecture/hidl/services)
- [Android IPC 메커니즘](https://source.android.com/docs/core/architecture/hidl/binder-ipc)