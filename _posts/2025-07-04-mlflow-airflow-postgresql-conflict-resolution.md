---
layout: post
title: "MLflow & Airflow Docker 환경에서 PostgreSQL 충돌 해결하기"
date: 2025-07-04 00:00:00 +0900
categories: [MLOPS]
tags: [mlops, Docker, Docker-Compose, PostgreSQL, MLflow, Airflow, Alembic, 환경구축, 컨테이너, 트러블슈팅]
---

## 🚨 문제 상황

MLflow와 Airflow를 Docker Compose로 함께 운영하려다 보니 다음과 같은 오류들이 연속으로 발생했습니다.

### 1. Alembic 마이그레이션 충돌

```bash
mlflow-1 | alembic.script.revision.ResolutionError: No such revision or branch '405de8318b3a'
mlflow-1 | alembic.util.exc.CommandError: Can't locate revision identified by '405de8318b3a'
```

**원인**: Airflow와 MLflow가 같은 PostgreSQL 데이터베이스를 공유하면서 서로 다른 Alembic 마이그레이션 히스토리가 충돌

### 2. Docker Compose Version 경고

```bash
WARN[0000] /home/jiwon/mlops/docker-compose.yml: the attribute `version` is obsolete, it will be ignored
```

**원인**: 최신 Docker Compose에서는 `version` 필드가 더 이상 필요하지 않음

### 3. Airflow 데이터베이스 연결 실패

```bash
airflow-1 | ERROR! Maximum number of retries (20) reached.
airflow-1 | connection to server at "airflow" (172.24.0.5), port 5432 failed: Connection refused
```

**원인**: 잘못된 호스트명과 네트워크 설정으로 인한 연결 실패

## 🔧 해결 과정

### 단계 1: 데이터베이스 분리 전략 수립

가장 근본적인 해결책은 **각 서비스가 독립적인 PostgreSQL 인스턴스를 사용**하도록 하는 것입니다.

**장점:**
- 마이그레이션 충돌 방지
- 서비스 간 의존성 감소
- 데이터 격리 및 보안 향상
- 독립적인 스케일링 가능

### 단계 2: Docker 권한 설정

sudo 없이 Docker 명령어를 사용하기 위해 사용자를 docker 그룹에 추가:

```bash
# 사용자를 docker 그룹에 추가
sudo usermod -aG docker $USER

# 즉시 적용
newgrp docker

# 테스트
docker ps
docker-compose down -v  # sudo 없이 실행 가능!
```

### 단계 3: Docker Compose 설정 최적화

#### Before (문제가 있던 설정)

```yaml
version: '3.8'  # ❌ 최신 Docker Compose에서 불필요

services:
  postgres:  # ❌ 두 서비스가 같은 DB 공유
    image: postgres:13
    environment:
      - POSTGRES_DB=airflow
      
  airflow:
    environment:
      # ❌ 잘못된 호스트명
      - AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://postgres:airflow@airflow:5432/airflow
```

#### After (해결된 설정)

```yaml
# ✅ version 제거
services:
  # ✅ Airflow 전용 PostgreSQL
  postgres:
    image: postgres:13
    container_name: postgres-airflow
    environment:
      - POSTGRES_DB=airflow
      - POSTGRES_USER=airflow
      - POSTGRES_PASSWORD=airflow
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - mlops-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U airflow -d airflow"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ✅ MLflow 전용 PostgreSQL
  postgres-mlflow:
    image: postgres:13
    container_name: postgres-mlflow
    environment:
      - POSTGRES_DB=mlflow
      - POSTGRES_USER=mlflow
      - POSTGRES_PASSWORD=mlflow
    volumes:
      - postgres_mlflow_data:/var/lib/postgresql/data
    ports:
      - "5433:5432"  # ✅ 다른 포트 사용
    networks:
      - mlops-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mlflow -d mlflow"]
      interval: 10s
      timeout: 5s
      retries: 5

  airflow-webserver:
    image: apache/airflow:2.7.0-python3.9
    depends_on:
      postgres:
        condition: service_healthy  # ✅ DB 준비 대기
    environment:
      # ✅ 올바른 연결 문자열
      - AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@postgres:5432/airflow
    networks:
      - mlops-network  # ✅ 네트워크 연결
    command: >
      bash -c "
        airflow db migrate &&
        airflow users create 
          --username admin 
          --password admin 
          --firstname Admin 
          --lastname User 
          --role Admin 
          --email admin@example.com &&
        airflow webserver
      "

  mlflow:
    image: mlflow/mlflow:2.7.1
    depends_on:
      postgres-mlflow:
        condition: service_healthy
    networks:
      - mlops-network
    command: >
      bash -c "
        pip install psycopg2-binary &&
        mlflow db upgrade postgresql+psycopg2://mlflow:mlflow@postgres-mlflow:5432/mlflow &&
        mlflow server 
          --backend-store-uri postgresql+psycopg2://mlflow:mlflow@postgres-mlflow:5432/mlflow 
          --default-artifact-root ./mlruns 
          --host 0.0.0.0 
          --port 5000
      "

volumes:
  postgres_data:
  postgres_mlflow_data:

networks:
  mlops-network:
    driver: bridge
```

