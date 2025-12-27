# 99. Decisions

> 심층 분석 및 의사결정 리포트 (Decision Analysis Reports)

---

## 현재 상태 (Current State)

| 지표 | 값 |
|------|-----|
| **총 ADR 수** | 10개 (001-010) |
| **상태** | 모두 Accepted |
| **마지막 업데이트** | 2024-12-28 |

**의사결정 범위:**
```
Edge Layer (보안/인증) → Pipeline Manager → Temporal Workflow → Connector → Worker → Inference Server
```

---

## 핵심 원칙 요약 (Key Principles)

| 원칙 | 설명 | 관련 ADR |
|------|------|---------|
| **Edge 인증** | 인증 책임은 Edge Layer(Nginx)에 위임 | 001, 008 |
| **분산 추적** | W3C Trace Context로 요청 추적, PLG Stack | 009 |
| **Native Async** | FastAPI + ASGI, 비동기 I/O 필수 | 010 |
| **Backend Agnostic** | 특정 추론 엔진 강제 없음 | 004, 007 |
| **Model Agnostic** | LLM/CV/Audio 등 모델 유형 비종속 | 005, 006 |
| **선형 파이프라인** | 조건 분기 금지, Content-based Routing만 허용 | 002 |
| **Thin Wrapper** | 추론 엔진 내부 수정 없음 | 전체 |
| **Why/What 원칙** | ADR은 구현 코드 없이 원칙만 기술 | 전체 |

---

## 철학 (Philosophy)

**의사결정 문서는 "결과 통보서"가 아니라 "논리적 증명서"이다.**

> "6개월 뒤의 나, 혹은 새로 합류한 팀원이 이 문서를 읽고 '왜 이런 결정을 내렸는지' 완전히 이해할 수 있어야 한다."

ADR(Architecture Decision Record)의 진정한 가치는 **"치열한 고민의 흔적"**을 남기는 것이다:
- 어떤 문제가 있었는가 (Problem)
- 무엇을 조사했는가 (Facts)
- 어떻게 생각했는가 (Reasoning)
- 그래서 무엇을 결정했는가 (Decision)

단순한 Pros/Cons 비교표는 "왜 그 장단점이 중요한가?"를 설명하지 못한다.
**사고의 흐름(Flow of Thought)**을 따라가는 문서만이 진정한 자산이 된다.

### Why/What 원칙

> **ADR은 "무엇을(What)" 결정하고, "어떻게(How)"는 설계 문서로 위임한다.**

| ADR에서 결정 | 설계 문서로 위임 |
|-------------|-----------------|
| Temporal 채택 | Retry Policy 구체 설정 |
| Connector Pattern 사용 | 메서드 시그니처 정의 |
| Dual Path 전략 | 저장소 선택 (MinIO, S3 등) |
| Task Queue 분리 | Worker 슬롯 수치 |

---

## 작성 원칙 (Writing Principles)

### 1. 팩트 기반 (Fact-Based)
- "뇌피셜"이 아닌 데이터, 조사 결과, 실제 경험에 근거
- 출처 명시: 벤치마킹 사례, 기술 문서, 로그 데이터

### 2. 논리의 흐름 (Logical Flow)
- **"왜 A를 기각했는가"가 "왜 B를 선택했는가"만큼 중요**
- 기각된 대안도 충분한 설명과 함께 기록
- 각 대안의 "결정적 결함" 명시

### 3. 쟁점 기록 (Debate Documentation)
- 팀 내 의견 충돌도 자산
- "개발팀은 A를 선호했으나, 기획팀은 B를 선호했다" 기록
- 합의 과정과 최종 판단 근거 명시

### 4. 비즈니스 임팩트 (Business Impact)
- 기술적 장단점뿐 아니라 비즈니스 영향도 고려
- "해결하지 않으면 어떤 손해가 발생하는가?"
- 가능하면 정량화 (예: 비용, 시간, 이탈률)

### 5. 실행 가능성 (Actionable)
- 결정 후 "누가, 언제까지, 무엇을" 명확히
- 재검토 조건(Revisit Triggers)을 미리 정의

---

## 4단계 구조 (Four-Phase Structure)

모든 ADR은 다음 4단계 구조를 따른다:

