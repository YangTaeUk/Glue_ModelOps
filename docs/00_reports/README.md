# 00. Reports

> 분석 리포트 및 조사 결과

## 디렉토리 구조

```
00_reports/
├── README.md
└── archive/          # GPT 초기 논의 문서 (2024-12-24 통합)
    ├── AI Model Serving Platform - 시스템 요건 정의서 (SRS).md
    ├── AI Serving Platform 전략적 포지셔닝 및 개발 경계 정의.md
    ├── AI 모델 서빙 백엔드 문서화 전략 및 모범 사례.docx.md
    ├── Claude Opus 4.5 문서 관리 방안 구축.md
    ├── nginx_oss_vs_traefik_의사결정_문서.md
    └── 통신_방식_의사결정_문서_sync_webhook_sse_web_socket.md
```

## Archive 폴더

`archive/` 폴더에는 GPT 5.2와의 초기 논의에서 생성된 원본 문서들이 보관되어 있다.

**통합 내역 (2024-12-24):**
- 전략적 포지셔닝 → `01_overview/01_strategy.md` 강화
- SRS 도메인 최적화 → `02_architecture/06_optimization.md` 분리
- nginx vs traefik → `99_decisions/001_gateway_nginx.md` ADR
- 통신 방식 → `99_decisions/002_communication_protocol.md` ADR
- Thin Wrapper → `99_decisions/003_thin_wrapper.md` ADR

**보관 목적:** 출처 추적 및 의사결정 이력 보존

---

## 리포트 유형

| 유형 | 설명 | 네이밍 |
|------|------|--------|
| 기술 조사 | 기술 스택 비교/선정 | `tech_*.md` |
| 성능 분석 | 벤치마크, 최적화 | `perf_*.md` |
| 사례 연구 | 타사 사례 분석 | `case_*.md` |
| 회고 | 프로젝트 회고 | `retro_*.md` |

## 작성 가이드

```markdown
# [리포트 제목]

> 한 줄 요약

## 배경
- 왜 이 조사/분석을 했는가

## 방법
- 어떻게 조사/분석했는가

## 결과
- 무엇을 발견했는가

## 결론
- 어떤 결정을 내렸는가

## Related
- 관련 문서 링크
```
