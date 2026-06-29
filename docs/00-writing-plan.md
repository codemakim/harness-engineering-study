[← README](../README.md)

# 집필 계획 / 리라이트 체크리스트

이 문서는 `harness-engineering-study` repo를 “대화 정리 노트”에서 “웹 개발자를 위한 AX / Agent Harness 입문 개론서”로 바꾸기 위한 작업 기준이다.

이 계획이 끝날 때까지 새 문서를 작성하거나 기존 장을 리라이트할 때는 이 문서를 먼저 확인한다.

## 목표 독자

AI 전공자는 아니지만, 웹 서버/API/DB/middleware/tooling 감각이 있는 웹 개발자.

독자는 LLM과 에이전트를 막연히 써본 적은 있지만, 다음 질문에는 아직 자신 있게 답하지 못한다고 가정한다.

```text
LLM은 무엇을 보고 답하는가?
에이전트는 모델과 무엇이 다른가?
기억은 어디에 있는가?
Tool call은 누가 실행하는가?
AGENTS.md, Skill, Hook, Plugin은 왜 나뉘었는가?
Caveman/Ponytail은 왜 단순 프롬프트보다 더 강하게 작동하는가?
AX 개발자는 실제 업무에서 무엇을 바꾸는가?
```

## 목표

프롬프트 몇 줄을 외우는 것이 아니라, LLM 주변의 runtime을 이해하게 만든다.

핵심 문장:

```text
LLM = context를 보고 다음 답변 또는 tool call을 만드는 모델
Agent harness = context 조립, tool 실행, memory 검색, 권한 검사, hook 실행을 담당하는 런타임
```

## 집필 톤

사용법 매뉴얼보다 개론서에 가깝게 쓴다.

좋은 문장 흐름:

```text
처음엔 X처럼 보인다.
하지만 실제로는 Y다.
왜냐하면 Z 때문이다.
그래서 A 상황에서는 B를 쓰고, C 상황에서는 D를 쓴다.
```

예:

```text
처음엔 AGENTS.md와 Hook이 둘 다 프롬프트를 넣는 기능처럼 보인다.
하지만 실제로는 로드 시점과 실행성이 다르다.
AGENTS.md는 세션 시작 때 읽히는 durable instruction이고,
Hook은 lifecycle event마다 실행되는 command다.
그래서 repo 규칙은 AGENTS.md가 맞고,
매 프롬프트마다 상태를 보고 reminder를 넣는 모드는 Hook이 맞다.
```

## 각 장의 기본 구조

모든 장에 억지로 다 넣을 필요는 없지만, 특히 앞부분 핵심 장은 이 구조를 따른다.

```text
1. 이 장에서 배울 것
2. 먼저 직관으로 이해하기
3. 정확한 개념
4. 웹 개발자 비유
5. 작은 예시
6. Codex에서는 어떻게 보이는가
7. 흔한 오해
8. 요약
9. 생각해볼 질문
```

## 1차 리라이트 목차

아래 목차는 “개발 목록”이 아니라 “개념이 쌓이는 순서”다.

- [x] 01. 모델은 왜 기억하는 것처럼 보일까?
  - 기존 `LLM은 기억하는 것이 아니라 context를 읽는다`를 개론서 스타일로 리라이트한다.
  - stateless API, cookie/session, DB lookup 비유를 넣는다.
  - 사용자의 Telegram + Node LLM 서버 경험을 예시로 사용한다.
  - “기억처럼 보이는 것 = context에 들어온 정보”를 반복해서 각인한다.

- [x] 02. 에이전트는 모델이 아니라 runtime이다
  - tool call, tool runner, observation 재주입을 설명한다.
  - “모델은 판단하고, 하네스는 실행한다”를 중심 문장으로 둔다.
  - Express/Nest 서버, middleware, worker, permission layer 비유를 사용한다.

- [x] 03. Context assembly: 답변 전 실제로 무슨 일이 일어나는가
  - 사용자가 보는 질문 본문 외에 어떤 layer가 모델 입력에 들어갈 수 있는지 설명한다.
  - system/developer/user/history/environment/tool schema/skill/AGENTS/hook/memory/tool output을 정리한다.
  - 공식 문서로 확인 가능한 부분과 내부 구현이라 개념적으로 설명하는 부분을 구분한다.

- [x] 04. Codex 지시문 표면: AGENTS.md, Skills, Hooks, Plugins
  - 왜 지시문 넣는 방법이 여러 개인지 설명한다.
  - AGENTS.md는 durable project guidance, Skill은 on-demand workflow, Hook은 lifecycle command, Plugin은 distribution unit으로 정리한다.
  - 각 표면의 “언제 써야 하는가”를 이유 중심으로 설명한다.