```
┌─────────────────────────────────────────────────────────────┐
│  Phase 1: THE PROBLEM (문제 정의)                            │
│  ├─ 배경: 왜 이 논의가 시작되었는가?                         │
│  ├─ 핵심 문제: 겉으로 드러난 현상 vs 본질적 원인             │
│  └─ 비즈니스 임팩트: 해결 안 하면 어떤 손해?                 │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Phase 2: FACTS & CONSTRAINTS (리서치 및 제약)               │
│  ├─ 사전 조사: 벤치마킹, 기술 문서, 데이터 로그              │
│  └─ 제약 사항: 예산, 일정, 레거시, 팀 역량                   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Phase 3: ANALYSIS & REASONING (사고 과정)                   │
│  ├─ 주요 쟁점: 팀 내 의견이 갈렸던 부분                      │
│  └─ 대안 분석: 각 대안의 매력/기각 사유 (논리의 흐름)        │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Phase 4: DECISION & ACTION (최종 결정)                      │
│  ├─ 최종 결정: 한 문장으로 명확히                            │
│  ├─ 트레이드오프: 감수해야 할 단점과 보완 계획               │
│  ├─ Action Items: 담당자, 기한, 작업 항목                    │
│  └─ 재검토 트리거: 언제 이 결정을 다시 논의할 것인가         │
└─────────────────────────────────────────────────────────────┘
```

---

## 문서 상태 (Status)

| 상태 | 설명 |
|------|------|
| `proposed` | 검토 중 (의견 수렴 단계) |
| `accepted` | 채택됨 (실행 중) |
| `revisiting` | 재검토 진행 중 (상황 변화로 재논의) |
| `deprecated` | 더 이상 유효하지 않음 (환경 변화) |
| `superseded` | 다른 ADR로 대체됨 (ADR-XXX 참조) |

---

## 작성 시점 (When to Write)

- 기술 스택 선정 (프레임워크, 라이브러리, 인프라)
- 아키텍처 패턴 결정 (마이크로서비스 vs 모놀리식 등)
- 주요 트레이드오프 결정 (성능 vs 비용, 속도 vs 안정성)
- 기존 결정 변경 (왜 이전 결정을 뒤집었는지)
- 팀 내 쟁점 해결 (의견 충돌 후 합의)

---

## 네이밍 규칙 (Naming Convention)

```
NNN_[제목].md

예시:
001_gateway_nginx.md
002_communication_protocol.md
003_thin_wrapper.md
```

> **Note**: 상태는 파일명이 아닌 문서 내부 메타데이터로 관리한다.
> 상태가 변경될 때마다 파일명을 바꾸는 것은 링크 깨짐 등의 문제를 유발한다.

---

## 문서 목록 (Document Index)

| # | 문서 | 상태 | 핵심 결정 |
|---|------|------|----------|
| 000 | [000_template.md](./000_template.md) | - | ADR 작성 템플릿 |
| 001 | [001_How_to_Securely_Expose_AI_API_Endpoint_[Nginx OSS].md](./001_How_to_Securely_Expose_AI_API_Endpoint_[Nginx%20OSS].md) | Accepted | AI API 외부 노출 방법, Nginx OSS 선택 |
| 002 | [002_Defining_Scope_and_Architecture_Style_of_AI_Backend_Application.md](./002_Defining_Scope_and_Architecture_Style_of_AI_Backend_Application.md) | Accepted | AI 백엔드 정체성: Atomic AI Pipeline Manager, 선형 파이프라인 제한 |
| 003 | [003_Inference_Task_Execution_Model_and_State_Management_Strategy.md](./003_Inference_Task_Execution_Model_and_State_Management_Strategy.md) | Accepted | 비동기 워크플로우 실행, Temporal 채택, 다중 통신 방식 지원 |
| 004 | [004_Universal_Connector_Strategy_for_Model_Integration.md](./004_Universal_Connector_Strategy_for_Model_Integration.md) | Accepted | Connector Pattern, Event Normalizer, Backend Agnostic 통신 |
| 005 | [005_Realtime_Streaming_Channel_Strategy.md](./005_Realtime_Streaming_Channel_Strategy.md) | Accepted | Dual Path 전략 (Temporal/Direct), 환경 기반 저장소 선택, 클라이언트 통신 채널 |
| 006 | [006_Worker_Resource_Separation_and_Parallel_Processing_Strategy.md](./006_Worker_Resource_Separation_and_Parallel_Processing_Strategy.md) | Accepted | Task Queue 기반 리소스 분리, CPU/Accelerator Worker, 리소스 이질성 대응 |
| 007 | [007_Inference_Server_Deployment_and_Worker_Communication_Boundary.md](./007_Inference_Server_Deployment_and_Worker_Communication_Boundary.md) | Accepted | 추론 서버 배포 책임(인프라), Worker-Server 분리 배포, 백엔드 선택 비강제 |
| 008 | [008_System_Client_Authentication_and_Access_Control_Strategy.md](./008_System_Client_Authentication_and_Access_Control_Strategy.md) | Accepted | Edge 인증 위임, Static API Key, X-Client-ID 신원 전파 |
| 009 | [009_Unified_Observability_and_Distributed_Tracing_Strategy.md](./009_Unified_Observability_and_Distributed_Tracing_Strategy.md) | Accepted | W3C Trace Context, PLG Stack, 구조화 로깅 |
| 010 | [010_Backend_Application_Framework_Selection.md](./010_Backend_Application_Framework_Selection.md) | Accepted | FastAPI 채택, Native Async, Pydantic 검증 |

