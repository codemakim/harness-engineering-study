# Repo guide for Codex

이 repo는 개인 학습 자료를 GitHub에서 읽기 좋게 모으는 study library다.

## 먼저 볼 파일

1. `README.md`
2. 작업 대상 course의 `README.md`
3. 작업 대상 course의 writing plan이 있으면 그것

현재 주요 course:

- `courses/ax-harness-engineering/`
- `courses/react/`

## 구조

```text
courses/<topic>/docs/
→ 교재형으로 다시 쓴 학습 문서

courses/<topic>/legacy/
→ 블로그 원문, 예전 자료, import 원본

courses/<topic>/labs/
→ 실습 코드와 관찰 기록

notes/inbox/
→ 아직 course로 정리하지 않은 임시 메모

templates/
→ 새 course/chapter/feedback 양식
```

## 작업 원칙

- 원문은 바로 고치지 말고 `legacy/` 또는 `notes/inbox/`에 보관한다.
- 다시 읽기 좋게 리라이트한 문서만 `docs/`에 둔다.
- 대량 자료는 한 번에 다 읽지 말고 목록 → 묶음 분류 → 장별 처리로 진행한다.
- 새 장을 만들면 course README 목차에 링크를 추가한다.
- 링크와 markdown fence를 검증한다.

## AX course 상태

`courses/ax-harness-engineering/`는 1~14장까지 작성됐다.

다음 예정 장:

```text
15. Portfolio: 실습 결과를 공개 가능한 기록으로 정리하기
```

## React course 상태

`courses/react/`는 빈 구조만 있다.

다음 작업:

```text
1. 사용자가 블로그 학습 자료 위치를 알려준다.
2. 원문을 courses/react/legacy/blog-import/에 보관한다.
3. 파일 목록과 주제를 먼저 파악한다.
4. 주제별 묶음으로 나눈다.
5. courses/react/docs/에 교재형 문서로 리라이트한다.
```
