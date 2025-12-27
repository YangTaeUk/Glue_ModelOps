# ADR-005: 실시간 스트리밍 채널 전략

> **Status**: Proposed
> **Date**: TBD
> **Author**: TBD
> **Context**: ADR-003(Dual Path Architecture) 후속 결정 — 스트리밍 채널 구체화
> **Supersedes**: N/A

---

## TL;DR (한 줄 요약)

> (작성 예정) Temporal을 우회하는 실시간 토큰 스트리밍 채널의 기술 선택과 아키텍처를 결정한다.

---

## 배경: ADR-003에서 위임된 결정

ADR-003에서 다음 원칙이 확정되었다:

```
"Temporal은 오케스트레이션 경로, 스트리밍은 별도 경로로 분리한다."

- Path 1 (Temporal): Job 상태, 단계 완료/실패 이벤트
- Path 2 (Direct):   토큰 단위 실시간 스트리밍 (Temporal 우회)
```

**본 ADR에서 결정할 사항:**
- Path 2 (Direct Streaming)의 구체적인 기술 선택 및 아키텍처

---

## 결정해야 할 사항 (TODO)

### 1. 클라이언트 통신 프로토콜

| 대안 | 고려 사항 |
|------|----------|
| SSE (Server-Sent Events) | 단방향, HTTP 호환, 브라우저 지원 |
| WebSocket | 양방향, 연결 관리 복잡 |
| gRPC Streaming | 고성능, 브라우저 직접 지원 제한 |

### 2. 내부 메시지 전달 메커니즘

| 대안 | 고려 사항 |
|------|----------|
| Redis Pub/Sub | 단순, 메시지 유실 가능 |
| Redis Streams | 메시지 보존, Consumer Group |
| Kafka | 고가용성, 운영 복잡도 |
| NATS | 경량, 클라우드 네이티브 |
| 직접 연결 | Activity → API 직접 푸시 |

### 3. 신뢰성 및 복구 전략

- 클라이언트 재연결 시 누락 토큰 복구 방안
- 메시지 순서 보장
- 백프레셔(Backpressure) 처리
- Activity 실패 시 스트리밍 상태 정리

### 4. 스케일 아웃 고려사항

- 다중 API 서버 환경에서 클라이언트 연결 라우팅
- Sticky Session vs 메시지 브로드캐스트
- Worker 수평 확장 시 스트리밍 채널 동기화

### 5. 모니터링 및 관측성

- 스트리밍 지연 시간 측정
- 연결 상태 모니터링
- 토큰 전송률 메트릭

---

## 관련 문서 (Related)

- [ADR-003: 추론 작업 실행 모델](./003_Inference_Task_Execution_Model_and_State_Management_Strategy.md) — Dual Path 원칙 정의
- [ADR-004: Universal Connector](./004_Universal_Connector_Strategy_for_Model_Integration.md) — Connector 스트리밍 인터페이스

---

## 변경 이력 (Changelog)

| 날짜 | 작성자 | 변경 내용 |
|------|--------|----------|
| 2024-12-27 | System Architect | 초안 작성 — 결정 필요 사항 목록화 |
