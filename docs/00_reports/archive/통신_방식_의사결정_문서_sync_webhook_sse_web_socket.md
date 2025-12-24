# 통신 방식 의사결정 문서 (Sync · Webhook · SSE · WebSocket)

작성일: 2025-12-24

## 1. 목적
본 문서는 AI 모델 기반 기능을 제공하는 서비스에서 **외부 클라이언트와의 통신 방식**을 표준화하기 위해, 다음 4가지 채널의 역할을 정의하고(동기/비동기/스트리밍), **폴링(Polling)을 기본 UX에서 제외**하는 의사결정을 문서화한다.

대상 환경 전제는 다음과 같다.
- 서버 내 Docker(Compose) 기반 운영
- Edge Gateway는 Nginx OSS 단독 구성
- 실제 장시간/고비용 작업은 Temporal Worker가 수행
- 외부 API는 FastAPI가 제공

---

## 2. 배경 및 문제 정의
AI 기능(예: LLM 토큰 생성, 음성 처리, 장시간 파이프라인)은
- 작업 시간이 가변적이고(수초~수분)
- 중간 진행 상황(토큰/진행률/로그)을 전달해야 하며
- 네트워크 환경(기업망/모바일)과 보안 정책으로 인해 특정 프로토콜(WebSocket)이 항상 안정적으로 동작하지 않을 수 있다.

따라서 단일 통신 방식에 의존하는 설계는 운영 리스크를 높인다.

---

## 3. 의사결정(Decision)
### 3.1 표준 통신 채널
본 시스템은 다음 4가지 통신 채널을 **표준 제공**한다.

1) **동기(Sync HTTP)**
- 목적: 매우 짧은 작업을 즉시 응답
- 원칙: 서버 타임아웃 범위 내(예: 3~10초 수준)에서만 제공
- 비고: 장시간 AI 작업의 기본 방식으로 사용하지 않음

2) **비동기 콜백(Webhook Callback)**
- 목적: 완료 통지 및 서버-서버 연동
- 원칙: 콜백은 최종 결과 자체가 아니라 **result_ref(URI/Key) 및 상태 메타데이터** 전달에 집중
- 필수 보안: 서명(HMAC), 재시도 정책, idempotency 지원

3) **SSE(Server-Sent Events) 스트리밍**
- 목적: 토큰 단위/진행률/중간 이벤트를 **서버→클라이언트**로 안정적으로 전송
- 원칙: 토큰 스트리밍(LLM) 및 progress/event 전달의 **기본 채널**
- 이유: HTTP 기반으로 구현/운영이 단순하며, 재연결(Last-Event-ID)로 안정성을 확보하기 용이

4) **WebSocket 스트리밍(선택/확장)**
- 목적: 양방향 상호작용이 필요한 기능(실시간 제어/대화형 상호작용 등)
- 원칙: SSE로 충족되지 않는 요구가 있을 때 제공
- 비고: 커넥션 유지/동시 접속 관리 비용이 증가할 수 있으므로, 초기에는 선택적 채널로 운영


### 3.2 폴링(Polling) 처리 방침
- **폴링 기반 UX는 제외**한다(짧은 간격 반복 조회로 트래픽 폭증 위험).
- 다만 운영 안정성 확보를 위해 **최소 상태 조회 API는 유지**한다.
  - 예: `GET /v1/jobs/{job_id}`
  - 제한 정책: Retry-After/ETag(304)/Rate Limit/백오프 권장으로 폭주 방지

---

## 4. 역할 분담(Responsibility Split)
### 4.1 Edge Gateway (Nginx OSS)
- TLS 종료, 라우팅, 기본 Rate Limit, 요청 크기 제한, CORS/보안 헤더
- SSE/WebSocket 경로에 대한 프록시 설정(버퍼링/타임아웃/keep-alive) 적용

### 4.2 API Gateway (FastAPI)
- 외부 계약(API Contract) 제공: 요청/응답 스키마, 버전(/v1), 오류코드
- 인증/인가, 입력 검증, 멱등성(Idempotency)
- 작업 생성/조회/취소, 스트리밍 엔드포인트(SSE/WS) 제공

### 4.3 Orchestration/Execution (Temporal)
- 워크플로우 상태 관리(재시도/타임아웃/우선순위)
- 실제 작업 수행은 Worker가 담당
- 원칙: 토큰 단위 이벤트를 Temporal 히스토리에 과도하게 적재하지 않음

