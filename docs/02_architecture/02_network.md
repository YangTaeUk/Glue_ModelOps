# 2. Network

> 통신 프로토콜 및 네트워크 구성

## 프로토콜

| Source | Target | Protocol | Port |
|--------|--------|----------|------|
| Client | Nginx | HTTPS | 443 |
| Nginx | FastAPI | HTTP | 8000 |
| FastAPI | Temporal | gRPC | 7233 |
| Worker | vLLM | HTTP (OpenAI) | 8080 |
| Worker | Triton | gRPC (KServe V2) | 8001 |

## 내부 통신

```
┌─────────────────────────────────────────────┐
│ Kubernetes Cluster                          │
│                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐ │
│  │ Nginx   │───▶│ FastAPI │───▶│Temporal │ │
│  │ :443    │    │ :8000   │    │ :7233   │ │
│  └─────────┘    └─────────┘    └─────────┘ │
│                                     │       │
│                      ┌──────────────┘       │
│                      ▼                      │
│              ┌─────────────┐                │
│              │   Worker    │                │
│              └──────┬──────┘                │
│                     │                       │
│       ┌─────────────┼─────────────┐         │
│       ▼             ▼             ▼         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
│  │  vLLM   │  │ Triton  │  │  Redis  │     │
│  │ :8080   │  │ :8001   │  │ :6379   │     │
│  └─────────┘  └─────────┘  └─────────┘     │
└─────────────────────────────────────────────┘
```

## Service Discovery

| Service | DNS | Type |
|---------|-----|------|
| API | api.svc.cluster.local | ClusterIP |
| Temporal | temporal.svc.cluster.local | ClusterIP |
| vLLM | vllm.svc.cluster.local | ClusterIP |
| Triton | triton.svc.cluster.local | ClusterIP |

## 보안

| Layer | 방식 |
|-------|------|
| 외부 통신 | TLS 1.3 |
| 내부 통신 | mTLS (Istio) |
| API 인증 | JWT Bearer Token |

## Timeout 설정

| Connection | Timeout |
|------------|---------|
| Client → Nginx | 300s |
| Nginx → FastAPI | 60s |
| FastAPI → Temporal | 30s |
| Worker → Engine | 300s (LLM), 30s (Vision) |

## Related

- [../01_overview/02_srs.md](../01_overview/02_srs.md) - REQ-SEC-01, REQ-SEC-02
