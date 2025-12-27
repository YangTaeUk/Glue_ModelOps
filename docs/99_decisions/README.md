# 99. Decisions

> 심층 분석 및 의사결정 리포트 (Decision Analysis Reports)

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
| 003 | [003_Inference_Task_Execution_Model_and_State_Management_Strategy.md](./003_Inference_Task_Execution_Model_and_State_Management_Strategy.md) | Accepted | 비동기 워크플로우 실행, Temporal 채택, SSE 스트리밍 |
| 004 | [004_Universal_Connector_Strategy_for_Model_Integration.md](./004_Universal_Connector_Strategy_for_Model_Integration.md) | Accepted | Connector Pattern, Event Normalizer, Backend Agnostic 통신 |

---

## 관련 문서 (Related)

- [01_overview/01_strategy.md](../01_overview/01_strategy.md) - 프로젝트 전략 및 포지셔닝
- [02_architecture/](../02_architecture/) - 아키텍처 문서
