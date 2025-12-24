# 3. Contributing

> 기여 가이드

## 워크플로우

```
1. Issue 생성 또는 할당
2. Feature branch 생성
3. 구현 및 테스트
4. PR 생성
5. 리뷰 및 수정
6. Merge
```

## 브랜치 전략

| 브랜치 | 용도 |
|--------|------|
| main | 프로덕션 |
| develop | 개발 통합 |
| feature/* | 기능 개발 |
| fix/* | 버그 수정 |
| docs/* | 문서 작업 |

## 커밋 메시지

```
<type>(<scope>): <subject>

<body>

<footer>
```

| Type | 설명 |
|------|------|
| feat | 새 기능 |
| fix | 버그 수정 |
| docs | 문서 |
| refactor | 리팩토링 |
| test | 테스트 |
| chore | 기타 |

## 코드 스타일

```bash
# 포맷팅
black src/ tests/
isort src/ tests/

# 린트
ruff check src/ tests/
mypy src/
```

## PR 체크리스트

- [ ] 테스트 통과
- [ ] 린트 통과
- [ ] 문서 업데이트
- [ ] 커밋 메시지 규칙 준수
- [ ] 리뷰어 지정

## 코드 리뷰

| 항목 | 기준 |
|------|------|
| 기능 | 요구사항 충족 |
| 테스트 | 커버리지 80%+ |
| 스타일 | 린트 통과 |
| 문서 | 변경사항 반영 |

## Related

- [../99_decisions/](../99_decisions/) - ADR 템플릿
