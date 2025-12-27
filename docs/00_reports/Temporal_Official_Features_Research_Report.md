# Temporal 공식 기능 리서치 보고서

> **목적**: ADR-003 의사결정 문서의 기반 자료
> **조사 일자**: 2024-12-27
> **조사 범위**: Temporal.io 공식 문서 및 SDK 문서
> **신뢰도**: 높음 (공식 문서 기반)

---

## Executive Summary

Temporal은 **Durable Execution Platform**으로, 분산 시스템에서 장시간 실행되는 비즈니스 로직을 안정적으로 실행하기 위한 오케스트레이션 엔진이다. 핵심 가치는 개발자가 상태 관리, 재시도, 타임아웃 코드를 직접 작성하지 않고도 장애에 강건한 워크플로우를 구현할 수 있다는 것이다.

---

## 1. 핵심 개념 (Core Concepts)

### 1.1 Workflow

| 속성 | 설명 |
|------|------|
| **정의** | 비즈니스 로직의 순서를 정의하는 코드 함수 (YAML/XML이 아닌 실제 코드) |
| **언어 지원** | Go, Java, Python, TypeScript, .NET |
| **특징** | 결정론적(Deterministic) 실행 필수 |
| **상태 보존** | Event History에 모든 상태 자동 저장 |

**공식 정의:**
> "Conceptually, a workflow defines a sequence of steps. With Temporal, those steps are defined by writing code, known as a Workflow Definition."

**핵심 제약:**
- Workflow 코드는 **결정론적(Deterministic)**이어야 함
- 외부 I/O, 랜덤, 현재 시간 직접 호출 금지
- 모든 외부 작업은 Activity로 위임

### 1.2 Activity

| 속성 | 설명 |
|------|------|
| **정의** | 실패 가능성이 있는 비즈니스 로직을 캡슐화한 함수 |
| **용도** | 외부 서비스 호출, DB 조회, 파일 처리 등 |
| **재시도** | 기본 Retry Policy 자동 적용 |
| **상태** | Stateless (상태 없음) |

**공식 정의:**
> "An Activity is a method or function that encapsulates business logic prone to failure (e.g., calling a service that may go down)."

**Activity vs Workflow 역할 분리:**
- **Workflow**: 조정자 (Coordinator) — Activity 실행 순서 정의
- **Activity**: 실행자 (Executor) — 실제 작업 수행

### 1.3 Worker

| 속성 | 설명 |
|------|------|
| **정의** | Workflow/Activity를 실행하는 프로세스 |
| **위치** | Temporal Service 외부 (개발자가 운영) |
| **특징** | Task Queue를 폴링하여 작업 수신 |
| **확장** | 수평 확장 가능 (동일 Task Queue 폴링) |

**핵심 포인트:**
> "Worker Processes are external to a Temporal Service. Temporal Application developers are responsible for developing Worker Programs and operating Worker Processes."

**Worker가 하는 일:**
1. Task Queue 폴링
2. Workflow/Activity Task 수신
3. 코드 실행
4. 결과를 Temporal Service에 보고

### 1.4 Task Queue

| 속성 | 설명 |
|------|------|
| **정의** | Worker가 폴링하는 가상 큐 |
| **용도** | 로드 밸런싱, 작업 라우팅 |
| **특징** | 이름 기반, 동적 생성 |

**Task Routing 사용 사례:**
- GPU/CPU 워커 분리
- 파일 처리 (동일 호스트에서 후속 작업 실행)
- 대용량 데이터셋 캐싱 활용

---

## 2. Durable Execution 메커니즘

### 2.1 Event Sourcing 기반 상태 관리

