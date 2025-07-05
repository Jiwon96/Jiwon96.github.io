---
title: "Docker Dockerfile 완전 정복 - Apache Airflow 예제로 배우기"
date: 2025-07-05
categories: [docker]
tags: [dockerfile, airflow, container, 문법]
---

# Docker Dockerfile 완전 정복! 🐳

Apache Airflow Dockerfile을 통해 Docker 문법을 완벽하게 마스터해보겠습니다.

## 예제 Dockerfile

```dockerfile
FROM apache/airflow:2.7.1
USER airflow
COPY requirements.txt /tmp/
RUN pip install --no-cache-dir -r /tmp/requirements.txt
COPY entrypoint.sh /entrypoint.sh
USER root
RUN chmod +x /entrypoint.sh
USER airflow
ENTRYPOINT ["/entrypoint.sh"]
```

## 한 줄씩 완벽 분석

### FROM apache/airflow:2.7.1
이미 만들어진 Apache Airflow 2.7.1 이미지를 기반으로 시작합니다. 바퀴를 다시 발명할 필요 없죠! 이 이미지에는 Python, Airflow가 이미 설치되어 있어서 우리는 그 위에 필요한 것만 추가하면 됩니다.

### USER airflow
보안의 기본! root로 실행하는 건 위험하니까 airflow라는 일반 사용자로 전환합니다. 해킹당했을 때 피해를 최소화하는 방어 전략입니다.

### COPY requirements.txt /tmp/
로컬의 requirements.txt 파일을 컨테이너 안 /tmp/ 폴더로 복사합니다. 빌드 컨텍스트(Dockerfile이 있는 폴더)에서 가져오는 것입니다.

### RUN pip install --no-cache-dir -r /tmp/requirements.txt
- `pip install`: Python 패키지 설치
- `--no-cache-dir`: 캐시 파일 안 만들어서 이미지 크기 최적화
- `-r`: requirements.txt 파일에 적힌 패키지들 한 번에 설치

### COPY entrypoint.sh /entrypoint.sh
초기화 스크립트를 컨테이너 루트로 복사합니다. 이게 컨테이너 시작할 때 실행될 것입니다.

### USER root → RUN chmod +x → USER airflow
이게 핵심 패턴입니다!
1. root로 전환 (권한 변경하려면 관리자 권한 필요)
2. 파일 실행 권한 부여
3. 다시 일반 사용자로 복귀 (보안!)

### ENTRYPOINT ["/entrypoint.sh"]
컨테이너 시작할 때 항상 실행되는 명령어입니다. CMD와 다르게 덮어쓸 수 없어서 더 안전하고, 배열 형태로 쓰면 exec 형식이라 더 효율적입니다.

## 핵심 포인트 정리

- **보안을 위한 USER 변경 패턴**: root → 작업 → 일반사용자
- **이미지 크기 최적화**: --no-cache-dir 옵션 활용
- **레이어 최적화**: COPY → RUN 순서로 효율적 빌드
- **초기화 스크립트 자동 실행**: ENTRYPOINT로 안전하게 설정

이제 여러분도 Dockerfile 마스터가 되었습니다! 🎉