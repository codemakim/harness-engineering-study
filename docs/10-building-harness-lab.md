[← README](../README.md)

# 10. 나만의 agent harness 설계하기

바로 “멋진 에이전트 앱”을 만들면 공부가 덜 된다. 지금 목표는 제품이 아니라 관찰 장치다.

추천 이름:

```text
Harness Lab
```

목표:

```text
Codex agent harness를 배우기 위한 실험형 플러그인/노트.
context, hooks, skills, memory, plugin packaging을 작은 실험으로 관찰한다.
```

## 개발 목록이 아니라 실험 목록

각 실험은 이렇게 기록한다.

```text
질문
가설
최소 구현
관찰
배운 점
다음 질문
```

예:

```text
질문:
UserPromptSubmit hook은 매 프롬프트마다 실행되는가?

가설:
사용자가 메시지를 보낼 때마다 hook이 stdin으로 prompt JSON을 받는다.

최소 구현:
timestamp와 prompt 일부를 logs/user-prompt-submit.jsonl에 기록한다.

관찰:
trust 전에는 실행되지 않고, trust 후 새 prompt마다 로그가 추가된다.

배운 점:
Hook은 instruction file이 아니라 lifecycle command다.
```

## 추천 실험 흐름

```text
1. Context Inspector
   지금 어떤 context surface가 영향을 줄 수 있는지 관찰

2. Hook Playground
   SessionStart/UserPromptSubmit hook의 실행 타이밍 관찰

3. Mini Mode Plugin
   작은 Ponytail/Caveman류 mode plugin 구현

4. Memory Harness
   파일 기반 memory retrieval 실험

5. Skill Authoring Lab
   skill description과 implicit invocation 실험
```

## 왜 이 순서인가

```text
Context Inspector
→ 모델이 뭘 보고 있는지 눈이 생김

Hook Playground
→ lifecycle 감각이 생김

Mini Mode Plugin
→ Caveman/Ponytail 원리를 손으로 복제

Memory Harness
→ 사용자가 만든 Telegram LLM 서버 경험과 연결

Skill Authoring Lab
→ 남이 설치할 수 있는 workflow로 정리
```

## 좋은 GitHub repo의 방향

이 repo는 처음부터 “완성품”을 주장하지 않는다. 대신 학습 과정이 보이게 만든다.

```text
docs/
→ 개념 강의

labs/
→ 작은 실험

plugins/
→ 실험에서 살아남은 것만 plugin으로 승격
```

개발자들이 좋아하는 건 완성품만이 아니다. 내부를 보이게 만드는 도구도 좋아한다. Harness Lab은 그런 작은 현미경 같은 프로젝트가 될 수 있다.

## 다음 큰 확장: AX 개발자로 가는 다리

이 장의 실험들은 agent harness 자체를 이해하기 위한 것이다. 하지만 AX 개발자 역량은 여기서 한 단계 더 나아간다.

```text
Harness engineering
→ agent가 어떻게 동작하는지 이해

AX delivery
→ 실제 업무를 AI 중심으로 바꾸고 정착시키는 능력
```

나중에 추가할 내용은 단순히 “무엇을 개발한다”가 아니라 다음 질문에서 출발해야 한다.

```text
AX 개발자는 실제로 어떤 문제를 푸는가?
업무 flow에서 AI가 들어갈 자리는 어떻게 찾는가?
agent가 맞는 문제와 단순 자동화가 맞는 문제는 어떻게 구분하는가?
평가 기준 없이 agent를 도입하면 왜 위험한가?
현업 사용자가 믿고 쓰게 만들려면 어떤 운영 장치가 필요한가?
```

후속 장 후보:

```text
11. AX 개발자는 무엇을 하는가
12. AX 프로젝트는 어떻게 발견하고 설계하는가
13. 평가와 운영: 데모에서 실제 업무 도구로
```

이 세 장은 현재 docs의 자연스러운 다음 단계다. 현재 문서가 “agent runtime을 이해하는 법”이라면, 다음 문서는 “그 runtime을 업무 혁신에 적용하는 법”이 되어야 한다.
