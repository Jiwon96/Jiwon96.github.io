---
title: "Docker Compose에서 Dockerfile로 리팩토링하기: MLOps 환경 구축 예제"
date: 2025-07-05
categories: [docker]
tags: [docker, docker-compose, dockerfile, mlops, airflow, mlflow, refactoring]
---

# Docker Compose에서 Dockerfile로 리팩토링하기

MLOps 환경을 구축하면서 Docker Compose만으로 모든 서비스를 정의했지만, 점점 복잡해지는 설정들 때문에 유지보수가 어려워졌다. 이번 글에서는 왜 Dockerfile로 리팩토링했는지, 어떤 장점이 있는지 살펴보자.

## 기존 구조의 문제점

처음에는 docker-compose.yml 하나로 모든 것을 해결하려고 했다:

```yaml
mlflow:
  image: python:3.9
  command: bash -c "pip install mlflow psycopg2-binary && mlflow server --backend-store-uri postgresql://mlflow:${POSTGRES_MLFLOW_PASSWORD}@postgres-mlflow:5432/mlflow --default-artifact-root ./mlruns --host 0.0.0.0 --port 5000"
```

**문제점들:**
- 긴 command로 인한 가독성 저하
- 컨테이너 시작할 때마다 패키지 재설치 (성능 저하)
- 복잡한 초기화 로직 관리 어려움
- 버전 관리 및 재사용성 부족

## 리팩토링 후 구조

### 1. MLflow 서비스 분리

**Before:**
```yaml
mlflow:
  image: python:3.9
  command: bash -c "pip install mlflow psycopg2-binary && mlflow server ..."
```

**After:**
```yaml
# docker-compose.yml
mlflow:
  build: 
    context: ./docker/mlflow
  environment:
    - POSTGRES_MLFLOW_PASSWORD=${POSTGRES_MLFLOW_PASSWORD}
```

```dockerfile
# docker/mlflow/Dockerfile
FROM python:3.9
WORKDIR /app
RUN pip install mlflow psycopg2-binary
EXPOSE 5000
CMD mlflow server \
    --backend-store-uri postgresql://mlflow:${POSTGRES_MLFLOW_PASSWORD}@postgres-mlflow:5432/mlflow \
    --default-artifact-root ./mlruns \
    --host 0.0.0.0 \
    --port 5000
```

**장점:**
- 이미지 빌드 시점에 패키지 설치 (레이어 캐싱으로 성능 향상)
- 명령어가 Dockerfile에 명확히 분리
- 환경변수 처리 개선

### 2. Airflow 서비스 개선

**Before:**
```yaml
airflow:
  image: apache/airflow:2.7.1
  command: bash -c "airflow db migrate && airflow users create --username ${AIRFLOW_ADMIN_USERNAME} --password ${AIRFLOW_ADMIN_PASSWORD} --firstname Admin --lastname User --role Admin --email ${AIRFLOW_ADMIN_EMAIL} && airflow webserver"
```

**After:**
```yaml
# docker-compose.yml
airflow:
  build:
    context: ./docker/airflow
  volumes:
    - ./dags:/opt/airflow/dags  # 로컬 개발 편의성 증대
```

```dockerfile
# docker/airflow/Dockerfile
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

```bash
# docker/airflow/entrypoint.sh
#!/bin/bash
set -e
airflow db migrate
airflow users create \
    --username ${AIRFLOW_ADMIN_USERNAME} \
    --password ${AIRFLOW_ADMIN_PASSWORD} \
    --firstname Admin \
    --lastname User \
    --role Admin \
    --email ${AIRFLOW_ADMIN_EMAIL} || true
exec airflow webserver
```

**장점:**
- 복잡한 초기화 로직을 스크립트로 분리
- 필요한 패키지들을 requirements.txt로 관리
- 에러 처리 개선 (`|| true`로 중복 계정 생성 방지)

### 3. 볼륨 구조 개선

**Before:**
```yaml
volumes:
  - airflow_dags:/opt/airflow/dags  # Named volume (수정 불편)
```

**After:**
```yaml
volumes:
  - ./dags:/opt/airflow/dags  # Bind mount (실시간 반영)
```

**장점:**
- 로컬에서 DAG 수정 시 즉시 반영
- 개발 생산성 크게 향상

## 리팩토링의 핵심 장점

### 1. **관심사의 분리**
- docker-compose.yml: 서비스 오케스트레이션
- Dockerfile: 개별 서비스 구성
- entrypoint.sh: 초기화 로직

### 2. **성능 개선**
- 패키지 설치가 이미지 빌드 시점으로 이동
- Docker 레이어 캐싱 활용
- 컨테이너 시작 시간 단축

### 3. **유지보수성 향상**
- 각 서비스별 독립적 관리
- 버전 관리 및 롤백 용이
- 설정 변경 시 영향 범위 명확

### 4. **개발 편의성**
- 로컬 파일 시스템과 실시간 동기화
- 디버깅 및 테스트 용이
- 재사용 가능한 컴포넌트

## 언제 Dockerfile로 분리해야 할까?

**Dockerfile이 필요한 경우:**
- 커스텀 패키지 설치가 필요한 경우
- 복잡한 초기화 로직이 있는 경우
- 애플리케이션 코드를 포함해야 하는 경우
- 환경 설정이 복잡한 경우

**기본 이미지만 사용해도 되는 경우:**
- PostgreSQL, Redis, MinIO 같은 표준 서비스
- 설정이 환경변수로만 충분한 경우

## 결론

처음에는 docker-compose.yml 하나로 시작하는 것이 간단해 보이지만, 프로젝트가 커질수록 Dockerfile로 분리하는 것이 필수가 된다. 

각 서비스의 책임을 명확히 분리하고, 개발 생산성과 유지보수성을 모두 향상시킬 수 있는 구조로 발전시키는 것이 중요하다.

MLOps 환경처럼 여러 서비스가 복잡하게 연결된 시스템에서는 더욱 그 가치가 빛난다.