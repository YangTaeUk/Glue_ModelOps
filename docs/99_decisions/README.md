# 99. Decisions

> Architecture Decision Records (ADR)

## 문서 구성

| 문서 | 상태 | 내용 |
|------|------|------|
| [000_template.md](./000_template.md) | - | ADR 작성 템플릿 |
| [001_gateway_nginx.md](./001_gateway_nginx.md) | Accepted | Gateway 선택: Nginx OSS 단독 |
| [002_communication_protocol.md](./002_communication_protocol.md) | Accepted | 통신 프로토콜: SSE 기본, 폴링 제외 |
| [003_thin_wrapper.md](./003_thin_wrapper.md) | Accepted | Thin Wrapper 패턴 및 개발 경계 정의 |

## ADR 목적

- 중요한 아키텍처 결정 기록
- 결정의 맥락과 이유 보존
- 미래 참조를 위한 문서화

## 작성 시점

- 기술 스택 선정
- 아키텍처 패턴 결정
- 주요 트레이드오프 결정
- 기존 결정 변경

## 네이밍 규칙

```
NNN_[상태]_[제목].md

예시:
001_accepted_temporal-workflow.md
002_superseded_redis-cache.md
003_proposed_grpc-protocol.md
```

## 상태

| 상태 | 설명 |
|------|------|
| proposed | 검토 중 |
| accepted | 채택됨 |
| deprecated | 더 이상 유효하지 않음 |
| superseded | 다른 ADR로 대체됨 |
