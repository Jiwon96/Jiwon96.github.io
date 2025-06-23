---
layout: post
title: "TensorFlow GRU 모델을 TFLite로 변환할 때 TensorListReserve 오류 해결하기"
date: 2025-06-23 09:13:00 +0900
categories: AI
tags: [TensorFlow, TFLite, GRU, 모델변환, 딥러닝, 음성데이터]
---

# TensorFlow GRU 모델을 TFLite로 변환할 때 TensorListReserve 오류 해결하기

TensorFlow의 GRU(Gated Recurrent Unit) 모델을 TensorFlow Lite로 변환하다 보면 `TensorListReserve` 관련 오류를 만날 수 있습니다. 이 글에서는 이 문제의 원인과 해결 방법을 알아보겠습니다.

## 문제 상황

```python
import os
import tensorflow as tf

cur_path = os.getcwd()
file_name="model_bs64_f1_0.983"
f_name = os.path.join(cur_path, file_name)
converter = tf.lite.TFLiteConverter.from_saved_model(f_name)

converter.optimizations = [tf.lite.OpsSet.TFLITE_BUILTINS]
converter.target_spec.supported_types = [tf.float16]

tflite_model = converter.convert()  # 여기서 오류 발생!
```

오류 메시지:
```
'tf.TensorListReserve' op requires element_shape to be static during TF Lite transformation pass
failed to legalize operation 'tf.TensorListReserve' that was explicitly marked illegal
```

## 원인 분석

GRU와 같은 RNN 계열 모델은 내부적으로 `TensorListReserve` 연산을 사용합니다. 이 연산은 동적 크기의 텐서 리스트를 다루는데, TFLite는 정적(static) 형태만 지원하기 때문에 변환 과정에서 충돌이 발생합니다.

## 해결 방법

### 방법 1: SELECT_TF_OPS 사용 (권장)

```python
import os
import tensorflow as tf

cur_path = os.getcwd()
file_name = "model_bs64_f1_0.983"
f_name = os.path.join(cur_path, file_name)

converter = tf.lite.TFLiteConverter.from_saved_model(f_name)

# TensorFlow 연산도 함께 지원하도록 설정
converter.target_spec.supported_ops = [
    tf.lite.OpsSet.TFLITE_BUILTINS,
    tf.lite.OpsSet.SELECT_TF_OPS  # 핵심!
]
converter.target_spec.supported_types = [tf.float16]

# 텐서 리스트 연산 비활성화
converter._experimental_lower_tensor_list_ops = False

tflite_model = converter.convert()

# 저장
output_path = os.path.join(cur_path, f"{file_name}.tflite")
with open(output_path, "wb") as f:
    f.write(tflite_model)
```

### 방법 2: 대표 데이터셋으로 최적화

```python
def representative_data_gen():
    # 음성 데이터 형태의 샘플 데이터 생성
    for _ in range(100):
        # 음성 데이터: [batch_size, sequence_length]
        yield [tf.random.normal([1, 32000])]

converter = tf.lite.TFLiteConverter.from_saved_model(f_name)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.representative_dataset = representative_data_gen
converter.target_spec.supported_ops = [
    tf.lite.OpsSet.TFLITE_BUILTINS,
    tf.lite.OpsSet.SELECT_TF_OPS
]
```

### 방법 3: 모델 구조 변경 (근본적 해결)

GRU 대신 TFLite에서 더 잘 지원되는 레이어로 교체:
- SimpleRNN
- LSTM (일부 제한 있음)
- 1D Convolution + Attention

## 음성 데이터 처리 시 추가 팁

음성 데이터(길이 32000)를 다룰 때는 다음 사항들도 고려해보세요:

```python
# 음성 데이터 전용 설정
converter = tf.lite.TFLiteConverter.from_saved_model(f_name)
converter.target_spec.supported_ops = [
    tf.lite.OpsSet.TFLITE_BUILTINS,
    tf.lite.OpsSet.SELECT_TF_OPS
]

# 음성 데이터는 보통 float32가 더 안정적
converter.target_spec.supported_types = [tf.float32]

# 긴 시퀀스 처리를 위한 설정
converter._experimental_lower_tensor_list_ops = False
converter.allow_custom_ops = True  # 필요시
```

## 주의사항

- `SELECT_TF_OPS`를 사용하면 TFLite 모델 크기가 증가할 수 있습니다
- 모바일 배포 시 성능에 영향을 줄 수 있으니 테스트가 필요합니다
- 가능하다면 모델 설계 단계에서 TFLite 호환성을 고려하는 것이 좋습니다

## 마무리

TensorFlow의 동적 연산과 TFLite의 정적 제약 사이의 간극에서 발생하는 대표적인 문제입니다. `SELECT_TF_OPS`를 활용하면 대부분 해결되지만, 배포 환경에 따라 성능 최적화를 위한 추가 고려사항이 있을 수 있습니다.

---

*Tags: TensorFlow, TFLite, GRU, 모델변환, 딥러닝*