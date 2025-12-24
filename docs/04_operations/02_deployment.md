# 2. Deployment

> 배포 전략

## Shadow Deployment

```
Production Traffic
    │
    ├──────────────────┐
    ▼                  ▼ (복제)
┌─────────┐      ┌─────────┐
│ v1.0    │      │ v1.1    │
│ (Live)  │      │ (Shadow)│
└─────────┘      └─────────┘
    │                  │
    ▼                  ▼
  Response          Log Only
```

## Circuit Breaker

```yaml
circuit_breaker:
  error_threshold: 50%
  evaluation_window: 60s
  recovery_timeout: 30s
  fallback_model: "llama-3-8b"
```

| State | 동작 |
|-------|------|
| Closed | 정상 처리 |
| Open | 즉시 fallback |
| Half-Open | 일부 요청 테스트 |

## Auto-scaling

| Component | Metric | Scale |
|-----------|--------|-------|
| API | CPU > 70% | HPA |
| Worker | Queue depth > 100 | HPA |
| GPU | VRAM > 80% | Custom |

## Rollback

```bash
# 이전 버전으로 롤백
kubectl rollout undo deployment/api

# 특정 버전으로 롤백
kubectl rollout undo deployment/api --to-revision=2
```

## Related

- [../02_architecture/03_gateway.md](../02_architecture/03_gateway.md) - Gateway 설정
