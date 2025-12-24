# ADR-003: Thin Wrapper 패턴 및 개발 경계 정의

> **Status**: Accepted
> **Date**: 2024-12-24
> **Deciders**: AI Serving Foundation Team

## Context

AI 모델 서빙 플랫폼 구축 시, 개발 범위와 책임 경계를 명확히 정의해야 한다.

**운영 환경 전제:**
- 중소규모 팀 (인프라 복잡도 최소화 우선)
- 서버 내 Docker(Compose) 기반
- 추론 엔진: vLLM, Triton, TGI 등 오픈소스 활용
- 오케스트레이션: Temporal 기반

**문제 정의:**
- 특화 라이브러리(vLLM, Triton)는 훌륭한 '엔진'이지만 완전한 서비스가 아님
- 대규모 플랫폼(KServe, SageMaker)은 소규모 팀에겐 오버엔지니어링
- 두 극단 사이의 "Missing Link"를 채우는 경량화된 미들웨어 필요

## Decision Drivers

- **리소스 제약**: 소규모 팀의 한정된 인력과 시간
- **운영 단순화**: 인프라 복잡도를 낮추고 코드 레벨 제어권 확보
- **유연성**: 모델/엔진 교체 시 코드 수정 최소화
- **안정성**: 엔진 장애 시 복구 가능한 구조

## Decision

### Thin Wrapper 패턴 채택

우리 시스템은 거대한 플랫폼이 아니라, 강력한 엔진들에 **표준화된 손잡이**를 달아주는 **'얇은 래퍼(Thin Wrapper)'**이다.

> **팀장 코멘트**: "우리는 '엔진'을 만드는 게 아니라 '자동차'를 만드는 거다. 엔진(vLLM)은 가져다 쓰고, 우리는 운전대와 계기판을 붙이는 역할."

### 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **Engine Agnostic** | 백엔드가 vLLM이든 Triton이든, `InferenceAdapter.predict()` 한 줄로 통일 |
| **Fail Fast, Retry Smart** | 엔진 장애 → 즉시 에러 → Temporal이 다른 워커/재시도 처리 |
| **Configuration over Coding** | 모델 추가 = 코드 수정이 아닌 YAML 설정 추가 |

### 개발 경계 (Core Competency)

**우리가 반드시 만들어야 하는 영역:**

| 영역 | 구현 | 이유 |
|------|------|------|
| **Workflow Orchestration** | Temporal | 비즈니스 로직 흐름, 재시도/보상 트랜잭션 |
| **Universal Interface** | OpenAI API Compatible | 프론트엔드 팀 방어막, 모델 교체 투명화 |
| **Gatekeeping** | Rate Limit, Auth, Quota | 보안, 빌링, PII 마스킹 |

> **팀장 코멘트**: "이 세 가지는 라이브러리가 대신 못 해준다. 우리의 비즈니스 로직이기 때문."

**우리가 절대 건드리지 말아야 할 영역:**

| 영역 | 담당 | 이유 |
|------|------|------|
| **추론 엔진 내부** | vLLM, Triton, TGI | PagedAttention 등 최적화는 그들의 몫 |
| **GPU 저수준 관리** | Docker, K8s NVIDIA Plugin | 드라이버/CUDA 버전 직접 제어 금지 |
| **Complex Network Mesh** | 회피 | Istio 같은 서비스 메시 도입 지양 |

> **팀장 코멘트**: "Pytorch로 직접 model.forward() 짜지 마라. 성능 최적화는 vLLM이 압도적으로 잘한다."

### 의사결정 매트릭스 (팀원 가이드)

| 상황 | 행동 지침 |
|------|----------|
| 새로운 모델 추가 | Docker 이미지(vLLM 등) 존재 확인 → 직접 Python 코드는 최후 수단 |
| 추론 속도 느림 | Python 코드 튜닝 X → TensorRT 변환 또는 엔진 설정(Batch Size) 튜닝 |
| 서버 다운 빈발 | 서버 코드 수정 전 → Temporal Retry 정책 + K8s Liveness Probe 확인 |
| 복잡한 파이프라인 | KServe YAML 작성 X → Temporal Workflow 코드로 해결 |

## Consequences

### Positive

- 추론 성능 최적화를 엔진에 위임하여 개발 리소스 집중
- Temporal UI로 워크플로우 디버깅 용이 (에러 위치 즉시 확인)
- 모델/엔진 교체 시 Adapter 구현만 추가
- 인프라 복잡도 최소화로 운영 부담 경감

### Negative

- 엔진 자체 버그/한계 발생 시 우회만 가능 (직접 수정 불가)
- 특수 최적화 요구 시 엔진 지원 여부에 의존
- Adapter 추상화 레이어로 인한 미미한 오버헤드

## Variable vs Invariant 분리

| 구분 | 내용 | 변경 주기 |
|------|------|----------|
| **Variable (모델)** | 도메인별 모델, 프롬프트, 파라미터 | 자주 변경 |
| **Invariant (인프라)** | Gateway, Temporal, 모니터링, 공통 인터페이스 | 거의 불변 |

> **팀장 코멘트**: "모델은 바뀌어도 인프라는 재사용된다. 인프라에 비즈니스 로직 넣지 말고, 모델에 인프라 의존성 넣지 마라."

## 비교 분석

### vs 특화 라이브러리 (vLLM/Triton) 단독

| 항목 | 특화 라이브러리 | 우리 시스템 |
|------|----------------|-------------|
| 요청 대기열 | 단순 FIFO (메모리) | Priority Queue + 영속적 상태 |
| 보안/인증 | Basic Auth 수준 | RBAC, PII 마스킹, Rate Limit |
| 장애 복구 | 요청 유실 | Temporal로 상태 보존 |
| 확장성 | 수동 복제 | 오토스케일링 정책 |

### vs 대규모 플랫폼 (KServe/SageMaker)

| 항목 | 대규모 플랫폼 | 우리 시스템 |
|------|-------------|-------------|
| 구축 난이도 | 최상 (Istio, Knative 등) | 중 (FastAPI + Temporal) |
| 비용 | 높음 | 최적화 가능 |
| 유연성 | 프레임워크 규칙 준수 | 코드 레벨 자유 수정 |
| 디버깅 | YAML 지옥, 로그 분산 | Python 코드 + Temporal UI |

> **팀장 코멘트**: "KServe는 '인프라'로 문제를 풀려 하고, 우리는 '코드(Temporal)'로 문제를 푼다. 인원이 적을수록 인프라 복잡도를 낮추고 코드 제어권을 가져가는 게 유리하다."

## 후속 작업 (Next Steps)

1. InferenceAdapter 인터페이스 정의
2. vLLM/Triton용 Adapter 구현 가이드
3. 모델 추가 YAML 설정 템플릿
4. Temporal Workflow 패턴 예시 코드

## Related

- [ADR-001: Gateway 선택](./001_gateway_nginx.md) - Nginx OSS 결정
- [ADR-002: 통신 프로토콜 표준화](./002_communication_protocol.md) - SSE 기본 채널
- [01_overview/01_strategy.md](../01_overview/01_strategy.md) - 전략적 포지셔닝
