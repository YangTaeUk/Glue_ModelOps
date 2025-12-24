# 03. Development

> 개발 가이드

## 문서 구성

| 문서 | 내용 |
|------|------|
| [01_setup.md](./01_setup.md) | 환경 설정 |
| [02_testing.md](./02_testing.md) | 테스트 가이드 |
| [03_contributing.md](./03_contributing.md) | 기여 가이드 |

## 빠른 시작

```bash
# 1. 저장소 클론
git clone <repo-url>
cd ai-serving

# 2. 환경 설정
make setup

# 3. 개발 서버 실행
make dev

# 4. 테스트 실행
make test
```

## 기술 스택

| Layer | Technology |
|-------|------------|
| API | FastAPI, Pydantic |
| Orchestration | Temporal |
| Inference | vLLM, Triton |
| Monitoring | Prometheus, Grafana |

## 디렉토리 구조

```
src/
├── api/          # FastAPI 애플리케이션
├── workflows/    # Temporal Workflow
├── adapters/     # 추론 엔진 Adapter
├── models/       # Pydantic 스키마
└── utils/        # 유틸리티
```
