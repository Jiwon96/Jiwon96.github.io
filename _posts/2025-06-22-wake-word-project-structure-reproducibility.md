---
layout: post
title: "Wake Word 프로젝트의 재현성과 추적성을 위한 폴더 구조 설계"
date: 2025-06-22 09:40:00 +0900
categories: [AI]
tags: [machine-learning, wake-word, project-structure, reproducibility, mlops]
---

# Wake Word 프로젝트의 재현성과 추적성을 위한 폴더 구조 설계

AI 모델 개발에서 가장 중요하면서도 간과되기 쉬운 부분이 바로 **재현성(Reproducibility)**과 **추적성(Traceability)**입니다. 특히 Wake Word Detection과 같은 실용적인 AI 프로젝트에서는 "3개월 전 실험을 정확히 재현하고 싶다" 또는 "가장 좋은 성능을 낸 설정이 무엇인가?" 같은 질문에 즉시 답할 수 있어야 합니다.

## Scripts 폴더 구조와 역할

```
scripts/
├── run_augmentation_pipeline.py   # 전체 증강 파이프라인 실행
├── augment_wake_words.py          # 대용량 증강 처리
├── train_wake_word_model.py       # 모델 학습 자동화
├── evaluate_model.py              # 모델 성능 평가
├── convert_to_tflite.py           # TFLite 변환
└── batch_experiments.py           # 배치 실험 실행
```

이러한 스크립트 중심 구조가 왜 필요하고, 어떻게 과학적 엄밀성을 ML 프로젝트에 도입할 수 있는지 살펴보겠습니다.

## 재현성(Reproducibility)에 도움이 되는 이유

### 1. 일관된 실행 환경

Scripts 폴더는 노트북과 달리 **동일한 조건에서 반복 실행**이 가능합니다. 노트북은 셀 실행 순서나 변수 상태에 따라 결과가 달라질 수 있지만, 스크립트는 매번 clean state에서 시작됩니다.

```bash
# 항상 동일한 결과를 보장하는 실행
python scripts/train_wake_word_model.py --seed 42 --config basic_config.yaml
```

### 2. 파라미터 외부화

모든 하이퍼파라미터와 설정을 YAML 파일로 분리하여, **코드 변경 없이 동일한 실험을 재현**할 수 있습니다. 예를 들어 `configs/augmentation/audio_aug_config.yaml`에 증강 설정이 저장되어 있어, 6개월 후에도 정확히 같은 증강을 적용할 수 있습니다.

```yaml
# configs/augmentation/basic_aug.yaml
augmentation:
  gaussian_noise:
    min_amplitude: 0.001
    max_amplitude: 0.015
    probability: 0.3
  time_stretch:
    min_rate: 0.8
    max_rate: 1.25
    probability: 0.3
```

### 3. 버전 관리 용이성

스크립트는 Git으로 버전 관리가 쉽고, **특정 커밋에서 정확히 같은 결과**를 얻을 수 있습니다. 노트북의 경우 출력 결과까지 Git에 저장되어 불필요한 변경사항이 많지만, 스크립트는 순수 코드만 관리됩니다.

## 추적성(Traceability)에 도움이 되는 이유

### 1. 실험 메타데이터 자동 생성

각 스크립트는 실행 시마다 **자동으로 메타데이터를 생성**합니다:
- 사용된 설정 파일 경로와 내용
- 실행 시간과 환경 정보
- 입력 데이터 체크섬과 버전
- 결과물 위치와 성능 지표

### 2. 실험 간 비교 가능성

`experiments/` 폴더에 각 실험별로 **완전한 실행 기록**이 저장됩니다:

```
experiments/
├── exp_20240101_1430/
│   ├── config.json          # 사용된 모든 설정
│   ├── training_log.txt     # 학습 로그
│   ├── model_performance.json # 성능 지표
│   └── plots/               # 시각화 결과
```

### 3. 데이터 계보 추적

각 데이터 변환 단계가 명확히 기록되어 **"이 모델은 어떤 데이터로 학습되었는가?"**를 정확히 추적할 수 있습니다:

