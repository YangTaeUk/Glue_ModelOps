# 2. System Requirements (SRS)

> 무엇을 만드는가?

## 기술 스택

| Layer | Component | Technology |
|-------|-----------|------------|
| Gateway | Ingress | Nginx |
| App | API Server | FastAPI |
| Orchestration | Workflow | Temporal |
| Inference | Engine | vLLM, Triton, SGLang |
| Data | Metadata | PostgreSQL |
| Monitor | Observability | Prometheus, Grafana |

## 데이터 흐름

```
Client
   │
   ▼
┌─────────────────────────────────────────────┐
│ IMMUTABLE LAYER                              │
│  Nginx → FastAPI → Temporal → Worker        │
└─────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────┐
│ VARIABLE LAYER                               │
│  Vision(DALI) │ RecSys(Feast) │ LLM(vLLM)   │
└─────────────────────────────────────────────┘
```

## 기능 요건

### API Layer

| ID | 요건 | Priority |
|----|------|----------|
| REQ-API-01 | 비동기 처리, Job ID 반환 | P0 |
| REQ-API-02 | SSE/WebSocket 스트리밍 | P0 |
| REQ-API-03 | Pydantic 유효성 검증 | P1 |

### Orchestration

| ID | 요건 | Priority |
|----|------|----------|
| REQ-ORC-01 | Workflow 단위 정의 | P0 |
| REQ-ORC-02 | Retry Policy (Exponential, OOM) | P1 |
| REQ-ORC-03 | Priority Queueing | P1 |

### Inference

| ID | 요건 | Priority |
|----|------|----------|
| REQ-INF-01 | OpenAI API Compatible | P0 |
| REQ-INF-02 | KServe V2 Protocol | P1 |
| REQ-INF-03 | Ensemble Support | P1 |
| REQ-INF-04 | Backend Agnostic | P0 |

## 비기능 요건

### Performance

| ID | 요건 | 목표 |
|----|------|------|
| REQ-PERF-01 | Vision/RecSys P99 | < 50ms |
| REQ-PERF-02 | LLM TTFT | < 500ms |

### Operations

| ID | 요건 | Priority |
|----|------|----------|
| REQ-OPS-01 | GPU Monitoring (dcgm-exporter) | P1 |
| REQ-OPS-02 | Logging (파라미터, 토큰, 시간) | P1 |
| REQ-DEP-01 | Shadow Deployment | P2 |
| REQ-DEP-02 | Circuit Breaker | P1 |

### Security & Cost

| ID | 요건 | Priority |
|----|------|----------|
| REQ-SEC-01 | PII Redaction | P1 |
| REQ-COST-01 | Spot Instance Recovery | P2 |

## 우선순위 분포

| Priority | Count | 설명 |
|----------|-------|------|
| P0 | 5 | 핵심 (반드시) |
| P1 | 9 | 중요 (가능한) |
| P2 | 3 | 최적화 (점진적) |