- [x] 05. Hooks lifecycle: 대화 주기에 코드를 꽂는다는 것
  - SessionStart, UserPromptSubmit, PreToolUse, PostToolUse를 설명한다.
  - stdout이 context/decision으로 연결되는 원리를 설명한다.
  - trust/review가 보안 모델임을 설명한다.

- [x] 06. Case study: Ponytail은 왜 강하게 작동하는가
  - manifest의 `hooks` 필드, SessionStart, UserPromptSubmit, 상태 파일, mode tracker를 설명한다.
  - 단순 프롬프트와 runtime mode의 차이를 보여준다.

- [x] 07. Case study: Caveman으로 보는 surface와 adapter의 차이
  - “코드 존재”와 “runtime discovery”는 다르다는 점을 중심으로 설명한다.
  - Codex plugin manifest에 hooks가 없어서 Settings > Hooks에 안 뜬다는 사례를 다룬다.
  - AGENTS.md, Skill, Hook, Plugin surface가 같은 behavior를 어떻게 다르게 만드는지 설명한다.

- [ ] 08. Memory design: 모델이 기억하는 게 아니라 하네스가 가져온다
  - recent history, summary, retrieval, stale memory, source discipline을 설명한다.
  - memory를 session/cache/DB search/ranking 문제로 비유한다.

- [ ] 09. Tool design: 모델에게 함수를 맡기는 법
  - tool schema는 모델을 위한 API 문서라는 관점을 설명한다.
  - name, description, input schema, output contract, when to call, when not to call을 다룬다.

- [ ] 10. 나만의 Harness Lab 설계하기
  - 바로 제품을 만들지 않고 관찰 장치를 만든다는 관점을 설명한다.
  - Context Inspector, Hook Playground, Mini Mode Plugin, Memory Harness, Skill Authoring Lab을 실험으로 소개한다.

## 2차 확장: AX 개발자 성장 커리큘럼

현재 repo는 1~2단계에 초점을 둔다.

```text
1단계: Agent literacy
2단계: Harness engineering
```

나중에 확장할 방향:

```text
3단계: Workflow automation
4단계: Evaluation & operations
5단계: AX delivery
```

후속 장 후보:

- [ ] 11. AX 개발자는 무엇을 하는가
  - AX를 단순 LLM API 개발이 아니라 업무 분석, 자동화 설계, 현업 정착, 성과 측정까지 포함하는 역할로 설명한다.
  - 웹 개발자의 API/DB/권한/운영 경험이 왜 강점인지 다룬다.

- [ ] 12. AX 프로젝트는 어떻게 발견하고 설계하는가
  - 어떤 업무가 agent에 적합한지, 어떤 업무는 일반 CRUD/스크립트/자동화가 더 나은지 판단한다.
  - 업무 flow를 쪼개고 병목, 반복 작업, human-in-the-loop 지점을 찾는 법을 다룬다.

- [ ] 13. 평가와 운영: 데모에서 실제 업무 도구로
  - agent 품질 측정, 실패 로그, 재시도, 권한, 보안, 비용, latency, 사용자 피드백, ROI를 다룬다.

## 리라이트 작업 순서

한 번에 전체를 갈아엎지 않는다. 한 장씩 끝낸다.

```text
1. legacy 문서 확인
2. 새 장 작성
3. 링크 검증
4. 체크리스트 업데이트
5. commit
6. push
```

첫 작업은 `01. 모델은 왜 기억하는 것처럼 보일까?`다.

## 완료 기준

각 장은 다음 기준을 만족해야 한다.

- [ ] 처음 읽는 웹 개발자가 따라올 수 있다.
- [ ] 핵심 개념과 이유가 있다.
- [ ] 웹 개발자 비유가 하나 이상 있다.
- [ ] 작은 예시가 있다.
- [ ] Codex와 연결되는 지점이 있다.
- [ ] 흔한 오해를 하나 이상 짚는다.
- [ ] 장 끝에 요약 또는 생각해볼 질문이 있다.

## 레거시 문서

개편 전 문서는 아래에 보존한다.

- [2026-06-25 개편 전 README](../legacy/2026-06-25-pre-course-rewrite/README.md)
- [2026-06-25 개편 전 docs](../legacy/2026-06-25-pre-course-rewrite/docs/)
- [2026-06-25 개편 전 labs](../legacy/2026-06-25-pre-course-rewrite/labs/)
