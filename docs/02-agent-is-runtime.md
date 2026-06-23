[← README](../README.md)

# 2. 에이전트는 모델이 아니라 runtime이다

Chatbot은 주로 답변한다. Agent는 답변만 하지 않고, 도구를 쓰고, 결과를 보고, 다시 판단한다.

모델 자체는 shell을 실행하지 않는다. 파일을 직접 수정하지 않는다. Calendar API를 직접 호출하지 않는다. 모델은 “이 도구를 이런 입력으로 호출하면 좋겠다”는 tool call을 만든다. 실제 실행은 하네스가 한다.

## 기본 루프

```text
사용자 입력
→ context 조립
→ LLM 호출
→ 답변 또는 tool call 생성
→ tool call이면 하네스가 실제 실행
→ 실행 결과를 context에 추가
→ 다시 LLM 호출
→ 최종 답변
```

이 루프가 agentic loop다.

## 역할 분리

```text
LLM
= 판단, 언어 생성, 다음 행동 제안

Harness
= context 준비, tool 실행, 권한 검사, 결과 재주입, lifecycle 관리
```

웹 개발자 비유:

```text
LLM = business logic을 제안하는 brain
Harness = Express/Nest 서버 + middleware + worker + permission layer
```

모델만 있으면 실행할 수 없다. 하네스만 있으면 판단할 수 없다. 둘이 합쳐져야 agent가 된다.

## Codex에서는 어떻게 보이나

Codex에서 모델은 이런 선택을 한다.

```text
답변하기
파일 읽기 tool 요청
shell command tool 요청
apply_patch 요청
web search 요청
GitHub/Gmail/Slack 같은 connector tool 요청
```

Codex는 그 요청을 실제로 실행한다. 실행 결과는 다시 모델에게 들어간다.

```text
모델: "이 파일을 읽어야겠다"
Codex: 파일 읽기 실행
Codex: 파일 내용을 모델 context에 추가
모델: 파일 내용을 보고 다음 판단
```

## 왜 중요한가

Agent를 잘 만든다는 것은 “더 멋진 프롬프트”만 쓰는 일이 아니다.

좋은 agent는 다음을 잘 설계한다.

```text
어떤 context를 넣을지
어떤 tool을 줄지
tool 설명을 어떻게 쓸지
어떤 권한을 요구할지
언제 멈출지
실패했을 때 어떻게 복구할지
```

즉 agent 개발은 모델 개발이 아니라 runtime 설계에 가깝다.

