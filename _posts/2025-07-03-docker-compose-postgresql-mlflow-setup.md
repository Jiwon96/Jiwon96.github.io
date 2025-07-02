---
layout: post
title: "Docker Compose로 PostgreSQL + MLflow 환경 구축하기"
date: 2025-07-03 15:00:00 +0900
categories: [DOCKER]
tags: [Docker, Docker-Compose, PostgreSQL, MLflow, MLOps, 환경구축, 컨테이너]
---

## 들어가며

MLOps 환경을 구축하면서 PostgreSQL과 MLflow를 Docker로 관리하는 방법을 정리해보았습니다. 초보자도 쉽게 따라할 수 있도록 실제 질문과 답변 형식으로 작성했습니다.

## 1. Docker Compose 기본 구조

**Q: Docker에서 `service:`는 언제 사용하나요?**

Docker Compose에서 여러 컨테이너를 정의할 때 사용합니다.

```yaml
version: '3.8'
services:
  web:
    image: nginx:latest
  database:
    image: mysql:8.0
```

## 2. 이미지 빌드 vs 다운로드

**Q: `build: ./`는 무엇인가요?**

로컬에서 Dockerfile을 빌드할 때 사용합니다.
- `image: postgres:15` - 기존 이미지 사용
- `build: ./` - 현재 디렉토리의 Dockerfile로 빌드

**Q: 이미지가 로컬에 없으면 어떻게 하나요?**

Docker가 자동으로 다운로드합니다! `docker-compose up` 실행 시 필요한 이미지를 자동으로 받아옵니다.

## 3. 환경변수 설정

**Q: `environment:`는 언제 작성하나요?**

컨테이너 내부의 환경변수를 설정할 때 사용합니다.

```yaml
environment:
  POSTGRES_PASSWORD: 0000
  POSTGRES_USER: root
  POSTGRES_DB: KWS
```

## 4. 볼륨과 데이터 영속성

**Q: 마지막에 `volumes:`는 무엇인가요?**

데이터를 영구적으로 저장하기 위한 설정입니다.

```yaml
services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data
      
volumes:
  postgres_data:  # Docker가 자동으로 생성
```

**중요**: PostgreSQL 데이터 경로는 `/var/lib/postgresql/data`로 고정입니다.

## 5. 완성된 Docker Compose 파일

```yaml
version: "3"

services:
  postgres:
    image: postgres
    environment:
      POSTGRES_PASSWORD: 0000
      POSTGRES_USER: root
      POSTGRES_DB: KWS
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  mlflow:
    image: python:3.9
    command: >
      bash -c "
        pip install mlflow psycopg2-binary &&
        mlflow server 
        --backend-store-uri postgresql://root:0000@postgres:5432/KWS
        --default-artifact-root ./mlruns
        --host 0.0.0.0
        --port 5000
      "
    ports:
      - "5000:5000"
    depends_on:
      - postgres
    volumes:
      - mlflow_data:/app/mlruns

volumes:
  postgres_data:
  mlflow_data:
```

## 6. MLflow 명령어 상세 분석

**Q: MLflow 설정 부분을 설명해주세요.**

```yaml
mlflow:
  image: python:3.9
  command: >
    bash -c "
      pip install mlflow psycopg2-binary &&
      mlflow server 
      --backend-store-uri postgresql://root:0000@postgres:5432/KWS
      --default-artifact-root ./mlruns
      --host 0.0.0.0
      --port 5000
    "
```

### 단계별 해석

#### 1. 베이스 이미지
```yaml
image: python:3.9
```
- Python 3.9가 설치된 기본 Ubuntu 이미지 사용
- MLflow는 Python 패키지이므로 Python 환경이 필요

#### 2. 명령어 실행 구조
```yaml
command: >
  bash -c "..."
```
- `>`: YAML에서 여러 줄을 한 줄로 합치는 문법 (긴 명령어를 여러 줄로 나누어 가독성 향상)
- `bash -c`: 쉘 명령어를 실행
- 컨테이너가 시작되면 이 명령어들을 순서대로 실행

#### 3. 패키지 설치
```bash
pip install mlflow psycopg2-binary &&
```
- `mlflow`: MLflow 패키지 설치
- `psycopg2-binary`: PostgreSQL과 연결하기 위한 Python 드라이버
- `&&`: 첫 번째 명령어가 성공하면 다음 명령어 실행

#### 4. MLflow 서버 실행 옵션들

**`--backend-store-uri postgresql://root:0000@postgres:5432/KWS`**
- 메타데이터 저장소 지정
- 형식: `postgresql://사용자:비밀번호@호스트:포트/데이터베이스`
- `postgres`: Docker Compose에서 서비스 이름 (내부 네트워크에서 호스트명으로 사용)

**`--default-artifact-root ./mlruns`**
- 아티팩트(모델 파일, 플롯 등) 저장 위치
- `./mlruns`: 현재 디렉토리의 mlruns 폴더

**`--host 0.0.0.0`**
- 모든 IP 주소에서 접속 허용
- Docker 컨테이너 외부에서 접속 가능하게 함

**`--port 5000`**
- MLflow 웹 UI가 실행될 포트 번호

### 실행 순서

1. Python 3.9 컨테이너 시작
2. MLflow + PostgreSQL 드라이버 설치
3. PostgreSQL에 연결된 MLflow 서버 실행
4. `localhost:5000`에서 웹 UI 접속 가능

### command의 다른 작성 방법들

**배열 형태:**
```yaml
command: 
  - bash
  - -c
  - |
    pip install mlflow psycopg2-binary &&
    mlflow server --backend-store-uri postgresql://root:0000@postgres:5432/KWS --host 0.0.0.0 --port 5000
```

**멀티라인 문자열:**
```yaml
command: |
  bash -c "
    pip install mlflow psycopg2-binary &&
    mlflow server 
    --backend-store-uri postgresql://root:0000@postgres:5432/KWS
    --default-artifact-root ./mlruns
    --host 0.0.0.0
    --port 5000
  "
```

## 7. 실행 및 문제 해결

### 버전 호환성 문제
```bash
# 구버전 docker-compose 사용 시 에러 발생
docker-compose version 1.29.2  # 구버전

# 해결방법: 새로운 명령어 사용
docker compose up -d  # 하이픈 없음
```

### Docker 이미지 정리
```bash
# 모든 이미지 삭제
docker system prune -a --volumes -f

# 사용하지 않는 이미지만 삭제
docker image prune -a
```

## 8. 주요 포인트

1. **nginx 불필요**: MLflow 자체 웹서버 제공
2. **자동 데이터 저장**: MLflow 메타데이터가 PostgreSQL에 자동 저장
3. **포트 설정**: `localhost:5000`에서 MLflow UI 접속 가능
4. **볼륨 자동 생성**: Docker가 필요한 볼륨을 자동으로 생성

## 마무리

이 설정으로 MLOps 환경의 기본 인프라를 구축할 수 있습니다. PostgreSQL은 메타데이터를 저장하고, MLflow는 실험 추적을 담당하며, 모든 것이 Docker로 관리되어 환경 일관성을 보장합니다.

---

**실행 방법**
```bash
docker compose up -d
# http://localhost:5000 접속
```