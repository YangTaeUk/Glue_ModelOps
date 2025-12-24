# 01. Overview

> 프로젝트 개요

## 문서 구성

| 문서 | 내용 |
|------|------|
| [01_strategy.md](./01_strategy.md) | 프로젝트 목적, 철학, 설계 원칙 |
| [02_srs.md](./02_srs.md) | 기능 요구사항, REQ-* ID |

## 핵심 개념

**Thin Wrapper Pattern**
- 추론 엔진(vLLM, Triton)을 감싸는 얇은 레이어
- 엔진 내부 수정 금지

**우리가 하는 것**
- Workflow Orchestration (Temporal)
- Universal Interface (OpenAI API Compatible)
- Gatekeeping (Rate Limit, Auth, Quota)

**우리가 안 하는 것**
- 추론 엔진 내부 최적화
- GPU 저수준 관리
- 모델 학습/파인튜닝