### 단계 4: 연결 문자열 디버깅

#### 연결 문자열 형식
```
postgresql+psycopg2://[사용자명]:[비밀번호]@[호스트명]:[포트]/[데이터베이스명]
```

#### 자주 발생하는 실수들
```bash
# ❌ 잘못된 예시들
postgresql://mlflow:@postgres-mlflow:5432/mlflow  # 비밀번호 누락
postgresql+psycopg2://airflow:airflow@airflow:5432/airflow  # 잘못된 호스트명
postgresql+psycopg2://postgres:airflow@postgres:5432/airflow  # 사용자명/비밀번호 순서 바뀜

# ✅ 올바른 예시들
postgresql+psycopg2://airflow:airflow@postgres:5432/airflow  # Airflow
postgresql+psycopg2://mlflow:mlflow@postgres-mlflow:5432/mlflow  # MLflow
```

## 🎯 최종 해결 방법

### 1. 완전한 초기화 (권장)

```bash
# 기존 환경 완전 정리
docker-compose down -v
docker system prune -f

# 필요한 디렉토리 생성
mkdir -p dags logs plugins mlruns

# 새로운 설정으로 시작
docker-compose up -d

# 로그 확인
docker-compose logs -f
```

### 2. 단계별 시작 (안전한 방법)

```bash
# 1단계: PostgreSQL만 먼저 시작
docker-compose up -d postgres postgres-mlflow

# 2단계: 헬스체크 확인
docker-compose exec postgres pg_isready -U airflow -d airflow
docker-compose exec postgres-mlflow pg_isready -U mlflow -d mlflow

# 3단계: 나머지 서비스 시작
docker-compose up -d airflow-webserver airflow-scheduler mlflow
```

### 3. 연결 테스트

```bash
# PostgreSQL 연결 확인
docker-compose exec postgres pg_isready -U airflow -d airflow
docker-compose exec postgres-mlflow pg_isready -U mlflow -d mlflow

# Airflow 데이터베이스 상태 확인
docker-compose exec airflow-webserver airflow db check

# 웹 UI 접속 테스트
curl http://localhost:8080  # Airflow
curl http://localhost:5000  # MLflow
```

## 📋 접속 정보

성공적으로 설정이 완료되면 다음 주소로 접속할 수 있습니다:

- **Airflow 웹 UI**: http://localhost:8080 (admin/admin)
- **MLflow UI**: http://localhost:5000
- **PostgreSQL Airflow**: localhost:5432
- **PostgreSQL MLflow**: localhost:5433

## 🎓 핵심 교훈

1. **데이터베이스 분리의 중요성**: 서로 다른 애플리케이션은 독립적인 데이터베이스를 사용해야 마이그레이션 충돌을 방지할 수 있습니다.

2. **네트워크 설정의 필요성**: Docker Compose에서 서비스 간 통신을 위해서는 명시적인 네트워크 설정이 필요합니다.

3. **헬스체크의 활용**: `depends_on`과 `condition: service_healthy`를 활용하여 의존성 서비스가 완전히 준비된 후 시작하도록 설정해야 합니다.

4. **연결 문자열 검증**: 데이터베이스 연결 문제의 대부분은 잘못된 연결 문자열에서 발생하므로, 형식을 정확히 확인해야 합니다.

5. **Docker 권한 관리**: 개발 환경에서는 docker 그룹 추가를 통해 sudo 없이 편리하게 작업할 수 있습니다.

## 🔗 추가 자료

- [Docker Compose Networking](https://docs.docker.com/compose/networking/)
- [Airflow Database Configuration](https://airflow.apache.org/docs/apache-airflow/stable/howto/set-up-database.html)
- [MLflow Tracking Server](https://mlflow.org/docs/latest/tracking.html#mlflow-tracking-servers)
- [PostgreSQL Docker Official Image](https://hub.docker.com/_/postgres)

이제 MLflow와 Airflow가 각자의 PostgreSQL 데이터베이스를 사용하여 안정적으로 운영됩니다! 🎉