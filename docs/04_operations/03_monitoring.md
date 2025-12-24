# 3. Monitoring

> 모니터링 및 알림

## Metrics

| Metric | Type | Labels |
|--------|------|--------|
| inference_requests_total | Counter | model, status |
| inference_latency_seconds | Histogram | model, quantile |
| gpu_utilization | Gauge | gpu_id |
| queue_depth | Gauge | queue_name |

## GPU Monitoring (dcgm-exporter)

```yaml
DCGM_FI_DEV_GPU_UTIL      # GPU 사용률
DCGM_FI_DEV_FB_USED       # VRAM 사용량
DCGM_FI_DEV_PCIE_TX_BYTES # PCIe 대역폭
DCGM_FI_DEV_POWER_USAGE   # 전력 사용량
```

## Logging

```json
{
  "timestamp": "2025-01-01T00:00:00Z",
  "job_id": "job_abc123",
  "model": "llama-3-70b",
  "input_tokens": 50,
  "output_tokens": 200,
  "latency_ms": 1500,
  "status": "completed"
}
```

## Alerting

| Alert | Condition | Severity |
|-------|-----------|----------|
| High Latency | P99 > 2x target | Warning |
| Error Rate | > 5% | Critical |
| GPU OOM | VRAM > 95% | Critical |
| Queue Backlog | depth > 1000 | Warning |

## Dashboards

| Dashboard | 용도 |
|-----------|------|
| API Overview | 요청량, 지연, 에러율 |
| GPU Status | GPU 사용률, 메모리 |
| Queue Status | 큐 깊이, 처리량 |
| Model Performance | 모델별 성능 비교 |

## Related

- [../01_overview/02_srs.md](../01_overview/02_srs.md) - REQ-OPS-*