---

## ADR 의존 관계 (Dependency Graph)

```
ADR-001 (Edge Layer: Nginx)
    │
    ├────────────────────────────────────────────────┐
    │                                                ▼
    ▼                                            ADR-008
ADR-002 (Pipeline Manager: 선형 파이프라인)      (Edge 인증: API Key)
    │
    ▼
ADR-003 (Execution Model: Temporal)
    │
    ├─────────────────┬─────────────────┐
    ▼                 ▼                 ▼
ADR-004           ADR-005           ADR-006
(Connector)       (Dual Path)       (Worker 분리)
    │                 │                 │
    └─────────────────┴────────┬────────┘
                               ▼
                           ADR-007
                    (배포 경계: 인프라 책임)

                               │
                               ▼
                           ADR-010
                    (프레임워크: FastAPI)
                    ← 002, 003, 005 요구사항 충족

────────────────────────────────────────────────────────────
               ▲ 횡단 관심사 (Cross-cutting Concern) ▲
────────────────────────────────────────────────────────────

ADR-009 (관측성: Trace ID 전파, PLG Stack)
    │
    └── 전체 시스템 (001, 003, 004, 008) 관통
```

---

## 설계 문서 위임 사항 (Pending Design Docs)

ADR에서 결정된 원칙을 구현할 때 작성이 필요한 설계 문서:

| ADR | 위임 사항 | 대상 문서 | 상태 |
|-----|----------|----------|------|
| 002 | Pipeline Template 스키마 | 02_architecture/ | 미작성 |
| 003 | Retry Policy 구체 설정 | 04_operations/01_temporal.md | 미작성 |
| 004 | Connector 인터페이스 정의 | 02_architecture/ | 미작성 |
| 005 | 저장소 선택 (MinIO, S3 등) | 04_operations/02_deployment.md | 미작성 |
| 006 | Worker 슬롯 수치 | 02_architecture/04_worker.md | 미작성 |
| 007 | 백엔드 선택 가이드 | 04_operations/02_deployment.md | 미작성 |
| 008 | API Key 검증 Nginx 설정 | 02_architecture/03_gateway.md | 미작성 |
| 008 | API Key 발급/로테이션 절차 | 04_operations/02_deployment.md | 미작성 |
| 009 | PLG Stack 구성, 대시보드 설정 | 04_operations/03_monitoring.md | 미작성 |
| 009 | 로깅 라이브러리 설정 가이드 | 03_development/01_setup.md | 미작성 |
| 010 | FastAPI 프로젝트 구조, 의존성 | 03_development/01_setup.md | 미작성 |
| 010 | 비동기 코딩 가이드라인 | 03_development/03_contributing.md | 미작성 |

---

## 관련 문서 (Related)

- [01_overview/01_strategy.md](../01_overview/01_strategy.md) — 프로젝트 전략 및 포지셔닝
- [02_architecture/](../02_architecture/) — 아키텍처 문서 (구현 시 작성)
