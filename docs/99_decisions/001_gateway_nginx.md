# ADR-001: Gateway 선택 (Nginx OSS vs Traefik)

> **Status**: Accepted
> **Date**: 2024-12-24
> **Deciders**: AI Serving Foundation Team

## Context

AI 모델 서빙 플랫폼의 Edge/Gateway 계층에서 사용할 Reverse Proxy를 선택해야 한다.

**운영 환경 전제:**
- 서버 내 Docker(Compose) 기반
- 중소규모 팀 (인프라 복잡도 최소화 우선)
- LLM/음성 등 스트리밍(SSE/WebSocket) 트래픽이 핵심

**Gateway 책임 범위:**
- TLS 종료 (SSL Termination)
- 라우팅 (Host/Path, 헤더 기반)
- 공통 정책 (Rate Limit, CORS)
- 스트리밍 경로의 안정적 전달

## Decision Drivers

- **스트리밍 튜닝**: LLM 토큰 스트리밍(SSE/WS)에서 버퍼링/타임아웃 제어 필요
- **운영 단순화**: docker.sock 접근 없이 보안 면에서 단순한 구조 선호
- **레퍼런스**: 장애 대응 시 참고자료가 풍부해야 함
- **팀 역량**: 학습 비용이 낮고 인력 수급이 용이해야 함

## Considered Options

### Option 1: Nginx OSS

**설명**: nginx.conf 기반으로 운영되는 오픈소스 Nginx 웹서버/리버스 프록시

| Pros | Cons |
|------|------|
| 스트리밍 경로 버퍼링/타임아웃 튜닝 용이 | 설정 변경 시 reload 필요 |
| docker.sock 접근 불필요 (보안 단순) | 동적 디스커버리 미지원 |
| 표준적, 레퍼런스 풍부 | 라우팅 변경 빈번 시 관리 부담 |
| 학습 비용 낮음 | - |

> **팀장 코멘트**: "vLLM SSE 스트리밍에서 proxy_buffering off, proxy_read_timeout 3600s 같은 설정을 정밀하게 제어하려면 Nginx가 압도적으로 편하다."

### Option 2: Traefik

**설명**: Docker/Kubernetes Provider를 통한 동적 라우팅 지원, Router/Service/Middleware 모델

| Pros | Cons |
|------|------|
| 라벨 기반 동적 라우팅 | docker.sock 접근 필요 (보안 설계 필요) |
| Let's Encrypt 자동화 가능 | 스트리밍 세부 튜닝 시 Nginx보다 복잡 |
| GitOps 친화적 | Router/Service/Middleware 모델 학습 필요 |
| K8s 전환 시 유리 | - |

> **팀장 코멘트**: "Traefik은 라우팅 자동화에 강하지만, 우리 팀 규모로 K8s 네트워크까지 관리하다간 비즈니스 로직 못 짠다."

## Decision

**선택**: Nginx OSS 단독

**이유**:
1. LLM/음성 스트리밍이 핵심 트래픽 → Nginx의 보수적 튜닝이 유리
2. 운영 복잡도 최소화 우선 → docker.sock 접근 회피
3. 소규모 팀의 레퍼런스 기반 장애 대응 → 표준적인 Nginx가 적합
4. 현재 Docker 환경에서 동적 라우팅 필요성 낮음

## Consequences

### Positive

- 스트리밍 경로에서 버퍼링, 타임아웃, 헤더 제어를 명시적으로 관리 가능
- 보안 구조 단순화 (docker.sock 접근 불필요)
- 장애 발생 시 풍부한 레퍼런스로 빠른 대응 가능
- 팀원 온보딩 시 학습 곡선 완만

### Negative

- 라우팅/정책 변경 시 설정 파일 수정 및 reload 필요
- 서비스 수가 크게 늘어날 경우 nginx.conf 관리 복잡도 증가
- 동적 라우팅 자동화 미지원

## Traefik 재평가 트리거

다음 조건 발생 시 Traefik (또는 ingress-nginx) 전환을 재검토:
- Kubernetes 환경으로 전환
- 마이크로서비스 수 증가로 라우팅 변경이 매우 빈번해짐
- GitOps/Provider 기반 동적 라우팅 자동화가 필수가 됨

> **주의**: "성장 = Traefik"이 아니라 "운영 형태 변화 = Traefik 후보군 강화"가 정확한 판단 기준

## 전환 비용 최소화 원칙

현재부터 다음을 표준으로 고정하여 향후 전환 비용을 낮춘다:

1. **Gateway 책임 분리**: TLS 종료/라우팅/공통정책만 Gateway에서 수행
2. **Backend 종속 제거**: 프록시 종속 로직을 백엔드에 넣지 않음
3. **API 계약 고정**: OpenAPI/에러코드/타임아웃 정책을 문서화하고 버전 관리
4. **관측 표준화**: request_id, latency, upstream_time 등 공통 로그 필드 정의

## Related

- [02_architecture/03_gateway.md](../02_architecture/03_gateway.md) - Nginx Gateway 설계
- [02_architecture/02_network.md](../02_architecture/02_network.md) - 통신 프로토콜
- ADR-002: 통신 프로토콜 표준화
