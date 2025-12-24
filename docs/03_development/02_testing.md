# 2. Testing

> 테스트 가이드

## 테스트 구조

```
tests/
├── unit/           # 단위 테스트
├── integration/    # 통합 테스트
├── e2e/            # E2E 테스트
└── conftest.py     # 공통 fixture
```

## 실행

```bash
# 전체 테스트
pytest

# 단위 테스트만
pytest tests/unit/

# 커버리지 포함
pytest --cov=src --cov-report=html

# 특정 테스트
pytest tests/unit/test_adapters.py -k "test_vllm"
```

## 단위 테스트

```python
# tests/unit/test_adapters.py
import pytest
from src.adapters.vllm import VLLMAdapter

@pytest.fixture
def adapter():
    return VLLMAdapter(base_url="http://mock:8080")

async def test_predict(adapter, mocker):
    mocker.patch.object(adapter.client, "create", return_value=mock_response)
    result = await adapter.predict(mock_request)
    assert result.content == "Hello"
```

## 통합 테스트

```python
# tests/integration/test_workflow.py
import pytest
from temporalio.testing import WorkflowEnvironment

@pytest.fixture
async def env():
    async with await WorkflowEnvironment.start_local() as env:
        yield env

async def test_inference_workflow(env):
    result = await env.client.execute_workflow(
        InferenceWorkflow.run,
        test_request,
        id="test-workflow",
        task_queue="test-queue",
    )
    assert result.status == "completed"
```

## E2E 테스트

```python
# tests/e2e/test_api.py
import httpx

async def test_inference_e2e():
    async with httpx.AsyncClient() as client:
        # 요청
        resp = await client.post("/v1/inference", json=request)
        job_id = resp.json()["job_id"]

        # 폴링
        while True:
            status = await client.get(f"/v1/jobs/{job_id}")
            if status.json()["status"] == "completed":
                break

        # 결과 확인
        result = await client.get(f"/v1/jobs/{job_id}/result")
        assert "content" in result.json()["output"]
```

## Related

- [01_setup.md](./01_setup.md) - 환경 설정
