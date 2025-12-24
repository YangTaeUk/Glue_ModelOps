# 5. Model Serving

> 추론 엔진 연동 상세 설계

## Adapter Pattern

```python
class InferenceAdapter(Protocol):
    async def predict(self, request: InferRequest) -> InferResponse:
        """동기 추론"""
        ...

    async def stream(self, request: InferRequest) -> AsyncIterator[Token]:
        """스트리밍 추론"""
        ...

    async def health(self) -> bool:
        """헬스 체크"""
        ...
```

## vLLM Adapter

```python
class VLLMAdapter(InferenceAdapter):
    def __init__(self, base_url: str):
        self.client = AsyncOpenAI(base_url=base_url)

    async def predict(self, request: InferRequest) -> InferResponse:
        response = await self.client.chat.completions.create(
            model=request.model,
            messages=request.input["messages"],
            **request.parameters,
        )
        return InferResponse(
            content=response.choices[0].message.content,
            usage=response.usage,
        )

    async def stream(self, request: InferRequest) -> AsyncIterator[Token]:
        stream = await self.client.chat.completions.create(
            model=request.model,
            messages=request.input["messages"],
            stream=True,
            **request.parameters,
        )
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                yield Token(content=chunk.choices[0].delta.content)
```

## Triton Adapter

```python
class TritonAdapter(InferenceAdapter):
    def __init__(self, url: str):
        self.client = tritonclient.grpc.aio.InferenceServerClient(url)

    async def predict(self, request: InferRequest) -> InferResponse:
        inputs = [
            tritonclient.grpc.InferInput(
                "input", request.input.shape, "FP32"
            )
        ]
        inputs[0].set_data_from_numpy(request.input)

        response = await self.client.infer(
            model_name=request.model,
            inputs=inputs,
        )
        return InferResponse(output=response.as_numpy("output"))
```

## 엔진별 특성

| Engine | Protocol | Batch | Streaming |
|--------|----------|-------|-----------|
| vLLM | OpenAI API | 자동 | 지원 |
| SGLang | OpenAI API | 자동 | 지원 |
| Triton | KServe V2 | Dynamic | 미지원 |

## Model Registry

```python
ADAPTERS = {
    "llama-3-70b": VLLMAdapter("http://vllm:8080/v1"),
    "llama-3-8b": VLLMAdapter("http://vllm:8080/v1"),
    "resnet50": TritonAdapter("triton:8001"),
    "whisper": TritonAdapter("triton:8001"),
}

def get_adapter(model: str) -> InferenceAdapter:
    if model not in ADAPTERS:
        raise ModelNotFoundError(model)
    return ADAPTERS[model]
```

## Circuit Breaker

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.failures = 0
        self.threshold = failure_threshold
        self.timeout = recovery_timeout
        self.state = "closed"

    async def call(self, adapter, request):
        if self.state == "open":
            if time.time() - self.opened_at > self.timeout:
                self.state = "half-open"
            else:
                raise CircuitOpenError()

        try:
            result = await adapter.predict(request)
            self.failures = 0
            self.state = "closed"
            return result
        except Exception as e:
            self.failures += 1
            if self.failures >= self.threshold:
                self.state = "open"
                self.opened_at = time.time()
            raise
```

## Related

- [../01_overview/02_srs.md](../01_overview/02_srs.md) - REQ-INF-01, REQ-INF-02
