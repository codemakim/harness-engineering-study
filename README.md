# Harness Engineering for Web Developers

Caveman과 Ponytail로 배우는 Codex 에이전트 구조.

이 repo는 “무언가를 만든다”보다 “왜 그렇게 동작하는지 이해한다”에 초점을 둔 강의형 노트다. 대상 독자는 AI 전공자가 아니라, 서버/API/DB/middleware/tooling 감각이 있는 웹 개발자다.

## 목표

프롬프트 몇 줄을 외우는 것이 아니라, LLM 주변의 runtime을 이해한다.

```text
LLM = context를 보고 다음 답변 또는 tool call을 만드는 모델
Agent harness = context 조립, tool 실행, memory 검색, 권한 검사, hook 실행을 담당하는 런타임
```

## 강의 목차

1. [LLM은 기억하는 것이 아니라 context를 읽는다](docs/01-llm-is-context.md)  
   모델을 `model(context) -> output`으로 보는 기초. “기억처럼 보이는 것”의 정체.

2. [에이전트는 모델이 아니라 runtime이다](docs/02-agent-is-runtime.md)  
   tool call, tool runner, observation 재주입, permission layer.

3. [Context assembly: 답변 전 실제로 무슨 일이 일어나는가](docs/03-context-assembly.md)  
   한 번의 질문에 system/developer/user/history/tool/schema/memory가 어떻게 붙는지.

4. [Codex 지시문 표면: AGENTS.md, Skills, Hooks, Plugins](docs/04-codex-surfaces.md)  
   왜 지시문 넣는 방법이 여러 개인지, 각각 언제 써야 하는지.

5. [Hooks lifecycle: 대화 주기에 코드를 꽂는다는 것](docs/05-hooks-lifecycle.md)  
   SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, trust/review.

6. [Case study: Ponytail은 왜 강하게 작동하는가](docs/06-case-study-ponytail.md)  
   manifest, hook 등록, mode tracker, 상태 파일, additionalContext.

7. [Case study: Caveman은 왜 Hooks에 안 떴나](docs/07-case-study-caveman.md)  
   코드 존재와 runtime discovery의 차이.

8. [Memory design: 모델이 기억하는 게 아니라 하네스가 가져온다](docs/08-memory-design.md)  
   recent history, summary, retrieval, stale memory, source discipline.

9. [Tool design: 모델에게 함수를 맡기는 법](docs/09-tool-design.md)  
   tool schema는 모델을 위한 API 문서다.

10. [나만의 agent harness 설계하기](docs/10-building-harness-lab.md)  
    개발 목록이 아니라 질문/가설/관찰 중심의 실험 설계.

## Labs

실험 코드는 나중에 [labs/](labs/README.md)에 둔다. 각 lab은 다음 형식으로 기록한다.

```text
질문
가설
최소 구현
관찰
배운 점
다음 질문
```

## 주요 공식 문서

- [Codex skills](https://developers.openai.com/codex/skills)
- [AGENTS.md guidance](https://developers.openai.com/codex/guides/agents-md)
- [Codex hooks](https://developers.openai.com/codex/hooks)
- [Build Codex plugins](https://developers.openai.com/codex/plugins/build)
- [Codex MCP](https://developers.openai.com/codex/mcp)
