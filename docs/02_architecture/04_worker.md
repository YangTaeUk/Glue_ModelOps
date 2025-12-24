# 4. Worker

> Temporal Worker 상세 설계

## 역할

| 기능 | 설명 |
|------|------|
| Workflow 실행 | Temporal Workflow 처리 |
| Activity 실행 | 전처리, 추론, 후처리 |
| Engine 연결 | vLLM, Triton 호출 |
| 상태 관리 | Job 진행 상태 업데이트 |

## 구조

```
┌─────────────────────────────────────────┐
│ Temporal Worker                         │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ Workflow Executor                 │  │
│  │  • InferenceWorkflow              │  │
│  │  • BatchWorkflow                  │  │
│  └───────────────────────────────────┘  │
│                  │                      │
│                  ▼                      │
│  ┌───────────────────────────────────┐  │
│  │ Activity Executor                 │  │
│  │  • preprocess()                   │  │
│  │  • infer()                        │  │
│  │  • postprocess()                  │  │
│  └───────────────────────────────────┘  │
│                  │                      │
│                  ▼                      │
│  ┌───────────────────────────────────┐  │
│  │ Inference Adapters                │  │
│  │  ┌─────────┐  ┌─────────────────┐ │  │
│  │  │ vLLM    │  │ Triton          │ │  │
│  │  │ Adapter │  │ Adapter         │ │  │
│  │  └─────────┘  └─────────────────┘ │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

## Workflow 구현

```python
@workflow.defn
class InferenceWorkflow:
    @workflow.run
    async def run(self, request: InferRequest) -> InferResponse:
        # 1. 전처리
        processed = await workflow.execute_activity(
            preprocess,
            request,
            start_to_close_timeout=timedelta(seconds=30),
        )

        # 2. 추론
        result = await workflow.execute_activity(
            infer,
            processed,
            start_to_close_timeout=timedelta(minutes=5),
            retry_policy=RetryPolicy(
                maximum_attempts=3,
                backoff_coefficient=2.0,
            ),
        )

        # 3. 후처리
        return await workflow.execute_activity(
            postprocess,
            result,
            start_to_close_timeout=timedelta(seconds=10),
        )
```

## Task Queue

```python
QUEUES = {
    "high": "inference-high",      # 50% workers
    "normal": "inference-normal",  # 35% workers
    "low": "inference-low",        # 15% workers
}
```

## Retry Policy

| 에러 유형 | 정책 |
|----------|------|
| Transient | Exponential backoff, 3회 |
| OOM | 배치 축소 후 재시도 |
| Timeout | 1회 재시도 |
| Fatal | 즉시 실패 |

## Worker 설정

```python
worker = Worker(
    client=temporal_client,
    task_queue="inference-normal",
    workflows=[InferenceWorkflow],
    activities=[preprocess, infer, postprocess],
    max_concurrent_activities=10,
    max_concurrent_workflow_tasks=100,
)
```

## Related

- [../04_operations/01_temporal.md](../04_operations/01_temporal.md) - Temporal 운영
- [../01_overview/02_srs.md](../01_overview/02_srs.md) - REQ-ORC-01