---

## 5. API 계약(Contract) 초안
본 통신 표준을 만족하기 위한 최소 엔드포인트 세트는 아래와 같다.

### 5.1 작업 생성
- `POST /v1/jobs`
  - 응답: `202 Accepted` + `job_id` + `status=QUEUED` + `events_url` + `result_url`

### 5.2 상태 조회(비상구)
- `GET /v1/jobs/{job_id}`
  - 응답: 상태(QUEUED/RUNNING/SUCCEEDED/FAILED/CANCELED), 진행률, 현재 단계, 요약 메시지
  - 정책: ETag/Retry-After/Rate Limit 적용

### 5.3 결과 조회
- `GET /v1/jobs/{job_id}/result`
  - 응답: 결과 메타데이터 또는 `result_ref`(URL/Key)

### 5.4 취소
- `POST /v1/jobs/{job_id}/cancel`

### 5.5 SSE 이벤트 스트림(기본)
- `GET /v1/jobs/{job_id}/events` (SSE)
  - 이벤트 타입(최소):
    - `progress`: 단계/퍼센트
    - `token`: 토큰/부분 출력
    - `message`: 로그/상태 메시지
    - `result`: 완료 및 result_ref
    - `error`: 오류 요약
  - 재연결: Last-Event-ID 기반 캐치업(지원 권장)
  - 효율: 토큰 이벤트는 배치/스로틀링 정책 적용 가능

### 5.6 WebSocket(선택)
- `WS /v1/jobs/{job_id}/ws` 또는 `WS /v1/ws` (구현 방식은 추후 확정)

### 5.7 Webhook(비동기 완료 통지)
- 클라이언트 제공 `callback_url`로 서버가 전송
  - 페이로드: job_id, status, result_ref, timestamps, signature
  - 재시도: 고정/지수 백오프 정책
  - 중복 방지: idempotency_key 또는 delivery_id

---

## 6. 효율성 및 운영 고려사항
1) **스트리밍 우선**: 토큰/진행률 전달은 SSE를 기본으로 하여 폴링 트래픽을 억제
2) **이벤트 저장소(권장)**: SSE는 연결이 끊길 수 있으므로, 이벤트를 외부 채널(Redis Streams 등)에 기록하여 재연결/캐치업 가능하게 설계
3) **스로틀링/배치**: 토큰 단위 이벤트는 네트워크/CPU 부하를 유발할 수 있으므로, 50~200ms 단위 배치 전송 등 정책을 고려
4) **장애 대응**: Webhook 실패/WS 차단 환경을 대비해 상태 조회 API를 유지

---

## 7. 대안 검토(Alternatives)
- **폴링 기반 표준**
  - 기각 사유: 짧은 간격 폴링 시 트래픽 폭증 및 서버 비용 증가, UX 품질 저하
- **WebSocket 단독**
  - 기각 사유: 기업망/모바일 환경에서 차단/불안정 가능, 운영 부담 증가
- **SSE 단독**
  - 보류: 단방향 스트리밍에는 충분하나, 향후 양방향 상호작용 요구가 생길 수 있어 WS를 선택 옵션으로 유지

---

## 8. 수용 기준(Acceptance Criteria)
- 동기 API는 지정된 타임아웃 범위 내에서만 제공되고, 그 외 작업은 비동기 방식으로 전환된다.
- 모든 비동기 작업은 `job_id`로 추적 가능하며, 상태 조회 API로 현재 상태를 확인할 수 있다.
- LLM 등 토큰 스트리밍은 SSE를 기본으로 제공하며, 끊김 후 재연결 시 이벤트가 유실되지 않도록(또는 최소화하도록) 설계된다.
- Webhook은 서명 검증 및 재시도/중복 방지 규격을 갖춘다.
- 과도한 폴링/조회 요청은 Edge Gateway 및 API 계층에서 Rate Limit으로 차단된다.

---

## 9. 후속 작업(Next Steps)
1) `/v1/jobs` 요청/응답 스키마 확정(작업 타입, 입력 형태, callback_url)
2) 상태 전이(State Machine) 확정 및 오류코드 표준화
3) SSE 이벤트 스키마 및 Last-Event-ID 재연결 규칙 확정
4) Webhook 서명/재시도/idempotency 규격 확정
5) Nginx OSS에서 SSE/WS에 필요한 프록시 설정 템플릿 확정

