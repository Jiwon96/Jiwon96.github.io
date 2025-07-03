---
layout: post
title: "온디바이스 MLOps 환경 구축: Docker Compose로 MLflow, Airflow, MinIO 통합하기"
date: 2025-07-04 01:00:00 +0900
categories: [MLOPS]
tags: [mlops, docker, mlflow, airflow, minio, postgresql, 환경구축]
author: jiwon
---

## 개요

Docker Compose를 활용한 완전한 온프레미스 MLOps 환경 구축 가이드입니다. MLflow(실험 추적), Airflow(워크플로우), MinIO(온프레미스 객체 저장소), PostgreSQL(메타데이터)을 통합한 프로덕션급 MLOps 스택을 구성합니다.

## 아키텍처 설계 원칙

### 서비스 분리 전략
MLOps 모범 사례에 따라 각 서비스별로 독립적인 데이터베이스를 구성했습니다:

- **MLflow PostgreSQL**: 실험 메타데이터, 모델 정보 저장
- **Airflow PostgreSQL**: 워크플로우 상태, 스케줄링 정보 저장
- **MinIO**: 온프레미스 객체 저장소 (S3 호환 API 제공)
- **서비스 격리**: 장애 전파 방지 및 독립적 스케일링 지원

### 데이터베이스 분리의 장점

각 서비스마다 별도의 PostgreSQL 인스턴스를 사용하는 이유:

1. **장애 격리**: MLflow DB 문제가 Airflow 워크플로우에 영향을 주지 않음
2. **성능 최적화**: 각 서비스별 특성에 맞는 DB 튜닝 가능
3. **스케일링 독립성**: 서비스별로 독립적인 확장 가능
4. **백업/복구 전략**: 서비스별 독립적인 데이터 관리

## Docker Compose 구성

### 완전한 docker-compose.yml

```yaml
services:
  postgres:
    image: postgres:13
    environment:
      - POSTGRES_DB=airflow
      - POSTGRES_USER=airflow
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  postgres-mlflow:
    image: postgres:13
    environment:
      - POSTGRES_DB=mlflow
      - POSTGRES_USER=mlflow
      - POSTGRES_PASSWORD=${POSTGRES_MLFLOW_PASSWORD}
    volumes:
      - postgres_mlflow_data:/var/lib/postgresql/data

  mlflow:
    image: python:3.9
    command: bash -c "pip install mlflow psycopg2-binary && mlflow server --backend-store-uri postgresql://mlflow:${POSTGRES_MLFLOW_PASSWORD}@postgres-mlflow:5432/mlflow --default-artifact-root ./mlruns --host 0.0.0.0 --port 5000"
    ports:
      - "5000:5000"
    depends_on:
      - postgres-mlflow
    volumes:
      - mlflow_data:/app/mlruns

  airflow:
    image: apache/airflow:2.7.1
    environment:
      - AIRFLOW__CORE__EXECUTOR=LocalExecutor
      - AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:${POSTGRES_PASSWORD}@postgres:5432/airflow
      - AIRFLOW__CORE__FERNET_KEY=${AIRFLOW_FERNET_KEY}
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    volumes:
      - airflow_dags:/opt/airflow/dags
      - airflow_logs:/opt/airflow/logs
    command: bash -c "airflow db migrate && airflow users create --username ${AIRFLOW_ADMIN_USERNAME} --password ${AIRFLOW_ADMIN_PASSWORD} --firstname Admin --lastname User --role Admin --email ${AIRFLOW_ADMIN_EMAIL} && airflow webserver"

  airflow-scheduler:
    image: apache/airflow:2.7.1
    environment:
      - AIRFLOW__CORE__EXECUTOR=LocalExecutor
      - AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:${POSTGRES_PASSWORD}@postgres:5432/airflow
      - AIRFLOW__CORE__FERNET_KEY=${AIRFLOW_FERNET_KEY}
    depends_on:
      - postgres
      - airflow
    volumes:
      - airflow_dags:/opt/airflow/dags
      - airflow_logs:/opt/airflow/logs
    command: airflow scheduler
  
  minio:
    image: minio/minio:latest
    ports:
      - "9000:9000"      # API 포트
      - "9001:9001"      # 웹 콘솔 포트
    environment:
      - MINIO_ROOT_USER=${MINIO_ROOT_USER}
      - MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD}
    volumes:
      - minio_data:/data
    command: server /data --console-address ":9001"

volumes:
  postgres_data:
  postgres_mlflow_data:
  mlflow_data:
  airflow_dags:
  airflow_logs:
  minio_data:
```

## 환경 변수 관리

### .env 파일 구성

Docker Compose는 자동으로 `.env` 파일을 읽어 환경 변수를 로드합니다. 별도의 import 과정 없이 바로 사용할 수 있습니다.