```
┌─────────────────────────────────────────────────────────────────┐
│                   Temporal Durable Execution                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Workflow 실행 중:                                              │
│   ─────────────────                                              │
│   Step 1 완료 → Event 기록 → Persistence Store                  │
│   Step 2 완료 → Event 기록 → Persistence Store                  │
│   Step 3 실행 중 → Worker 크래시                                │
│                                                                  │
│   복구 시:                                                       │
│   ─────────                                                      │
│   Event History 로드 → Replay → Step 3 재실행                   │
│   (Step 1, 2 결과는 Event에서 복원, 재실행 없음)                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**공식 설명:**
> "Temporal uses durable event sourcing combined with idempotent execution. Workflow actions and decisions are logged durably, allowing Temporal to replay events to reach the exact state before failures."

### 2.2 Event History

| 속성 | 설명 |
|------|------|
| **정의** | Workflow Execution의 완전한 이벤트 로그 |
| **용도** | 상태 복구, 디버깅, 감사 |
| **저장소** | MySQL, PostgreSQL, Cassandra 등 |
| **특징** | Append-only, 불변 |

**핵심 가치:**
> "Temporal records the values returned by every single Activity call, which means the memory is completely visible and debuggable through Temporal tooling."

### 2.3 Replay 메커니즘

- Workflow가 재시작되면 Event History를 읽어 **동일한 결과**를 재생성
- 이미 완료된 Activity는 재실행하지 않음 (저장된 결과 사용)
- 결정론적 코드 요구의 이유

---

## 3. 타임아웃 체계 (Official Specifications)

### 3.1 Activity 타임아웃 (4가지)

| 타임아웃 | 기본값 | 목적 | 권장 사항 |
|---------|--------|------|----------|
| **Schedule-To-Close** | ∞ | 전체 Activity 실행 시간 제한 (재시도 포함) | Start-To-Close와 함께 설정 권장 |
| **Start-To-Close** | Schedule-To-Close와 동일 | 단일 Activity 실행 시간 제한 | **필수 설정 권장** |
| **Schedule-To-Start** | ∞ | Task Queue 대기 시간 제한 | 거의 사용 안 함, 메트릭 모니터링 권장 |
| **Heartbeat** | 없음 | Heartbeat 간격 제한 | 장시간 Activity에서 권장 |

**공식 권장:**
> "We strongly recommend setting a Start-To-Close Timeout."
> "The Temporal Server doesn't detect failures when a Worker loses communication with the Server or crashes. Therefore, the Temporal Server relies on the Start-To-Close Timeout to force Activity retries."

### 3.2 Heartbeat 상세

| 속성 | 설명 |
|------|------|
| **목적** | 장시간 Activity의 Worker 생존 확인 |
| **동작** | Worker → Temporal Service로 주기적 신호 전송 |
| **실패 시** | Activity 실패 처리 → Retry Policy에 따라 재시도 |

**사용 시나리오:**
- GPU 추론 (분 단위 실행)
- 대용량 파일 처리
- 외부 API 폴링

---

## 4. Retry Policy (Official Specifications)

### 4.1 Retry Policy 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|---------|------|--------|------|
| **Initial Interval** | Duration | 1초 | 첫 번째 재시도까지 대기 시간 |
| **Backoff Coefficient** | Decimal | 2.0 | 재시도 간격 증가 계수 |
| **Maximum Interval** | Duration | 100 × Initial | 최대 재시도 간격 |
| **Maximum Attempts** | Integer | 무제한 | 최대 재시도 횟수 |
| **Non-Retryable Errors** | String[] | 빈 배열 | 재시도하지 않을 에러 타입 |

### 4.2 재시도 간격 계산

```
retry_interval = min(Initial Interval × Backoff Coefficient^(retry_count), Maximum Interval)
```

**예시 (기본값 사용 시):**
- 1차 재시도: 1초 후
- 2차 재시도: 2초 후
- 3차 재시도: 4초 후
- ...
- N차 재시도: min(2^N, 100)초 후

### 4.3 기본 적용 규칙

| 대상 | 기본 Retry Policy |
|------|------------------|
| **Activity** | 자동 적용됨 |
| **Workflow** | 적용 안 됨 (명시적 설정 필요) |

---

## 5. 메시지 패싱 (Query, Signal, Update)

### 5.1 세 가지 메시지 유형

| 유형 | 방향 | 동기/비동기 | 용도 |
|------|------|------------|------|
| **Query** | 읽기 | 동기 | 현재 상태 조회 |
| **Signal** | 쓰기 | 비동기 | 외부 이벤트 전달 |
| **Update** | 쓰기 | 동기 | 상태 변경 + 응답 대기 |

### 5.2 Query

| 속성 | 설명 |
|------|------|
| **정의** | Workflow 상태를 읽는 읽기 전용 요청 |
| **특징** | Event History에 기록되지 않음 |
| **장점** | 완료된 Workflow에도 실행 가능 |

**공식 설명:**
> "Queries are read requests. They can read the current state of the Workflow but cannot block in doing so."

### 5.3 Signal

| 속성 | 설명 |
|------|------|
| **정의** | Workflow에 데이터를 전달하는 비동기 쓰기 요청 |
| **특징** | Event History에 기록됨 (내구성 보장) |
| **장점** | 응답을 기다리지 않음 (Fire-and-forget) |

**공식 설명:**
> "Signals provide a fully async and durable mechanism for providing data to a running workflow."

### 5.4 Update

| 속성 | 설명 |
|------|------|
| **정의** | 상태 변경 후 결과를 반환하는 동기 쓰기 요청 |
| **특징** | Worker 응답을 기다림 |
| **용도** | 변경 확인이 필요한 경우 |

### 5.5 Cancel

| 속성 | 설명 |
|------|------|
| **정의** | 실행 중인 Workflow를 취소 요청 |
| **동작** | `WorkflowExecutionCancelRequested` 이벤트 기록 |
| **특징** | Workflow가 정리 작업(cleanup) 수행 가능 |

---

## 6. Python SDK 특성

### 6.1 Async/Await 통합

**공식 설명:**
> "The workflow implementation basically turns async def functions into workflows backed by a distributed, fault-tolerant event loop."

```
특징:
- asyncio와 완전 통합
- Task 관리, Sleep, Cancellation이 asyncio 개념과 통합
- async def 함수가 분산 Workflow로 변환됨
```

### 6.2 Activity 실행 모드

| 모드 | 설명 | 권장 용도 |
|------|------|----------|
| **Threaded** | 스레드 기반 실행 | **기본 권장** |
| **Async** | 비동기 실행 | 이벤트 루프 블로킹 없을 때만 |
| **Multiprocess** | 멀티프로세스 실행 | CPU 집약적 작업 |

**공식 권장:**
> "By default, Activities should be synchronous rather than asynchronous. You should only make an Activity asynchronous if you are certain that it doesn't block the event loop."

### 6.3 주요 API

| API | 설명 |
|-----|------|
| `execute_activity()` | Activity 실행 후 결과 대기 |
| `start_activity()` | Activity 시작, Handle 반환 |
| `workflow.sleep()` | Durable한 대기 (서버 재시작에도 유지) |
| `workflow.wait_condition()` | 조건 만족까지 대기 |

---

## 7. Observability (관측 가능성)

### 7.1 제공되는 관측 기능

| 기능 | 설명 |
|------|------|
| **Metrics** | Prometheus 호환 메트릭 |
| **Tracing** | OpenTelemetry 통합 |
| **Logging** | SDK 로깅 |
| **Visibility** | Workflow 검색 및 필터링 |
| **Web UI** | 시각적 모니터링 인터페이스 |

### 7.2 주요 메트릭

| 메트릭 | 설명 |
|--------|------|
| `temporal_activity_schedule_to_start_latency` | Activity 대기 시간 |
| `temporal_workflow_task_schedule_to_start_latency` | Workflow Task 대기 시간 |
| Worker, Client 관련 다양한 메트릭 | SDK별 참조 |

### 7.3 Search Attributes

- Custom 속성을 Workflow에 추가하여 검색 가능
- 지원 타입: Text, Keyword, Int, Double, Bool, Datetime, KeywordList

---

## 8. Sticky Execution (성능 최적화)

### 8.1 개념

| 속성 | 설명 |
|------|------|
| **목적** | Workflow 상태 캐싱으로 성능 향상 |
| **동작** | 동일 Worker에서 연속 Task 처리 |
| **구현** | Worker 고유의 Sticky Queue 사용 |

**공식 설명:**
> "Workers cache the state of the Workflow they execute. To make this caching more effective, Temporal employs a performance optimization known as 'Sticky Execution'."

### 8.2 Sticky Queue Timeout

- 기본값: 5초
- Sticky Queue에서 Task를 가져가지 못하면 → 원래 Task Queue로 복귀
- 다른 Worker가 처리 가능

---

## 9. ADR-003 연계 포인트

### 9.1 ADR-003 결정 사항과 Temporal 공식 기능 매핑

| ADR-003 결정 | Temporal 공식 기능 | 검증 결과 |
|-------------|-------------------|----------|
| 비동기 워크플로우 실행 | Workflow + Activity 분리 | ✓ 공식 지원 |
| Durable Execution | Event Sourcing + Replay | ✓ 핵심 기능 |
| 장애 복구 | Event History 기반 재개 | ✓ 공식 지원 |
| SSE 스트리밍 | Activity에서 스트리밍 처리 | ✓ 구현 가능 |
| 재시도 정책 | Retry Policy (선언적) | ✓ 공식 지원 |
| 상태 조회 | Query API | ✓ 공식 지원 |
| 작업 취소 | Cancel + Signal | ✓ 공식 지원 |

### 9.2 구현 시 고려사항

| 항목 | 권장 사항 | 근거 |
|------|----------|------|
| Activity 타임아웃 | Start-To-Close 필수 설정 | Temporal 공식 권장 |
| 장시간 Activity | Heartbeat 사용 | 빠른 장애 감지 |
| Python Activity | Threaded 모드 기본 사용 | 공식 권장 |
| GPU/CPU 분리 | 별도 Task Queue 사용 | Task Routing 패턴 |

---

## 10. 출처 (Sources)

### 공식 문서
- [Temporal Workflows](https://docs.temporal.io/workflows)
- [Temporal Activities](https://docs.temporal.io/activities)
- [Temporal Workers](https://docs.temporal.io/workers)
- [Task Queues](https://docs.temporal.io/task-queue)
- [Retry Policies](https://docs.temporal.io/encyclopedia/retry-policies)
- [Detecting Activity Failures](https://docs.temporal.io/encyclopedia/detecting-activity-failures)
- [Workflow Message Passing](https://docs.temporal.io/encyclopedia/workflow-message-passing)
- [Sticky Execution](https://docs.temporal.io/sticky-execution)
- [Task Routing](https://docs.temporal.io/task-routing)
- [Python SDK Developer Guide](https://docs.temporal.io/develop/python)
- [Python SDK Failure Detection](https://docs.temporal.io/develop/python/failure-detection)
- [Python SDK Observability](https://docs.temporal.io/develop/python/observability)
- [Observability Features](https://docs.temporal.io/evaluate/development-production-features/observability)

### 공식 블로그
- [Temporal: Durable Execution Platform](https://temporal.io/product)
- [Activity Timeouts](https://temporal.io/blog/activity-timeouts)
- [Durable Execution meets AI](https://temporal.io/blog/durable-execution-meets-ai-why-temporal-is-the-perfect-foundation-for-ai)

### SDK 레퍼런스
- [Python SDK GitHub](https://github.com/temporalio/sdk-python)
- [Python SDK RetryPolicy](https://python.temporal.io/temporalio.common.RetryPolicy.html)

---

## 변경 이력

| 날짜 | 작성자 | 변경 내용 |
|------|--------|----------|
| 2024-12-27 | System Architect | 초안 작성 — Temporal 공식 기능 조사 완료 |
