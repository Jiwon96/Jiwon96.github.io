---
layout: post
title: "Docker MLflow 환경에서 클라이언트 연결 오류 해결하기: Unable to locate credentials 에러 분석"
date: 2025-07-06 23:00:00 +0900
categories: [docker]
tags: [MLflow, Docker Compose, 환경변수, MinIO, 클라이언트-서버 연결]
---

## 문제 상황 소개

Docker Compose로 MLflow + MinIO + PostgreSQL 환경을 구성하고, 로컬에서 Python 스크립트를 실행할 때 다음과 같은 에러가 발생했습니다:

```
❌ 통합 테스트 실패: Unable to locate credentials
```

MLflow 서버는 정상적으로 동작하고 있었지만, validation.py에서 `mlflow.start_run()`을 실행할 때 자격 증명을 찾을 수 없다는 에러가 발생했습니다.

## 환경 구성 분석

### 서버 측 (Docker 컨테이너)
```yaml
mlflow:
  build: 
    context: ./docker/mlflow
  ports:
    - "5000:5000"
  environment:
    - POSTGRES_MLFLOW_PASSWORD=${POSTGRES_MLFLOW_PASSWORD}
    - AWS_ACCESS_KEY_ID=${MINIO_ROOT_USER}
    - AWS_SECRET_ACCESS_KEY=${MINIO_ROOT_PASSWORD}
    - MLFLOW_S3_ENDPOINT_URL=http://minio:9000
  depends_on:
    - postgres-mlflow
    - minio
```

### 클라이언트 측 (로컬 Ubuntu)
```python
def validate_data_quality(self, dataset, dataset_name="dataset"):
    # ... 데이터 검증 로직 ...
    
    # MLflow 로깅 - 여기서 에러 발생!
    with mlflow.start_run(run_name=f"data_validation_{dataset_name}"):
        mlflow.log_metric(f"{dataset_name}_total_samples", total_samples)
        # ...
```

### 네트워크 차이점
- **Docker 내부 통신**: `minio:9000` (서비스명으로 접근)
- **외부 접근**: `localhost:9000` (포트 포워딩을 통한 접근)

## 에러 발생 원인 3가지

### 1. MLflow 서버 위치 불명
MLflow 클라이언트는 기본적으로 로컬 `mlruns` 폴더를 사용합니다. Docker 컨테이너에서 실행 중인 MLflow 서버의 위치를 모르기 때문에 연결에 실패합니다.

### 2. S3 자격 증명 누락
`mlflow.start_run()` 내부에서 아티팩트(메트릭, 그래프 등)를 MinIO에 업로드하려 할 때, 클라이언트 환경에서 AWS 호환 자격 증명을 찾을 수 없어 실패합니다.

### 3. 엔드포인트 설정 부재
MLflow가 기본적으로 AWS S3에 연결을 시도하지만, 실제로는 로컬 MinIO에 연결해야 하므로 엔드포인트 설정이 필요합니다.

## 해결 방법: 환경변수 기반 설정

### .env 파일 활용
```env
# MLflow 설정
MLFLOW_TRACKING_URI=http://localhost:5000

# MinIO 설정
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=00000000
MLFLOW_S3_ENDPOINT_URL=http://localhost:9000
```

### python-dotenv 설치
```bash
pip install python-dotenv
```

## 실제 구현 코드

### Before: 설정 없는 코드 (에러 발생)
```python
import mlflow

def validate_data_quality(self, dataset, dataset_name="dataset"):
    # ... 데이터 검증 로직 ...
    
    # 에러 발생! MLflow 서버 위치와 S3 자격 증명을 모름
    with mlflow.start_run(run_name=f"data_validation_{dataset_name}"):
        mlflow.log_metric(f"{dataset_name}_total_samples", total_samples)
```

### After: 환경변수 로드가 포함된 코드
```python
import mlflow
import os
from dotenv import load_dotenv

# .env 파일에서 환경변수 로드
load_dotenv()

# MLflow 서버 연결 설정
mlflow.set_tracking_uri(os.getenv("MLFLOW_TRACKING_URI", "http://localhost:5000"))

# MinIO S3 자격 증명 설정
os.environ["AWS_ACCESS_KEY_ID"] = os.getenv("MINIO_ROOT_USER")
os.environ["AWS_SECRET_ACCESS_KEY"] = os.getenv("MINIO_ROOT_PASSWORD")
os.environ["MLFLOW_S3_ENDPOINT_URL"] = os.getenv("MLFLOW_S3_ENDPOINT_URL", "http://localhost:9000")

def validate_data_quality(self, dataset, dataset_name="dataset"):
    # ... 데이터 검증 로직 ...
    
    # 정상 동작!
    with mlflow.start_run(run_name=f"data_validation_{dataset_name}"):
        mlflow.log_metric(f"{dataset_name}_total_samples", total_samples)
```

### MLflow 설정 클래스 예제
```python
class MLflowConfig:
    def __init__(self):
        load_dotenv()
        self.setup_mlflow()
    
    def setup_mlflow(self):
        """MLflow 연결 설정"""
        mlflow.set_tracking_uri(os.getenv("MLFLOW_TRACKING_URI"))
        
        # S3 자격 증명 설정
        os.environ["AWS_ACCESS_KEY_ID"] = os.getenv("MINIO_ROOT_USER")
        os.environ["AWS_SECRET_ACCESS_KEY"] = os.getenv("MINIO_ROOT_PASSWORD")
        os.environ["MLFLOW_S3_ENDPOINT_URL"] = os.getenv("MLFLOW_S3_ENDPOINT_URL")
        
        print("MLflow 설정 완료!")
```

## 검증 및 테스트

### 1. 연결 확인
```python
# S3 연결 테스트
import boto3

s3 = boto3.client('s3', 
    endpoint_url=os.getenv("MLFLOW_S3_ENDPOINT_URL"),
    aws_access_key_id=os.getenv("MINIO_ROOT_USER"),
    aws_secret_access_key=os.getenv("MINIO_ROOT_PASSWORD"))

buckets = s3.list_buckets()
print('S3 연결 성공:', [b['Name'] for b in buckets['Buckets']])
```

### 2. MLflow UI 확인
브라우저에서 `http://localhost:5000`에 접속하여 실험 데이터가 정상적으로 로깅되는지 확인합니다.

### 3. 아티팩트 저장 확인
MinIO 웹 UI(`http://localhost:9001`)에서 `mlflow` 버킷에 아티팩트가 저장되는지 확인할 수 있습니다.

## 핵심 포인트

1. **클라이언트-서버 분리**: Docker 컨테이너의 MLflow 서버와 로컬 클라이언트는 별개의 환경
2. **보안 설정**: 환경변수를 통한 자격 증명 관리로 코드 보안 강화
3. **네트워크 이해**: Docker 내부 통신과 외부 접근의 차이점 파악 필요

이러한 설정을 통해 Docker 환경의 MLflow와 로컬 클라이언트 간의 원활한 연결을 구현할 수 있습니다.
