# 1. Setup

> 개발 환경 설정

## 요구사항

| Tool | Version |
|------|---------|
| Python | 3.11+ |
| Docker | 24.0+ |
| kubectl | 1.28+ |
| Make | 4.0+ |

## 설치

```bash
# Python 환경
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"

# Pre-commit hooks
pre-commit install
```

## 로컬 서비스

```bash
# Docker Compose로 의존성 실행
docker compose up -d temporal redis

# FastAPI 개발 서버
uvicorn src.api.main:app --reload --port 8000

# Temporal Worker
python -m src.workflows.worker
```

## 환경 변수

```bash
# .env.local
TEMPORAL_HOST=localhost:7233
REDIS_URL=redis://localhost:6379
VLLM_URL=http://localhost:8080/v1
LOG_LEVEL=DEBUG
```

## IDE 설정

### VS Code

```json
{
  "python.defaultInterpreterPath": ".venv/bin/python",
  "python.formatting.provider": "black",
  "editor.formatOnSave": true
}
```

## Makefile 명령어

| 명령어 | 설명 |
|--------|------|
| make setup | 초기 설정 |
| make dev | 개발 서버 실행 |
| make test | 테스트 실행 |
| make lint | 린트 검사 |
| make format | 코드 포맷팅 |

## Related

- [02_testing.md](./02_testing.md) - 테스트 가이드
