# 6. Domain-Specific Optimization

> 도메인별 성능 최적화 전략

## 개요

각 모델 타입은 고유한 병목 현상을 겪는다. 본 문서는 도메인별 최적화 요건과 기술 선택을 정의한다.

**핵심 원칙:**
- 추론 엔진 내부 수정 없이 설정/구성으로 해결
- 도메인 특화 도구 활용 (엔진이 제공하는 최적화 기능)
- 측정 기반 결정 (추측으로 최적화하지 않음)

---

## REQ-OPT-01: Vision (GPU Pre-processing)

### 문제
고해상도 이미지 디코딩 및 리사이징 과정에서 **CPU 병목** 발생

### 해결책

| 기술 | 역할 | 효과 |
|------|------|------|
| **NVIDIA DALI** | 전처리를 GPU에서 수행 | CPU→GPU 전송 병목 제거 |
| **TensorRT** | FP16 컴파일 | 추론 속도 2-3x 향상 |

### 적용 지침

```yaml
# 예시: Vision 모델 배포 설정
model_config:
  name: "image_classifier"
  platform: "tensorrt_plan"
  optimization:
    preprocessing: "dali"  # GPU 전처리 활성화
    precision: "fp16"      # FP16 추론
```

### 성능 목표
- 이미지 전처리: < 5ms (GPU)
- 추론 지연: < 20ms (FP16)

---

## REQ-OPT-02: RecSys/Tabular (Low Latency Feature)

### 문제
추천/분류 모델 추론 시 **피처 조회 I/O 병목**

### 해결책

| 기술 | 역할 | 효과 |
|------|------|------|
| **RAPIDS FIL** | XGBoost/LightGBM GPU 추론 | CPU 대비 10-100x 빠름 |
| **Redis/Feast** | Feature Store | DB I/O 제거 |

### 적용 지침

```yaml
# 예시: RecSys 모델 배포 설정
model_config:
  name: "product_recommender"
  platform: "fil"  # RAPIDS FIL 백엔드
  feature_store:
    backend: "feast"
    cache: "redis"
    max_latency: "5ms"
```

### 성능 목표
- 피처 조회: < 5ms (Redis)
- 추론 지연: < 10ms (GPU)

---

## REQ-OPT-03: Audio/General (Dynamic Batching)

### 문제
입력 길이가 가변적 (오디오, 문장)하여 **GPU 활용률 저하**

### 해결책

| 기술 | 역할 | 효과 |
|------|------|------|
| **Triton Dynamic Batching** | 요청 자동 묶음 | GPU 활용률 극대화 |
| **Padding Strategy** | 가변 길이 처리 | 배치 효율성 |

### 적용 지침

```protobuf
# 예시: Triton config.pbtxt
dynamic_batching {
  max_queue_delay_microseconds: 50000  # 50ms 내 요청 묶음
  preferred_batch_size: [8, 16, 32]
}
```

### 성능 목표
- 배칭 지연: < 50ms
- GPU 활용률: > 80%

---

## REQ-OPT-04: LLM (Generative Optimization)

### 문제
자동회귀 생성 특성상 **토큰당 순차 생성으로 느림**

### 해결책

| 기술 | 역할 | 효과 |
|------|------|------|
| **Speculative Decoding** | Draft 모델로 예측 후 검증 | 2-3x 속도 향상 |
| **Semantic Caching** | 유사 질문 캐싱 | 중복 연산 제거 |
| **Continuous Batching** | vLLM/TGI 내장 | 처리량 증가 |

### 적용 지침

```yaml
# 예시: vLLM 서빙 설정
vllm_config:
  model: "meta-llama/Llama-2-7b"
  speculative_decoding:
    enabled: true
    draft_model: "meta-llama/Llama-2-1b"  # 작은 Draft 모델
  semantic_cache:
    enabled: true
    backend: "redis"
    similarity_threshold: 0.95
```

### 성능 목표
- 첫 토큰 지연 (TTFT): < 200ms
- 토큰 생성 속도: > 30 tokens/sec

---

## 도메인별 기술 스택 요약

| 도메인 | 추론 엔진 | 전처리 | 최적화 기술 |
|--------|----------|--------|-------------|
| **Vision** | Triton + TensorRT | NVIDIA DALI | FP16, GPU decode |
| **RecSys/Tabular** | Triton + RAPIDS FIL | - | Feature Store |
| **Audio/NLP** | Triton | - | Dynamic Batching |
| **LLM** | vLLM / TGI | - | Speculative, Caching |

---

## 측정 및 검증

### 필수 메트릭

| 메트릭 | 설명 | 임계값 |
|--------|------|--------|
| `latency_p99` | 99% 지연시간 | 도메인별 목표 |
| `gpu_utilization` | GPU 사용률 | > 70% |
| `cache_hit_rate` | 캐시 적중률 | > 80% (LLM) |
| `batch_size_avg` | 평균 배치 크기 | > 4 |

### 최적화 전 체크리스트

- [ ] 현재 병목 지점 프로파일링 완료
- [ ] 베이스라인 성능 측정 완료
- [ ] 도메인별 최적화 기술 적용
- [ ] A/B 테스트로 효과 검증

---

## Related

- [05_model.md](./05_model.md) - 모델 레이어 아키텍처
- [ADR-003: Thin Wrapper 패턴](../99_decisions/003_thin_wrapper.md) - 엔진 최적화는 엔진에 위임
- [01_strategy.md](../01_overview/01_strategy.md) - 개발 경계 정의