```bash
# .env 파일
# PostgreSQL 설정
POSTGRES_PASSWORD=secure_airflow_password_123
POSTGRES_MLFLOW_PASSWORD=secure_mlflow_password_456

# MinIO 설정
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=secure_minio_password_789

# Airflow 보안 설정
AIRFLOW_FERNET_KEY=81HqDtbqAywKSOumSha3BhWNOdQ26slT6K0YaZeZyPs=

# Airflow 관리자 계정
AIRFLOW_ADMIN_USERNAME=admin
AIRFLOW_ADMIN_PASSWORD=secure_admin_password
AIRFLOW_ADMIN_EMAIL=admin@yourcompany.com
```

### 환경 변수 자동 로드 원리

Docker Compose는 다음 순서로 환경 변수를 처리합니다:

1. `docker-compose.yml`과 같은 디렉토리의 `.env` 파일 자동 탐색
2. `${VARIABLE_NAME}` 형태로 참조된 변수를 `.env` 파일에서 찾아 치환
3. 시스템 환경 변수와 `.env` 파일 중복 시 시스템 환경 변수 우선

## 보안 고려사항

### 핵심 보안 요소

1. **Fernet Key**: Airflow 연결 정보 암호화를 위한 필수 설정
2. **데이터베이스 분리**: 서비스별 독립적 인증 및 권한 관리
3. **환경 변수**: 설정 파일에 하드코딩된 비밀번호 제거
4. **네트워크 격리**: Docker 내부 네트워크를 통한 서비스 간 통신

### Fernet Key 생성

```bash
# Python으로 새로운 Fernet Key 생성
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

### .gitignore 설정

```bash
# .gitignore에 추가하여 민감한 정보 보호
.env
.env.local
.env.production
*.env
```

## 실행 및 검증

### 환경 구축

#### 1. 프로젝트 디렉토리 설정

```bash
# 원하는 위치에 MLOps 환경용 디렉토리 생성
mkdir mlops-environment
cd mlops-environment

# 현재 위치 확인
pwd
# 출력 예시: /home/user/mlops-environment
```

#### 2. 필수 파일 생성

**docker-compose.yml 파일 생성:**
```bash
# 텍스트 에디터로 docker-compose.yml 파일 생성
nano docker-compose.yml
# 또는
vim docker-compose.yml
# 또는
code docker-compose.yml  # VS Code 사용 시
```

**위에서 제공한 docker-compose.yml 내용을 복사하여 붙여넣기**

**.env 파일 생성:**
```bash
# 환경 변수 파일 생성
nano .env

# .env 파일에 다음 내용 작성:
POSTGRES_PASSWORD=secure_airflow_password_123
POSTGRES_MLFLOW_PASSWORD=secure_mlflow_password_456
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=secure_minio_password_789
AIRFLOW_FERNET_KEY=81HqDtbqAywKSOumSha3BhWNOdQ26slT6K0YaZeZyPs=
AIRFLOW_ADMIN_USERNAME=admin
AIRFLOW_ADMIN_PASSWORD=secure_admin_password
AIRFLOW_ADMIN_EMAIL=admin@yourcompany.com
```

#### 3. 파일 구조 검증

```bash
# 디렉토리 내용 확인
ls -la

# 예상 출력:
# total 16
# drwxr-xr-x  2 user user 4096 Jul  4 01:00 .
# drwxr-xr-x 20 user user 4096 Jul  4 01:00 ..
# -rw-r--r--  1 user user  xxx Jul  4 01:00 .env
# -rw-r--r--  1 user user  xxx Jul  4 01:00 docker-compose.yml

# 파일 내용 간단 확인
head -5 docker-compose.yml
head -3 .env
```

#### 4. Docker 환경 확인

```bash
# Docker 데몬 실행 상태 확인
docker --version
docker-compose --version

# Docker 서비스 상태 확인 (Linux의 경우)
sudo systemctl status docker

# Docker 권한 확인 (sudo 없이 실행되는지)
docker ps
```

#### 5. 환경 변수 로드 테스트

```bash
# Docker Compose가 .env 파일을 제대로 읽는지 확인
docker-compose config --services
# 출력: postgres, postgres-mlflow, mlflow, airflow, airflow-scheduler, minio

# 환경 변수 치환 결과 확인 (민감한 정보는 표시되지 않음)
docker-compose config | grep -A 3 "environment:"
```

#### 6. 서비스 시작

```bash
# 백그라운드에서 모든 서비스 시작
docker-compose up -d

# 실시간 로그를 보면서 시작 (디버깅용)
docker-compose up

# 특정 서비스만 시작
docker-compose up -d postgres postgres-mlflow
```

#### 7. 서비스 상태 모니터링

```bash
# 모든 컨테이너 상태 확인
docker-compose ps

# 예상 출력:
#           Name                         Command               State           Ports
# ----------------------------------------------------------------------------------------
# mlops_airflow_1            /usr/bin/dumb-init -- /entrypoint  Up      0.0.0.0:8080->8080/tcp
# mlops_airflow-scheduler_1  /usr/bin/dumb-init -- /entrypoint  Up      8080/tcp
# mlops_mlflow_1             bash -c pip install mlflow ...    Up      0.0.0.0:5000->5000/tcp
# mlops_minio_1              /usr/bin/docker-entrypoint ...    Up      0.0.0.0:9000->9000/tcp, 0.0.0.0:9001->9001/tcp
# mlops_postgres_1           docker-entrypoint.sh postgres     Up      5432/tcp
# mlops_postgres-mlflow_1    docker-entrypoint.sh postgres     Up      5432/tcp

