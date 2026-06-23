[← README](../README.md)

# 3. Context assembly: 답변 전 실제로 무슨 일이 일어나는가

사용자는 질문 하나를 보낸다. 하지만 모델은 질문 본문만 받지 않는다. 하네스가 여러 층의 context를 조립해서 함께 보낸다.

## Codex에서 한 턴의 개념적 흐름

```text
1. 사용자 메시지 입력

2. Codex가 context 준비
   - system/developer/user instruction
   - 이전 대화 또는 summary
   - cwd, shell, 날짜, timezone 같은 환경 정보
   - tool definitions와 input schema
   - skill 목록과 description
   - 선택된 skill의 SKILL.md
   - AGENTS.md guidance
   - hook output
   - memory summary 또는 관련 memory

3. hook이 있다면 lifecycle에 따라 실행

4. LLM 호출

5. tool call이 나오면 Codex가 실행

6. tool result가 다시 context에 추가

7. 모델이 답변 또는 다음 tool call 결정
```

정확한 내부 직렬화 형식은 제품 내부 구현이지만, 개념적으로는 이 구조가 중요하다.

## Context layer들

```text
System instructions
→ 안전, 역할, 제품 전체 규칙

Developer instructions
→ Codex로서의 작업 방식, 도구 사용 규칙, 현재 모드

User message
→ 사용자가 방금 쓴 질문

Conversation history / summary
→ 이전 대화 일부 또는 압축 요약

Environment context
→ cwd, shell, 날짜, timezone, filesystem permission

Tool definitions
→ 호출 가능한 tool 이름, 설명, input schema

Skill list
→ 사용할 수 있는 skill의 name, description, path

Loaded skill instructions
→ 선택된 SKILL.md의 본문

AGENTS.md guidance
→ 세션 시작 때 로드된 global/project instruction

Hook output
→ lifecycle hook이 stdout으로 낸 additionalContext 또는 decision

Memory
→ 관련 memory summary나 검색된 기억

Tool outputs
→ shell/file/web/connector 실행 결과
```

## 왜 중요한가

모델 답변 품질은 사용자 질문만으로 결정되지 않는다.

```text
좋은 질문 + 나쁜 context = 이상한 답변
평범한 질문 + 좋은 context = 꽤 좋은 답변
```

웹 개발에서 controller만 보고 버그를 못 잡는 경우와 비슷하다. request가 controller에 도착하기 전에 middleware, auth, validation, session, feature flag가 이미 많이 건드렸을 수 있다.

LLM도 마찬가지다. 답변을 이해하려면 “모델이 뭘 봤는가”를 봐야 한다.

