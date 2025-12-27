# ADR-006: Worker 리소스 분리 및 병렬 처리 전략

> **Status**: Proposed
> **Date**: TBD
> **Author**: TBD
> **Context**: ADR-003(Temporal 채택) 후속 결정 — Worker 구성 및 리소스 분리 구체화
> **Supersedes**: N/A

---

## TL;DR (한 줄 요약)

> (작성 예정) GPU/CPU Worker 분리, Task Queue 라우팅, 병렬 처리 전략을 결정한다.

---

## 배경: ADR-003에서 위임된 결정

ADR-003에서 다음 개념이 언급되었다:

```
| Temporal 개념 | 우리 시스템에서의 의미 | 가치 |
|--------------|----------------------|------|
| Task Queue   | 모델별/리소스별 큐 분리 | GPU/CPU 워커 분리 가능 |
| Worker       | 파이프라인 실행 프로세스 | 수평 확장, 장애 격리 |
```

**본 ADR에서 결정할 사항:**
- Task Queue 기반 리소스 분리 전략
- Worker 병렬 처리 및 슬롯 관리

---

## 결정해야 할 사항 (TODO)

### 1. Task Queue 분리 전략

| 대안 | 고려 사항 |
|------|----------|
| 리소스 기반 분리 (GPU/CPU) | 하드웨어 특성에 따른 분리 |
| 모델 기반 분리 | 모델별 전용 큐 (대용량 모델 격리) |
| 파이프라인 단계 기반 분리 | Pre/Inference/Post 각각 분리 |
| 우선순위 기반 분리 | 긴급/일반 작업 분리 |

### 2. Worker 구성 전략

| 항목 | 결정 필요 사항 |
|------|---------------|
| **Worker 유형** | GPU Worker / CPU Worker / Hybrid Worker |
| **슬롯 관리** | Workflow 슬롯 vs Activity 슬롯 분리 |
| **동시성 제어** | Worker당 최대 동시 실행 Activity 수 |
| **리소스 할당** | Worker 프로세스당 GPU 메모리 할당 |

### 3. 병렬 처리 전략

| 항목 | 결정 필요 사항 |
|------|---------------|
| **Activity 병렬화** | 독립 Activity 동시 실행 전략 |
| **배치 처리** | 다중 요청 배치 실행 vs 개별 실행 |
| **파이프라인 병렬화** | 다중 파이프라인 동시 실행 제어 |

### 4. 스케일링 전략

| 항목 | 결정 필요 사항 |
|------|---------------|
| **수평 확장** | Worker 인스턴스 추가 기준 |
| **GPU 활용률** | 유휴 GPU 감지 및 재할당 |
| **Cold Start 대응** | GPU Worker 웜업 전략 |
| **Sticky Execution** | 캐시 활용을 위한 동일 Worker 라우팅 |

### 5. 장애 격리 및 복구

| 항목 | 결정 필요 사항 |
|------|---------------|
| **Worker 장애 감지** | Heartbeat 주기, 타임아웃 설정 |
| **GPU OOM 대응** | 메모리 부족 시 재시도/라우팅 전략 |
| **작업 재분배** | 장애 Worker의 작업 다른 Worker로 이전 |

---

## 관련 문서 (Related)

- [ADR-003: 추론 작업 실행 모델](./003_Inference_Task_Execution_Model_and_State_Management_Strategy.md) — Temporal 채택, Worker 개념 정의
- [Temporal Task Queue 문서](https://docs.temporal.io/task-queue) — 공식 문서
- [Temporal Worker 튜닝](https://temporal.io/blog/an-introduction-to-worker-tuning) — 성능 최적화

---

## 변경 이력 (Changelog)

| 날짜 | 작성자 | 변경 내용 |
|------|--------|----------|
| 2024-12-27 | System Architect | 초안 작성 — 결정 필요 사항 목록화 |
