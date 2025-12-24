# 1. Infrastructure

> 인프라 레이어 구조

## Immutable Layer (공통)

변하지 않는 인프라:

| Component | Role | Technology |
|-----------|------|------------|
| Gateway | SSL Termination, Rate Limit | Nginx |
| Application | Request Validation, Async | FastAPI |
| Orchestration | State Management, Queue | Temporal |
| Monitoring | Metrics Collection | Prometheus |
| Storage | Config, Cache | Redis, S3 |

### Nginx Gateway

```nginx
upstream api {
    server fastapi:8000;
}

server {
    listen 443 ssl;

    # Rate Limiting
    limit_req zone=api burst=20;

    location /v1/ {
        proxy_pass http://api;
    }
}
```

### FastAPI Application

```python
@app.post("/v1/inference")
async def inference(request: InferRequest) -> JobResponse:
    job_id = await temporal.start_workflow(
        InferenceWorkflow.run,
        request,
        id=generate_job_id(),
    )
    return JobResponse(job_id=job_id, status="queued")
```

## Variable Layer (도메인별)

도메인 특화 최적화:

| Domain | Preprocessing | Engine | Backend |
|--------|---------------|--------|---------|
| LLM | - | vLLM, SGLang | OpenAI API |
| Vision | DALI | Triton | TensorRT |
| RecSys | Feast | Triton | RAPIDS FIL |
| Audio | - | Triton | ONNX |

## 스케일링

| Component | Strategy | Trigger |
|-----------|----------|---------|
| Nginx | 수평 확장 | CPU > 70% |
| FastAPI | HPA | RPS > 1000 |
| Worker | Queue 기반 | Depth > 100 |
| GPU | 수직 우선 | VRAM > 80% |

## Related

- [../01_overview/02_srs.md](../01_overview/02_srs.md) - REQ-INF-01, REQ-INF-02
