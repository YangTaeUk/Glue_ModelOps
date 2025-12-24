# ADR-002: 통신 프로토콜 표준화

> **Status**: Accepted
> **Date**: 2024-12-24
> **Deciders**: AI Serving Foundation Team

## Context

AI 모델 기반 기능을 제공하는 서비스에서 외부 클라이언트와의 통신 방식을 표준화해야 한다.

**운영 환경 전제:**
- 서버 내 Docker(Compose) 기반
- Edge Gateway: Nginx OSS 단독 (ADR-001 참조)
- 실제 장시간/고비용 작업은 Temporal Worker가 수행
- 외부 API는 FastAPI가 제공

**문제 정의:**
- AI 기능(LLM 토큰 생성, 음성 처리 등)은 작업 시간이 가변적 (수초~수분)
- 중간 진행 상황(토큰/진행률/로그) 전달 필요
- 네트워크 환경(기업망/모바일)에 따라 특정 프로토콜이 불안정할 수 있음
- 단일 통신 방식 의존은 운영 리스크 증가

## Decision Drivers

- **폴링 회피**: 짧은 간격 반복 조회 시 트래픽 폭증 및 서버 비용 증가
- **스트리밍 안정성**: LLM 토큰 스트리밍에 적합한 채널 필요
- **운영 단순화**: 구현/운영이 단순하고 디버깅이 용이해야 함
- **유연성**: 모델/기능이 정해지지 않은 상태에서 과도한 스펙 정의 회피

## Decision

### 표준 통신 채널 (4채널)

| 채널 | 목적 | 원칙 |
|------|------|------|
| **Sync HTTP** | 매우 짧은 작업 즉시 응답 | 서버 타임아웃 범위 내(3~10초)에서만 제공 |
| **Webhook Callback** | 완료 통지, 서버-서버 연동 | result_ref 및 상태 메타데이터 전달에 집중 |
| **SSE** | 토큰/진행률/중간 이벤트 전송 | **기본 스트리밍 채널** |
| **WebSocket** | 양방향 상호작용 필요 시 | 선택적 제공, SSE로 충족 안 될 때만 |

### 폴링(Polling) 처리 방침

- **폴링 기반 UX는 제외** (트래픽 폭증 위험)
- 운영 안정성 확보를 위해 **최소 상태 조회 API는 유지**
  - 예: `GET /v1/jobs/{job_id}`
  - 제한 정책: Retry-After, ETag(304), Rate Limit, 백오프 권장

### API 계약 방향 (스키마는 추후 정의)

현재 시점에서는 **엔드포인트 역할만 정의**, 구체적 스키마/이벤트 타입은 모델 확정 후 결정:

```
POST   /v1/jobs              # 작업 생성 → 202 + job_id
GET    /v1/jobs/{job_id}     # 상태 조회 (비상구)
GET    /v1/jobs/{job_id}/events   # SSE 스트림
GET    /v1/jobs/{job_id}/result   # 결과 조회
POST   /v1/jobs/{job_id}/cancel   # 취소
```

> **팀장 코멘트**: "모델도 안 정해진 상태에서 이벤트 타입(progress, token 등)을 확정하는 건 무리수. 지금은 채널 역할과 원칙만 잡고, 구체적 스펙은 기능 개발하면서 예시 코드/문서 가이드로 채워가자."

## Consequences

### Positive

- SSE 기본 채널로 폴링 트래픽 억제
- HTTP 기반이라 구현/운영 단순
- 재연결(Last-Event-ID) 활용으로 안정성 확보 가능
- 유연한 스펙으로 향후 모델/기능에 맞게 확장 용이

### Negative

- SSE는 단방향 → 양방향 필요 시 WebSocket 추가 구현 필요
- 구체적 이벤트 타입/스키마가 현재 미정의 → 개발 시 점진적 정의 필요
- Webhook 보안 스펙(HMAC, idempotency) 추후 정의 필요

## 후속 작업 (Next Steps)

구체적 스펙은 모델/기능 확정 후 별도 문서로 정의:

1. `/v1/jobs` 요청/응답 스키마 확정
2. 상태 전이(State Machine) 및 오류코드 표준화
3. SSE 이벤트 스키마 및 Last-Event-ID 재연결 규칙
4. Webhook 서명/재시도/idempotency 규격
5. Nginx OSS SSE/WS 프록시 설정 템플릿

## 대안 검토 (Rejected Options)

| 대안 | 기각 사유 |
|------|----------|
| 폴링 기반 표준 | 짧은 간격 폴링 시 트래픽 폭증, UX 품질 저하 |
| WebSocket 단독 | 기업망/모바일 환경에서 차단/불안정, 운영 부담 |
| SSE 단독 | 향후 양방향 요구 대비 WS를 선택 옵션으로 유지 |

## Related

- [ADR-001: Gateway 선택](./001_gateway_nginx.md) - Nginx OSS 결정
- [02_architecture/02_network.md](../02_architecture/02_network.md) - 통신 프로토콜
- [02_architecture/03_gateway.md](../02_architecture/03_gateway.md) - Nginx 스트리밍 설정
