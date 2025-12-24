# 04. Operations

> 운영 가이드

## 문서 구성

| 문서 | 내용 |
|------|------|
| [01_temporal.md](./01_temporal.md) | Workflow, Retry, Queue |
| [02_deployment.md](./02_deployment.md) | 배포 전략, Circuit Breaker |
| [03_monitoring.md](./03_monitoring.md) | Metrics, Logging, Alerting |

## 운영 원칙

**Immutable Infrastructure**
- 배포 시 기존 인스턴스 교체
- 설정 변경 = 새 배포

**Observability**
- Metrics: Prometheus
- Logs: Structured JSON
- Traces: OpenTelemetry

**Resilience**
- Circuit Breaker로 장애 격리
- Temporal로 자동 복구
- Spot Instance 대응