```
원본 데이터 → 증강 설정 → 증강 데이터 → 학습 → 모델
```

## 효과적인 관리 방법

### 1. 설정 파일 중심 관리

```yaml
# configs/experiment/exp_basic.yaml
experiment:
  name: "wake_word_basic"
  description: "Basic augmentation with CNN model"
  
data:
  augmentation_config: "configs/augmentation/basic_aug.yaml"
  train_ratio: 0.7
  
model:
  architecture: "cnn"
  config_path: "configs/model/cnn_basic.yaml"
  
training:
  epochs: 100
  batch_size: 32
```

### 2. 실험 실행 명령 표준화

```bash
# 재현 가능한 실행 명령
python scripts/train_wake_word_model.py \
    --experiment_config configs/experiment/exp_basic.yaml \
    --seed 42 \
    --output_dir experiments/

# 특정 실험 재현
python scripts/train_wake_word_model.py \
    --experiment_id exp_20240101_1430 \
    --reproduce
```

### 3. 자동화된 추적 시스템

각 스크립트에 **추적 기능을 내장**하여:
- 실행 전: 환경 체크, 데이터 검증
- 실행 중: 로그 기록, 중간 결과 저장
- 실행 후: 결과 요약, 메타데이터 저장

### 4. 데이터 무결성 보장

- 입력 데이터 체크섬 검증
- 증강 데이터 품질 자동 검사
- 모델 파일 해시 저장

### 5. 성능 비교 자동화

```python
# tracking/performance_tracker.py가 자동으로
# - 모든 실험 결과 수집
# - 성능 트렌드 시각화
# - 최고 성능 모델 식별
```

## 실제 워크플로우에서의 활용

**개발 단계**: 노트북으로 탐색 → 검증된 코드를 스크립트로 이전

**실험 단계**: 스크립트로 체계적인 실험 수행 → 자동 추적

**운영 단계**: 스크립트로 안정적인 모델 학습 → 완전한 재현성

## 완전한 프로젝트 구조

```
wake-word-project/
├── data/
│   ├── raw/                 # 원본 오디오 데이터
│   ├── augmented/           # 증강된 데이터
│   ├── processed/           # 전처리된 데이터
│   └── splits/              # train/val/test 분할
├── notebooks/               # 탐색적 개발
│   ├── 01_audio_exploration.ipynb
│   ├── 02_audio_augmentation.ipynb
│   ├── 03_feature_extraction.ipynb
│   └── 04_wake_word_training.ipynb
├── utils/                   # 재사용 함수들
│   ├── audio_augmentation.py
│   ├── audio_data_loaders.py
│   └── evaluation_metrics.py
├── scripts/                 # 자동화 스크립트
│   ├── run_augmentation_pipeline.py
│   ├── train_wake_word_model.py
│   ├── evaluate_model.py
│   └── convert_to_tflite.py
├── configs/                 # 설정 파일들
│   ├── augmentation/
│   ├── model/
│   └── training/
├── experiments/             # 실험 관리
├── models/                  # 모델 저장소
└── tracking/                # 성능 추적
```

## 결론

스크립트 중심 구조는 **과학적 엄밀성**을 ML 프로젝트에 도입하여, 실험의 신뢰성과 결과의 재현성을 보장하는 기반이 됩니다. 이를 통해 다음과 같은 이점을 얻을 수 있습니다:

1. **완전한 재현성**: 몇 개월 후에도 정확히 같은 실험 재현 가능
2. **체계적 추적**: 모든 실험 과정과 결과의 완전한 기록
3. **효율적 비교**: 다양한 설정과 모델의 객관적 성능 비교
4. **안정적 운영**: 검증된 파이프라인을 통한 신뢰할 수 있는 모델 개발

Wake Word Detection과 같은 실용적인 AI 프로젝트에서는 이러한 체계적 접근이 특히 중요합니다. 실제 제품에 적용될 모델의 품질과 안정성을 보장하기 위해서는 개발 과정 자체가 과학적이고 재현 가능해야 하기 때문입니다.
