# 1. Temporal

> Workflow 운영

## Workflow 구조

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

        return result
```

## Retry Policy

| 에러 유형 | 정책 |
|----------|------|
| Transient | Exponential backoff, 3회 |
| OOM | 배치 축소 후 재시도 |
| Timeout | 1회 재시도 |
| Fatal | 즉시 실패 |

## Priority Queue

```python
QUEUES = {
    "high": "inference-high",      # 50% workers
    "normal": "inference-normal",  # 35% workers
    "low": "inference-low",        # 15% workers
}
```

## Spot Instance Recovery

```python
@workflow.defn
class SpotAwareWorkflow:
    @workflow.run
    async def run(self, request):
        try:
            return await self._execute(request)
        except SpotInterruptionError:
            workflow.continue_as_new(request)
```

## Related

- [../02_architecture/04_worker.md](../02_architecture/04_worker.md) - Worker 설계
