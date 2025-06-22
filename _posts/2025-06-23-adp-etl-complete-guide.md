---
layout: post
title: "ADP 데이터분석전문가 2과목 - ETL(Extract, Transform, Load) 완전 가이드"
date: 2025-06-23
categories: [ADP, 데이터분석전문가]
tags: [ADP, ETL, 데이터처리, 데이터웨어하우스, 스테이징]
---

# ADP ETL 완전 가이드

## 🎯 핵심 개념 요약

**ETL(Extract, Transform, Load)**은 데이터 웨어하우스(DW), 운영 데이터 스토어(ODS), 데이터 마트(DM)의 핵심 요소로, 데이터의 이동과 변환을 담당하는 핵심 프로세스입니다.

### 📌 ETL의 3단계

1. **추출(Extraction)**: 데이터 소스로부터 데이터 획득
2. **변형(Transformation)**: 데이터 클렌징, 형식 변환, 표준화, 통합
3. **적재(Loading)**: 변형이 완료된 데이터를 특정 목표 시스템에 적재

---

## 🔍 ETL 상세 프로세스

### Step 0: Interface
- 다양한 이기종 DBMS 등 데이터 소스로부터 데이터를 획득하기 위한 각기 다른 인터페이스

### Step 1: Staging ETL
- 데이터 소스로부터 Transaction Data 획득 작업 수행
- **Staging Table**에 저장
- 임시 저장 영역의 역할

### Step 2: Profiling ETL
- Staging Table에서 **데이터 특성을 식별하고 품질 측정**
- 데이터 품질 분석 수행

### Step 3: Cleansing ETL
- 다양한 규칙들로 프로파일링된 데이터 보정
- **데이터 정제 및 품질 향상**

### Step 4: Integration ETL
- 이름, 값, 구조 등 **데이터 충돌을 해소**하고 클렌징된 데이터 통합
- 데이터 표준화 수행

### Step 5: De-normalizing ETL
- 운영 보고서 생성, DW 또는 DM에 데이터 적재를 위해 **데이터 비정규화** 수행
- 분석 및 조회 성능 최적화

---

## 🏗️ ODS(Operation Data Store) 구성

### 정의와 목적
- **정의**: 다양한 데이터 소스들로부터 데이터를 추출, 통합한 DB
- **목적**: 주로 실시간 또는 실시간 근접 Transaction 또는 하위 수준의 데이터를 저장하기 위해 설계됨

### Layered ODS의 구성

#### Interface Layer
- 다양한 데이터 소스로부터 데이터 획득
- OLEDB, ODBC, FTP, Real Time OLAP, 데이터 복제

#### Staging Layer
- 데이터 소스로부터 Transaction Data 추출되어 Staging Table에 저장
- Timestamp, Checksum 활용

#### Profiling Layer
- 데이터 품질 측정 및 분석
- 데이터 특성 식별

---

## 🎯 ADP 기출문제 및 예상문제

### 기출문제 유형 1: ETL 프로세스 순서
**문제**: ETL 프로세스의 올바른 단계 순서를 나열하시오.

**정답**: Interface → Staging → Profiling → Cleansing → Integration → De-normalizing

### 기출문제 유형 2: 각 단계의 역할
**문제**: Profiling ETL 단계의 주요 역할은?

**정답**: 스테이징 테이블에서 데이터 특성을 식별하고 품질을 측정하는 단계

### 예상문제 1: ODS 구성요소
**문제**: ODS(Operation Data Store)의 Layered 구성에서 Interface Layer의 주요 기능을 설명하시오.

**정답**: 다양한 데이터 소스(OLEDB, ODBC, FTP, Real Time OLAP 등)로부터 데이터를 획득하는 계층

### 예상문제 2: 데이터 변환 유형
**문제**: ETL의 Transform 단계에서 수행되는 주요 작업들을 3가지 이상 나열하시오.

**정답**: 
- 데이터 클렌징
- 형식 변환
- 표준화
- 통합
- 비즈니스 룰 적용

### 예상문제 3: 스테이징 테이블
**문제**: Staging Table의 주요 목적과 특징을 설명하시오.

**정답**: 
- 목적: 데이터 소스로부터 추출된 Transaction Data를 임시 저장
- 특징: 임시적 성격, 데이터 변환 전 중간 저장소 역할

---

## 💻 Python ETL 예시 코드

