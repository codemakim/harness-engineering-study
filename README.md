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

## 나중에 확장할 방향: AX 개발자 성장 커리큘럼

이 repo는 현재 `Agent literacy`와 `Harness engineering`에 초점을 둔다. AX 개발자 또는 Agentic AI Engineer 포지션까지 이어가려면, 이후에는 “에이전트가 어떻게 동작하는가”에서 “업무를 어떻게 AI 중심으로 바꾸는가”로 확장해야 한다.

나중에 추가할 장 후보:

11. `AX 개발자는 무엇을 하는가`  
    AX를 단순 LLM API 개발이 아니라 업무 분석, 자동화 설계, 현업 정착, 성과 측정까지 포함하는 역할로 정리한다. 웹 개발자가 왜 이 영역에 강점을 가질 수 있는지도 다룬다.

12. `AX 프로젝트는 어떻게 발견하고 설계하는가`  
    어떤 업무가 agent에 적합한지, 어떤 업무는 일반 CRUD/스크립트/자동화가 더 나은지 판단하는 법을 다룬다. 업무 flow를 쪼개고, 병목과 반복 작업을 찾고, human-in-the-loop 지점을 정한다.

13. `평가와 운영: 데모에서 실제 업무 도구로`  
    agent 품질을 어떻게 측정할지, 실패 로그와 재시도, 권한, 보안, 비용, latency, 사용자 피드백, ROI를 어떻게 다룰지 정리한다.

이 확장은 바로 개발 목록으로 가면 안 된다. 먼저 개념과 이유를 설명해야 한다.

```text
왜 AX는 단순 챗봇 개발이 아닌가?
왜 업무 분석이 먼저인가?
왜 평가/evaluation 없이는 실무 도입이 위험한가?
왜 human-in-the-loop가 필요한가?
왜 웹 개발자의 API/DB/권한/운영 경험이 AX에 강점인가?
```

그 다음에야 실제 예제로 넘어간다.

```text
업무 하나 선택
→ 현재 flow 기록
→ AI 적용 후보 찾기
→ agent/tool/workflow 설계
→ 작은 prototype
→ 평가 기준 설정
→ 운영/보안/권한 검토
→ 현업 사용성 피드백
```

현재 repo의 위치:

```text
1단계: Agent literacy
2단계: Harness engineering
```

추가로 가야 할 위치:

```text
3단계: Workflow automation
4단계: Evaluation & operations
5단계: AX delivery
```

## 주요 공식 문서

- [Codex skills](https://developers.openai.com/codex/skills)
- [AGENTS.md guidance](https://developers.openai.com/codex/guides/agents-md)
- [Codex hooks](https://developers.openai.com/codex/hooks)
- [Build Codex plugins](https://developers.openai.com/codex/plugins/build)
- [Codex MCP](https://developers.openai.com/codex/mcp)