# 특정 서비스 로그 확인
docker-compose logs -f mlflow
docker-compose logs -f airflow
docker-compose logs postgres

# 모든 서비스 로그 실시간 모니터링
docker-compose logs -f
```

#### 8. 초기화 확인

```bash
# 각 서비스가 정상적으로 초기화되었는지 확인
# MLflow 초기화 확인
curl http://localhost:5000/health
# 또는 브라우저에서 http://localhost:5000 접속

# MinIO 접속 확인
curl http://localhost:9001
# 또는 브라우저에서 http://localhost:9001 접속

# Airflow 초기화 확인 (시간이 좀 걸릴 수 있음)
curl http://localhost:8080/health
# 또는 브라우저에서 http://localhost:8080 접속
```

#### 9. 문제 해결

```bash
# 서비스 시작 실패 시 상세 로그 확인
docker-compose logs <service-name>

# 컨테이너 내부 접속하여 디버깅
docker-compose exec postgres bash
docker-compose exec mlflow bash

# 특정 서비스 재시작
docker-compose restart mlflow

# 전체 환경 정리 후 재시작
docker-compose down
docker-compose up -d
```

### 서비스 접속 및 확인

- **MLflow UI**: http://localhost:5000
- **Airflow UI**: http://localhost:8080
- **MinIO Console**: http://localhost:9001

### 환경 변수 적용 검증

```bash
# 실제 적용된 설정 확인 (민감한 정보는 마스킹됨)
docker-compose config

# 특정 서비스의 환경 변수 확인
docker-compose exec postgres env | grep POSTGRES

# 서비스 로그 확인
docker-compose logs mlflow
docker-compose logs airflow
```

## MLOps 워크플로우 활용

구축된 환경에서 가능한 MLOps 워크플로우:

### 1. 실험 관리
- MLflow를 통한 모델 실험 추적
- 하이퍼파라미터 및 메트릭 자동 로깅
- 모델 버전 관리 및 스테이지 관리

### 2. 데이터 파이프라인
- Airflow DAG를 통한 데이터 전처리 자동화
- 스케줄 기반 모델 훈련 파이프라인
- 배치 추론 및 모델 평가 자동화

### 3. 아티팩트 관리
- MinIO를 통한 온프레미스 객체 저장소 활용
- 대용량 데이터셋 및 모델 아티팩트 로컬 저장
- S3 호환 API 제공으로 클라우드 마이그레이션 용이성 확보

### 4. 메타데이터 중앙화
- PostgreSQL을 통한 모든 메타데이터 통합 관리
- 서비스별 독립적 데이터 스키마 유지
- 백업 및 복구 전략 구현

## 서비스별 세부 설정

### MLflow 최적화
```python
# MLflow 클라이언트 설정 예시
import mlflow

mlflow.set_tracking_uri("http://localhost:5000")
mlflow.set_experiment("my-experiment")

# 실험 로깅
with mlflow.start_run():
    mlflow.log_param("learning_rate", 0.01)
    mlflow.log_metric("accuracy", 0.95)
```

### Airflow DAG 개발
```python
# 간단한 MLOps DAG 예시
from airflow import DAG
from airflow.operators.bash_operator import BashOperator
from datetime import datetime

dag = DAG(
    'ml_training_pipeline',
    start_date=datetime(2025, 7, 4),
    schedule_interval='@daily'
)

train_task = BashOperator(
    task_id='train_model',
    bash_command='python /opt/airflow/dags/train.py',
    dag=dag
)
```

## 트러블슈팅

### 일반적인 문제 해결

1. **서비스 시작 실패**: `docker-compose logs <service_name>`으로 로그 확인
2. **환경 변수 미적용**: `.env` 파일 위치 및 문법 확인
3. **포트 충돌**: 기본 포트 변경 또는 실행 중인 서비스 종료
4. **데이터베이스 연결 실패**: PostgreSQL 컨테이너 상태 및 네트워크 확인

### 서비스 재시작

```bash
# 특정 서비스만 재시작
docker-compose restart mlflow

# 전체 환경 재구축
docker-compose down
docker-compose up -d
```

## 확장 및 개선 방향

이 기본 환경을 바탕으로 다음과 같은 확장이 가능합니다:

- **모니터링**: Prometheus + Grafana 추가
- **로깅**: ELK Stack 또는 Loki 통합
- **보안 강화**: HTTPS, OAuth 인증 구현
- **GPU 지원**: NVIDIA Docker 런타임 추가
- **클러스터 확장**: Kubernetes 마이그레이션

완전한 MLOps 환경이 구축되어 모델 개발부터 배포까지 전체 생명주기를 효율적으로 관리할 수 있습니다.