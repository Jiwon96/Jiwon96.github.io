---
layout: post
title: "MLflow Docker 환경에서 권한 오류 해결 과정"
date: 2025-07-06
categories: [docker]
tags: [mlflow, docker, minio, permissions]
---

# MLflow Docker 환경에서 권한 오류 해결 과정

## 문제 상황

Docker Compose로 MLflow 서버를 구성하고 TensorFlow 모델을 로깅하는 과정에서 권한 오류가 발생했습니다.

```
❌ 훈련 모듈 테스트 실패: [Errno 13] Permission denied: '/app'
```

## 오류 분석

### 초기 설정
```yaml
# docker-compose.yml
mlflow:
  build: 
    context: ./docker/mlflow
  ports:
    - "5000:5000"
  volumes:
    - mlflow_data:/app/mlruns
```

```dockerfile
# Dockerfile
CMD mlflow server \
    --backend-store-uri postgresql://mlflow:${POSTGRES_MLFLOW_PASSWORD}@postgres-mlflow:5432/mlflow \
    --default-artifact-root ./mlruns \
    --host 0.0.0.0 \
    --port 5000
```

### 단계별 문제 해결 시도

**1단계: 기본 MLflow 연결 테스트**
- 기본적인 parameter 로깅은 정상 작동
- 모델 저장 시에만 권한 오류 발생

**2단계: 볼륨 마운트 방식 변경**
```yaml
volumes:
  - ./mlruns:/app/mlruns  # 호스트 디렉토리 마운트로 변경
```

**3단계: 권한 문제 확인**
```bash
docker exec -it mlops-mlflow-1 ls -la /app/mlruns
# 결과: 소유자가 1000:1000 (일반사용자), MLflow는 root로 실행
```

**4단계: Dockerfile에서 절대 경로 사용**
```dockerfile
CMD mlflow server \
    --default-artifact-root file:///app/mlruns \
```

## 핵심 문제: MLflow의 구조적 한계

### 문제의 본질
MLflow의 아키텍처에서 클라이언트-서버 분리 환경의 구조적 한계가 드러났습니다:

1. **MLflow 서버**: Docker 컨테이너에서 실행
2. **MLflow 클라이언트**: 호스트에서 실행 
3. **Artifact 저장 방식**: 파일 시스템 경로를 직접 공유

### 구조적 문제점
```python
# 클라이언트가 모델 저장 요청
mlflow.tensorflow.log_model(model, "test_model")

# 서버 응답: "file:///app/mlruns에 저장하세요"
# 클라이언트 해석: 호스트의 /app/mlruns에 접근 시도 → 권한 오류
```

클라이언트가 서버의 파일 경로를 자신의 로컬 경로로 해석하면서 호스트의 `/app` 디렉토리에 접근하려고 시도하는 것이 근본 원인입니다.

## 최종 해결책: MinIO 활용

### MinIO 기반 Artifact Storage 구성
```yaml
# docker-compose.yml
mlflow:
  environment:
    - AWS_ACCESS_KEY_ID=${MINIO_ROOT_USER}
    - AWS_SECRET_ACCESS_KEY=${MINIO_ROOT_PASSWORD}
    - MLFLOW_S3_ENDPOINT_URL=http://minio:9000
  depends_on:
    - postgres-mlflow
    - minio
```

```dockerfile
# Dockerfile
RUN pip install mlflow psycopg2-binary boto3

CMD mlflow server \
    --backend-store-uri postgresql://mlflow:${POSTGRES_MLFLOW_PASSWORD}@postgres-mlflow:5432/mlflow \
    --default-artifact-root s3://mlflow/artifacts \
    --host 0.0.0.0 \
    --port 5000
```

### 해결 효과
- 파일 시스템 권한 문제 완전 해결
- 클라이언트-서버 간 네트워크 기반 통신으로 전환
- 확장성 및 운영 안정성 향상

## 교훈

MLflow를 Docker 환경에서 사용할 때는 로컬 파일 시스템보다는 S3 호환 스토리지(MinIO)를 사용하는 것이 구조적으로 더 안전하고 확장 가능한 접근법입니다.
