# AI Serving Foundation - Project Context

## Core Philosophy

**Variable vs Invariant 분리**
- Variable: 모델 (도메인별 최적화)
- Invariant: 인프라 (공통 기반)

**Thin Wrapper Pattern**
- 추론 엔진(vLLM, Triton)을 감싸는 얇은 레이어
- 엔진 내부 수정 없음

## Core Competency

| 우리가 하는 것 | 구현 |
|---------------|------|
| Workflow Orchestration | Temporal |
| Universal Interface | OpenAI API Compatible |
| Gatekeeping | Rate Limit, Auth, Quota |

| 우리가 안 하는 것 | 담당 |
|------------------|------|
| 추론 엔진 최적화 | vLLM, Triton |
| GPU 저수준 관리 | Kubernetes |

## Documentation

```
docs/
├── README.md              # 로드맵
├── 00_reports/            # 분석 리포트
├── 01_overview/           # 프로젝트 개요
│   ├── 01_strategy.md     # WHY
│   └── 02_srs.md          # WHAT
├── 02_architecture/       # 시스템 구조
│   ├── 01_infrastructure.md
│   ├── 02_network.md
│   ├── 03_gateway.md
│   ├── 04_worker.md
│   └── 05_model.md
├── 03_development/        # 개발 가이드
│   ├── 01_setup.md
│   ├── 02_testing.md
│   └── 03_contributing.md
├── 04_operations/         # 운영 가이드
│   ├── 01_temporal.md
│   ├── 02_deployment.md
│   └── 03_monitoring.md
└── 99_decisions/          # ADR
    └── 000_template.md
```

## Key Requirements

| ID | Summary | Priority |
|----|---------|----------|
| REQ-API-01 | 비동기, Job ID | P0 |
| REQ-API-02 | 스트리밍 | P0 |
| REQ-ORC-01 | Temporal Workflow | P0 |
| REQ-INF-01 | OpenAI Compatible | P0 |
| REQ-INF-04 | Backend Agnostic | P0 |

## Quick Reference

| 상황 | 문서 |
|------|------|
| 프로젝트 이해 | `docs/01_overview/` |
| 시스템 구조 | `docs/02_architecture/` |
| 개발 시작 | `docs/03_development/01_setup.md` |
| 배포/운영 | `docs/04_operations/` |
| 설계 결정 | `docs/99_decisions/` |
