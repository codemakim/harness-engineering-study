# Harness Engineering Study

Codex의 `AGENTS.md`, Skills, Hooks, Plugins가 어떻게 LLM 행동을 바꾸는지 공부하는 개인 노트.

출발점은 `caveman`, `ponytail` 플러그인 분석이다. 목표는 나중에 직접 설치 가능한 Codex 플러그인/에이전트 하네스를 만드는 것.

## 읽는 순서

1. [LLM과 에이전트의 기본 동작 원리](docs/01-llm-agent-foundations.md)  
   모델 자체와 에이전트 하네스를 분리해서 보는 기초. Telegram + Node LLM 서버 경험과 Codex를 비교한다.

2. [Codex의 context surface: AGENTS.md, Skills, 기본 정신모델](docs/02-codex-context-surfaces.md)  
   어떤 지시문이 어디서 발견되고, 언제 context에 들어가는지 정리한다.

3. [Hooks와 lifecycle](docs/03-hooks-and-lifecycle.md)  
   세션 시작, 유저 프롬프트 제출, 툴 사용 전후에 스크립트가 어떻게 실행되는지 정리한다.

4. [Caveman / Ponytail case study](docs/04-caveman-ponytail-case-study.md)  
   왜 Ponytail은 Settings > Hooks에 뜨고 Caveman은 안 뜨는지, 로컬 설치 파일 기준으로 분석한다.

5. [직접 에이전트/플러그인 만들기](docs/05-building-your-own-agent.md)  
   최소 plugin 구조, mini mode plugin 실험, 하네스 엔지니어링 체크리스트.

6. [정정 사항, 레퍼런스, 다음 질문](docs/06-corrections-references-next.md)  
   헷갈렸던 표현 정정, 공식 문서 링크, 앞으로 파볼 질문 목록.

## 핵심 요약

```text
LLM
= context를 보고 다음 답변 또는 tool call을 만드는 모델

Agent harness
= context 조립, tool 실행, memory 검색, 권한 검사, hook 실행을 담당하는 런타임

AGENTS.md
= 세션 시작 때 로드되는 지속 지시문

Skill
= 필요할 때 읽히는 작업 매뉴얼

Hook
= lifecycle 이벤트마다 실행되는 자동 장치

Plugin
= skills/hooks/MCP/assets를 설치 가능한 단위로 묶은 패키지
```

## 프로젝트 상태

현재는 학습 노트 repo다. 아직 배포용 plugin 코드는 없다.

나중에 커지면 이런 구조로 확장한다.

```text
docs/          학습 문서
experiments/   작은 hook/skill 실험
plugins/       직접 만든 Codex plugin
examples/      예제 입력/출력
```

## 주요 공식 문서

- [Codex skills](https://developers.openai.com/codex/skills)
- [AGENTS.md guidance](https://developers.openai.com/codex/guides/agents-md)
- [Codex hooks](https://developers.openai.com/codex/hooks)
- [Build Codex plugins](https://developers.openai.com/codex/plugins/build)
- [Codex MCP](https://developers.openai.com/codex/mcp)
