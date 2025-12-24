# Documentation Roadmap

> 상황별 문서 가이드

## 문서 구조

| # | 폴더 | 내용 |
|---|------|------|
| 00 | [reports/](./00_reports/) | 분석 리포트 |
| 01 | [overview/](./01_overview/) | 프로젝트 개요 |
| 02 | [architecture/](./02_architecture/) | 시스템 구조 |
| 03 | [development/](./03_development/) | 개발 가이드 |
| 04 | [operations/](./04_operations/) | 운영 가이드 |
| 99 | [decisions/](./99_decisions/) | ADR |

## 상황별 가이드

### "이 프로젝트가 뭔가요?"

| 문서 | 읽어야 할 때 |
|------|-------------|
| [01_overview/01_strategy.md](./01_overview/01_strategy.md) | 프로젝트 목적, 철학 |
| [01_overview/02_srs.md](./01_overview/02_srs.md) | 기능 요구사항, REQ-* ID |

### "어떻게 동작하나요?"

| 문서 | 읽어야 할 때 |
|------|-------------|
| [02_architecture/](./02_architecture/) | 전체 시스템 구조 |
| [02_architecture/03_gateway.md](./02_architecture/03_gateway.md) | Nginx, Rate Limit |
| [02_architecture/04_worker.md](./02_architecture/04_worker.md) | Temporal Worker |
| [02_architecture/05_model.md](./02_architecture/05_model.md) | 추론 엔진 연동 |

### "개발 시작하려면?"

| 문서 | 읽어야 할 때 |
|------|-------------|
| [03_development/01_setup.md](./03_development/01_setup.md) | 환경 설정 |
| [03_development/02_testing.md](./03_development/02_testing.md) | 테스트 작성 |
| [03_development/03_contributing.md](./03_development/03_contributing.md) | PR, 커밋 컨벤션 |

### "배포/운영하려면?"

| 문서 | 읽어야 할 때 |
|------|-------------|
| [04_operations/01_temporal.md](./04_operations/01_temporal.md) | Workflow 운영 |
| [04_operations/02_deployment.md](./04_operations/02_deployment.md) | 배포 전략 |
| [04_operations/03_monitoring.md](./04_operations/03_monitoring.md) | 모니터링, 알림 |

## Quick Reference

| 상황 | 참조 문서 |
|------|----------|
| 온보딩 | overview → architecture |
| 새 기능 개발 | overview/srs → development |
| 새 엔진 추가 | architecture/05_model |
| 배포 준비 | operations/02_deployment |
| 장애 대응 | operations/03_monitoring |
| 설계 변경 | decisions/000_template |