```python
import pandas as pd
import sqlite3
from datetime import datetime

class ETLProcessor:
    def __init__(self, source_db, target_db):
        self.source_db = source_db
        self.target_db = target_db
        
    def extract(self, query):
        """Step 1: 데이터 추출"""
        try:
            conn = sqlite3.connect(self.source_db)
            data = pd.read_sql_query(query, conn)
            conn.close()
            print(f"추출 완료: {len(data)}건의 데이터")
            return data
        except Exception as e:
            print(f"추출 오류: {e}")
            return None
    
    def profiling(self, data):
        """Step 2: 데이터 프로파일링"""
        profile_result = {
            'row_count': len(data),
            'null_count': data.isnull().sum().to_dict(),
            'data_types': data.dtypes.to_dict(),
            'duplicates': data.duplicated().sum()
        }
        print(f"프로파일링 결과: {profile_result}")
        return profile_result
    
    def cleansing(self, data):
        """Step 3: 데이터 클렌징"""
        # 중복 제거
        data_cleaned = data.drop_duplicates()
        
        # 결측값 처리
        for column in data_cleaned.columns:
            if data_cleaned[column].dtype == 'object':
                data_cleaned[column] = data_cleaned[column].fillna('Unknown')
            else:
                data_cleaned[column] = data_cleaned[column].fillna(0)
        
        print(f"클렌징 완료: {len(data_cleaned)}건의 데이터")
        return data_cleaned
    
    def transform(self, data):
        """Step 4: 데이터 변환 및 통합"""
        # 데이터 타입 변환
        if 'date_column' in data.columns:
            data['date_column'] = pd.to_datetime(data['date_column'])
        
        # 파생 변수 생성
        data['processed_date'] = datetime.now()
        
        # 데이터 표준화 (예: 문자열 대문자 변환)
        for column in data.select_dtypes(include=['object']).columns:
            data[column] = data[column].str.upper()
        
        print("변환 완료")
        return data
    
    def load(self, data, table_name):
        """Step 5: 데이터 적재"""
        try:
            conn = sqlite3.connect(self.target_db)
            data.to_sql(table_name, conn, if_exists='replace', index=False)
            conn.close()
            print(f"적재 완료: {table_name} 테이블에 {len(data)}건 저장")
            return True
        except Exception as e:
            print(f"적재 오류: {e}")
            return False
    
    def run_etl_pipeline(self, extract_query, target_table):
        """전체 ETL 파이프라인 실행"""
        print("=== ETL 파이프라인 시작 ===")
        
        # Extract
        raw_data = self.extract(extract_query)
        if raw_data is None:
            return False
        
        # Profiling
        self.profiling(raw_data)
        
        # Cleansing
        cleaned_data = self.cleansing(raw_data)
        
        # Transform
        transformed_data = self.transform(cleaned_data)
        
        # Load
        success = self.load(transformed_data, target_table)
        
        print("=== ETL 파이프라인 완료 ===")
        return success

# 사용 예시
if __name__ == "__main__":
    # ETL 프로세서 초기화
    etl = ETLProcessor('source.db', 'target.db')
    
    # ETL 파이프라인 실행
    query = "SELECT * FROM customer_data WHERE created_date >= '2025-01-01'"
    etl.run_etl_pipeline(query, 'processed_customers')
```

---

## 📋 ETL 단계별 체크리스트

### ✅ Extract (추출) 단계
- [ ] 데이터 소스 식별 및 연결 확인
- [ ] 추출 범위 및 조건 설정
- [ ] 데이터 추출 성공 여부 확인
- [ ] 스테이징 영역에 임시 저장

### ✅ Transform (변환) 단계
- [ ] 데이터 프로파일링 수행
- [ ] 데이터 품질 이슈 식별
- [ ] 클렌징 규칙 적용
- [ ] 데이터 표준화 및 통합
- [ ] 비즈니스 룰 적용

### ✅ Load (적재) 단계
- [ ] 목표 시스템 연결 확인
- [ ] 데이터 적재 전략 수립
- [ ] 적재 성공 여부 확인
- [ ] 데이터 무결성 검증

---

## 🎓 시험 대비 핵심 포인트

### 반드시 암기해야 할 내용
1. **ETL 3단계**: Extract → Transform → Load
2. **ETL 세부 프로세스**: Interface → Staging → Profiling → Cleansing → Integration → De-normalizing
3. **ODS 구성**: Interface Layer, Staging Layer, Profiling Layer
4. **각 단계별 주요 기능과 목적**

### 자주 출제되는 문제 유형
- ETL 프로세스의 순서와 각 단계의 역할
- ODS의 구성요소와 특징
- 스테이징 테이블의 목적과 활용
- 데이터 품질 관리 방법

### 실무 연계 포인트
- 데이터 웨어하우스 구축 프로젝트에서의 ETL 역할
- 빅데이터 환경에서의 ETL 도구 활용
- 실시간 데이터 처리와 배치 처리의 차이점

---

## 📚 추가 학습 자료

- 한국데이터산업진흥원 공식 가이드북
- 데이터 웨어하우스 설계 및 구축 관련 서적
- 실무 ETL 도구 (Talend, Informatica, SSIS 등) 학습

**💡 Tip**: ETL은 데이터 엔지니어링의 핵심 개념이므로 실제 프로젝트 사례와 연결하여 학습하면 더욱 효과적입니다!