# 02. Architecture

> 시스템 구조

## 전체 구조

```
Client
  │
  ▼
┌─────────────────┐
│ GATEWAY: Nginx  │  SSL, Rate Limit, Routing
└─────────────────┘
  │
  ▼
┌─────────────────┐
│ APP: FastAPI    │  Validation, Async Job
└─────────────────┘
  │
  ▼
┌─────────────────┐
│ ORCH: Temporal  │  State, Queue, Retry
└─────────────────┘
  │
  ▼
┌─────────────────┐
│ WORKER          │  Activity 실행
└─────────────────┘
  │
  ▼
┌─────────────────────────────────────┐
│ MODEL: Inference Adapters           │
│ LLM(vLLM) | Vision(Triton) | RecSys │
└─────────────────────────────────────┘
```

## 문서 구성

| 문서 | 내용 |
|------|------|
| [01_infrastructure.md](./01_infrastructure.md) | 레이어 구조, 컴포넌트 |
| [02_network.md](./02_network.md) | 통신 프로토콜, 포트 |
| [03_gateway.md](./03_gateway.md) | Nginx Gateway 설계 |
| [04_worker.md](./04_worker.md) | Temporal Worker 설계 |
| [05_model.md](./05_model.md) | 추론 엔진 연동 |
| [06_optimization.md](./06_optimization.md) | 도메인별 성능 최적화 |

## 핵심 패턴

**Thin Wrapper**: 추론 엔진을 감싸는 얇은 레이어

**Immutable/Variable**:
- Immutable: Nginx → FastAPI → Temporal
- Variable: 도메인별 Adapter
