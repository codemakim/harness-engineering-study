[← README](../README.md)

# 10장. 나만의 Harness Lab 설계하기

## 이 장에서 배울 것

여기까지 우리는 agent harness를 여러 조각으로 나눠 봤다.

```text
context assembly
AGENTS.md
skills
hooks
plugins
memory
tools
MCP
approval
observation
```

이제 자연스럽게 이런 생각이 든다.

```text
좋아. 그럼 나도 내 agent를 만들고 싶다.
무엇부터 만들면 될까?
```

보통 여기서 바로 제품을 만들고 싶어진다.

```text
일정 등록 agent
문서 요약 agent
PR 리뷰 agent
업무 자동화 agent
나만의 Codex plugin
```

만들 수 있다.

하지만 공부 목적이라면 첫 제품은 조금 늦춰도 된다.

이 장의 중심 문장은 이것이다.

```text
처음부터 agent 제품을 만들지 말고,
agent가 어떻게 동작하는지 보이게 만드는 관찰 장치를 먼저 만들어라.
```

훌륭한 harness engineer는 모델에게 멋진 말을 시키는 사람이 아니다.

모델 앞뒤에서 실제로 무슨 일이 일어나는지 볼 수 있게 만드는 사람이다.

```text
무슨 context가 들어갔는가?
어떤 instruction이 영향을 줬는가?
어떤 tool이 왜 호출됐는가?
hook은 언제 실행됐는가?
memory는 어떤 기준으로 들어왔는가?
실패 output은 다음 판단에 어떻게 쓰였는가?
```

이걸 볼 수 있으면 agent를 고칠 수 있다.

안 보이면 감으로 프롬프트를 만지게 된다.

## 먼저 직관으로 이해하기

웹 개발을 처음 배울 때를 떠올려보자.

처음부터 대형 SaaS를 만들면 공부가 잘 될까?

아마 아니다.

오히려 실력이 늘었던 순간은 이런 작은 실험을 할 때다.

```text
request가 controller까지 어떻게 가는지 찍어보기
middleware 순서 바꿔보기
cookie와 session store 연결해보기
DB query log 보기
cache hit/miss 출력하기
auth guard가 어디서 막는지 확인하기
queue worker가 실패하면 retry되는지 보기
```

이 실험들은 제품으로는 작다.

하지만 시스템을 보는 눈을 만든다.

agent harness도 같다.

처음부터 “완성형 비서”를 만들면 너무 많은 문제가 한꺼번에 섞인다.

```text
prompt 문제인가?
context 문제인가?
tool schema 문제인가?
runner validation 문제인가?
memory retrieval 문제인가?
hook timing 문제인가?
permission 문제인가?
사용자 UX 문제인가?
```

다 섞이면 디버깅이 안 된다.

그래서 공부용 lab은 작아야 한다.

단, 작지만 보이는 것이 많아야 한다.

```text
좋은 lab
→ 기능은 작음
→ 관찰 지점은 선명함
→ 실패가 재현됨
→ 로그가 남음
→ 한 번에 하나의 개념을 배움
```

이 장에서는 여섯 개의 lab을 설계한다.

```text
1. Context Inspector
2. Hook Playground
3. Tool Runner / Observation Lab
4. Memory Harness
5. Skill Authoring Lab
6. Mini Mode Plugin
```

전부 거창한 제품이 아니다.

하지만 이 여섯 개를 제대로 만들고 관찰하면, 앞 장들의 개념이 손에 잡힌다.

## Lab을 만들기 전에 정해야 할 원칙

공부용 lab은 제품과 목표가 다르다.

제품의 목표:

```text
사용자 문제를 안정적으로 해결한다.
```

lab의 목표:

```text
agent harness의 한 동작을 눈으로 확인한다.
```

그래서 lab에서는 일부러 범위를 줄인다.

```text
실제 Gmail 전송
→ 하지 않는다

draft_email JSON 만들기
→ 한다

실제 캘린더 생성
→ 처음엔 하지 않는다

create_calendar_event dry-run 결과 만들기
→ 한다

복잡한 vector DB
→ 처음엔 하지 않는다

로컬 markdown memory 5개에서 검색
→ 한다
```

게으른 게 아니다.

학습 신호를 깨끗하게 만드는 것이다.

처음부터 외부 API, OAuth, 결제, 배포, UI까지 붙이면 harness를 배우는 게 아니라 잡일과 싸우게 된다.

잡일도 언젠가 배워야 한다.

하지만 지금은 agent harness의 신경계를 보는 시간이 더 중요하다.

## Lab 1. Context Inspector

첫 번째 lab은 Context Inspector다.

목표는 단순하다.

```text
모델에게 무엇이 들어갔는지 볼 수 있게 만든다.
```

1장에서 우리는 말했다.

```text
LLM은 기억하는 것이 아니라 context를 읽는다.
```

그렇다면 가장 먼저 만들어야 할 것은 “context를 보는 장치”다.

### 무엇을 관찰할까?

Context Inspector는 한 번의 요청을 여러 layer로 나눠 보여준다.

```text
system/developer instruction
user prompt
recent history
AGENTS.md 또는 project guidance
skill metadata
loaded skill body
tool schema
memory snippets
hook output
tool output / observation
environment context
```

실제 Codex 내부 context 전체를 그대로 볼 수 있다는 뜻은 아니다.

제품 내부 구현과 정책상 노출되지 않는 layer가 있을 수 있다.

Context Inspector는 hosted Codex의 실제 내부 payload를 완전히 재현하는 도구가 아니다. 내가 만든 작은 harness에서 어떤 layer를 어떤 이유로 조립했는지 관찰하는 장치다.

공부용 lab에서는 직접 만든 작은 harness에서 이 구조를 흉내 내면 된다.

```text
input/
  system.md
  developer.md
  history.json
  memories/
  tools.json
  user-prompt.md

scripts/
  assemble-context.js

out/
  assembled-context.md
  context-report.json
```

핵심은 모델 호출보다 assembly 과정을 보는 것이다.

### 작은 예시

사용자 prompt:

```text
내일 오후 3시에 병원 예약 일정 등록해줘.
```

Context Inspector는 이렇게 report를 만든다.

```json
{
  "layers": [
    {
      "name": "user_prompt",
      "tokensApprox": 18,
      "source": "current turn"
    },
    {
      "name": "memory",
      "tokensApprox": 42,
      "source": "memories/user-preferences.md",
      "reason": "contains user's default timezone"
    },
    {
      "name": "tool_schema",
      "tokensApprox": 310,
      "source": "tools/calendar.json",
      "reason": "calendar action available"
    }
  ],
  "warnings": [
    "Date is relative: 내일",
    "Timezone resolved from memory: Asia/Seoul",
    "Write action requires confirmation or dry-run"
  ]
}
```

이 report를 보면 agent가 왜 timezone을 알았는지, 왜 바로 실행하지 않고 확인해야 하는지 보인다.

여기서 `tokensApprox`는 정확한 과금 단위가 아니다. layer별 상대적 크기를 보기 위한 근사치다. 실제 token 수는 사용하는 모델과 tokenizer에 따라 달라진다.

### 왜 중요한가?

프롬프트를 잘 쓰는 것보다 먼저 해야 할 일이 있다.

```text
프롬프트가 어디에 들어가는지 알아야 한다.
```

같은 문장도 들어가는 위치에 따라 힘이 다르다.

```text
system/developer instruction
→ 강한 기본 제약

AGENTS.md
→ repo 작업 전반의 durable guidance

skill body
→ 특정 workflow가 선택됐을 때만 들어오는 절차

hook output
→ lifecycle event 때 runtime이 만든 신호

memory
→ 과거에서 검색되어 들어온 참고 정보

user prompt
→ 현재 사용자의 직접 요청
```

Context Inspector를 만들면 “모델이 이상하다”는 말을 더 정확히 바꿀 수 있다.

```text
필요한 context가 안 들어갔다.
너무 많은 context가 들어갔다.
instruction 위치가 약했다.
memory가 오래됐다.
tool output이 너무 시끄럽다.
hook output이 매 turn 들어와서 noise가 됐다.
```

이게 디버깅 언어다.

## Lab 2. Hook Playground

두 번째 lab은 Hook Playground다.

목표는 hook을 추상 개념이 아니라 lifecycle event로 보는 것이다.

5장에서 우리는 말했다.

```text
Hook은 모델에게 무엇을 말할지보다,
언제 runtime에 개입할지를 설계하는 장치다.
```

Codex 공식 문서 기준으로 hooks는 agentic loop 안에 script를 주입하는 확장 프레임워크다. `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop` 같은 event에 command hook을 붙일 수 있고, non-managed command hook은 review/trust를 거쳐야 실행된다.

Hook Playground는 이 원리를 작게 재현한다.

### 무엇을 만들까?

처음 버전은 진짜 Codex hook을 만들 필요도 없다.

작은 Node script 하나로 충분하다.

```text
scripts/run-turn.js

1. session_start hooks 실행
2. user_prompt_submit hooks 실행
3. fake model decision 생성
4. pre_tool_use hooks 실행
5. fake tool 실행
6. post_tool_use hooks 실행
7. stop hooks 실행
8. event log 출력
```

진짜 모델을 붙이지 않아도 된다.

처음에는 fake model이면 충분하다.

```js
function fakeModel(prompt) {
  if (prompt.includes("일정")) {
    return {
      type: "tool_call",
      toolName: "create_calendar_event",
      args: { title: "병원 예약" }
    };
  }

  return { type: "message", content: "답변" };
}
```

이 lab의 목적은 모델 성능이 아니다.

hook이 언제 실행되는지 보는 것이다.

### 관찰할 것

Hook Playground는 event log를 남긴다.

```json
[
  {
    "event": "SessionStart",
    "hook": "load-mode",
    "output": "mode=ponytail"
  },
  {
    "event": "UserPromptSubmit",
    "hook": "prompt-lint",
    "output": "no secrets detected"
  },
  {
    "event": "PreToolUse",
    "tool": "create_calendar_event",
    "hook": "write-action-policy",
    "output": "requires dry-run"
  },
  {
    "event": "PostToolUse",
    "tool": "create_calendar_event",
    "hook": "summarize-result",
    "output": "dry-run event preview created"
  }
]
```

이렇게 보면 hook은 “프롬프트 파일”이 아니다.

runtime event에 붙은 code다.

### 실험 질문

Hook Playground에서 이런 실험을 해본다.

```text
UserPromptSubmit에서 매번 같은 reminder를 넣으면 어떻게 되는가?
PreToolUse에서 write tool을 막으면 모델은 다음에 무엇을 하는가?
PostToolUse output이 너무 길면 다음 판단이 흐려지는가?
SessionStart에만 mode를 넣으면 긴 대화에서 얼마나 버티는가?
Stop hook에서 요약을 만들면 memory 후보로 쓸 수 있는가?
```

이 실험을 해보면 Ponytail이 왜 강하게 작동했는지 이해가 쉬워진다.

단순히 “말투 지시문”이 강한 게 아니다.

반복 주입되는 lifecycle 신호가 강한 것이다.

단, 이 playground는 lifecycle 감각을 익히기 위한 toy harness다. 실제 Codex hook은 event마다 stdin payload, plain stdout 처리, JSON output contract, trust/review flow가 다르므로 실제 hook으로 옮길 때는 공식 event contract를 다시 확인해야 한다.

## Lab 3. Tool Runner / Observation Lab

세 번째 lab은 Tool Runner / Observation Lab이다.

9장에서 우리는 tool을 이렇게 설명했다.

```text
Tool schema는 모델에게 주는 API 문서이고,
tool runner는 실제 API 서버다.
```

그러면 lab에서는 schema와 runner를 분리해서 봐야 한다.

```text
schema
→ 모델이 어떤 tool을 어떤 arguments로 호출할지 결정하는 재료

runner
→ arguments 검증, 권한 확인, dry-run, 실제 실행, observation 생성
```

모델은 tool call을 제안한다.

실행과 안전은 runner가 책임진다.

### 무엇을 만들까?

작은 calendar tool runner를 만든다.

처음에는 진짜 캘린더 API를 붙이지 않는다.

fake runner면 충분하다.

```text
tools/
  calendar-tools.json

scripts/
  run-tool.js

out/
  tool-observation.json
```

tool은 네 개만 둔다.

```text
search_calendar_events
preview_calendar_event
create_calendar_event
delete_calendar_event
```

그리고 runner에는 일부러 다른 정책을 건다.

```text
search_calendar_events
→ read tool, 자동 실행 가능

preview_calendar_event
→ dry-run, 자동 실행 가능

create_calendar_event
→ write tool, 사용자 승인 필요

delete_calendar_event
→ destructive tool, 명시적 confirmation 필요
```

이 lab에서 중요한 것은 “진짜 일정이 생겼는가?”가 아니다.

runner가 모델 arguments를 어떻게 다루는지 보는 것이다.

### 작은 예시

모델이 이런 tool call을 만들었다고 하자.

```json
{
  "toolName": "create_calendar_event",
  "args": {
    "title": "병원 예약",
    "startAt": "tomorrow 3pm"
  }
}
```

나쁜 runner는 이걸 그대로 실행하려고 한다.

```text
calendarApi.createEvent(args)
```

좋은 runner는 막는다.

```json
{
  "ok": false,
  "errorCode": "AMBIGUOUS_DATETIME",
  "message": "startAt is not an ISO 8601 datetime with timezone.",
  "retryable": true,
  "userFixable": true,
  "partialResult": {
    "title": "병원 예약"
  },
  "suggestedNextStep": "Ask the user for the exact date and timezone, or use a preview tool first."
}
```

이 output은 실패지만 쓸모 있는 observation이다.

모델은 다음 turn에서 이렇게 복구할 수 있다.

```text
내일 오후 3시는 어떤 시간대 기준인가요?
기본 시간대 Asia/Seoul로 등록해도 될까요?
```

### 관찰할 것

Tool Runner Lab에서는 이런 것을 본다.

```text
모델이 tool name과 description만 보고 read/write를 구분하는가?
ambiguous date/time이면 create tool을 막는가?
dry-run result와 real execution result가 다르게 반환되는가?
runner validation 실패가 복구 가능한 observation으로 돌아오는가?
destructive tool은 approval 없이는 실행되지 않는가?
같은 write 요청이 두 번 들어오면 idempotency로 막히는가?
observation이 너무 커져 context noise가 되지 않는가?
```

작은 runner rule만으로도 9장의 핵심을 확인할 수 있다.

```text
schema는 모델을 유도한다.
runner는 시스템을 지킨다.
observation은 다음 판단을 만든다.
```

## Lab 4. Memory Harness

네 번째 lab은 Memory Harness다.

목표는 “기억”을 모델 능력이 아니라 retrieval pipeline으로 보는 것이다.

8장에서 우리는 memory를 이렇게 설명했다.

```text
Memory는 모델의 신비한 능력이 아니라,
하네스가 과거 정보를 고르고 요약해서 context에 다시 넣는 설계다.
```

Codex 공식 문서도 memory를 prior threads에서 useful context를 future work로 가져오는 기능으로 설명한다. 동시에 team guidance처럼 반드시 지켜야 하는 것은 memory보다 `AGENTS.md`나 checked-in docs에 두라고 권한다.

Memory Harness는 이 차이를 실험한다.

### 무엇을 만들까?

처음 버전은 vector DB가 필요 없다.

markdown 파일 5개와 간단한 keyword search면 충분하다.

```text
memories/
  user-preferences.md
  project-habits.md
  calendar-rules.md
  old-wrong-memory.md
  private-do-not-use.md

scripts/
  retrieve-memory.js
  assemble-context.js

out/
  memory-report.json
```

각 memory에는 metadata를 붙인다.

```md
---
id: calendar-rules
updatedAt: 2026-07-01
confidence: high
sensitivity: normal
---

사용자의 기본 timezone은 Asia/Seoul이다.
일정 등록은 실행 전에 dry-run preview를 먼저 보여준다.
```

retriever는 질문을 보고 후보를 고른다.

```json
{
  "query": "내일 오후 3시에 병원 예약 일정 등록해줘",
  "selected": [
    {
      "id": "calendar-rules",
      "reason": "contains timezone and calendar dry-run rule",
      "confidence": "high"
    }
  ],
  "rejected": [
    {
      "id": "old-wrong-memory",
      "reason": "stale updatedAt"
    },
    {
      "id": "private-do-not-use",
      "reason": "sensitivity policy"
    }
  ]
}
```

### 관찰할 것

Memory Harness의 핵심 질문은 이것이다.

```text
무엇을 저장할 것인가?
무엇을 검색할 것인가?
무엇을 버릴 것인가?
무엇을 context에 넣을 것인가?
무엇은 AGENTS.md로 승격해야 하는가?
```

memory는 편리하지만 위험하다.

```text
오래된 기억
→ 현재와 충돌할 수 있음

출처 없는 기억
→ 검증하기 어려움

민감한 기억
→ privacy 문제

너무 많은 기억
→ context noise

반드시 지켜야 하는 규칙
→ memory가 아니라 durable instruction/document가 맞음
```

따라서 Memory Harness는 “검색 잘하기”만 보는 실험이 아니다.

source discipline을 배우는 실험이다.

좋은 Memory Harness는 선택한 memory뿐 아니라 버린 memory와 버린 이유를 기록한다. 그래야 stale filtering, privacy filtering, scope mismatch를 디버깅할 수 있다.

### 작은 실패 실험

일부러 오래된 memory를 넣어본다.

```md
---
id: old-timezone
updatedAt: 2025-01-01
confidence: low
---

사용자의 timezone은 America/Los_Angeles다.
```

그리고 현재 질문을 던진다.

```text
내일 오후 3시에 일정 넣어줘.
```

좋은 harness는 이렇게 경고해야 한다.

```json
{
  "warning": "Conflicting timezone memories",
  "candidates": [
    {
      "id": "calendar-rules",
      "timezone": "Asia/Seoul",
      "updatedAt": "2026-07-01",
      "confidence": "high"
    },
    {
      "id": "old-timezone",
      "timezone": "America/Los_Angeles",
      "updatedAt": "2025-01-01",
      "confidence": "low"
    }
  ],
  "decision": "Use newer high-confidence memory, or ask user if action is high-risk."
}
```

이런 실험을 해야 memory가 “마법의 저장소”가 아니라는 감각이 생긴다.

## Lab 5. Skill Authoring Lab

다섯 번째 lab은 Skill Authoring Lab이다.

목표는 좋은 skill description과 workflow instruction을 쓰는 법을 배우는 것이다.

Codex 공식 문서 기준으로 skill은 `SKILL.md`와 optional scripts/references를 가진 directory다. Codex는 먼저 skill의 name, description, file path 같은 metadata로 discover하고, task와 맞는다고 판단할 때 full `SKILL.md`를 읽는다. 이 progressive disclosure 때문에 description이 매우 중요하다.

### 무엇을 만들까?

학습용 skill 하나를 만든다.

예:

```text
skill name: chapter-feedback
목표: 작성된 장을 읽고 입문서 품질 관점에서 피드백하기
```

처음에는 instruction-only skill이면 충분하다.

```text
.agents/
  skills/
    chapter-feedback/
      SKILL.md
```

아래 경로는 학습용 예시다. 실제 Codex에서 repo-local skill을 발견시키려면 현재 공식 문서가 요구하는 skill directory 위치와 plugin/repo surface를 확인해야 한다.

`SKILL.md`:

```md
---
name: chapter-feedback
description: Review one markdown chapter for an intermediate web developer learning agent harness engineering. Use when the user asks for chapter feedback, review, or improvement suggestions.
---

Read the target chapter.

Check:

1. Does it teach one clear concept?
2. Does it explain why the concept matters?
3. Does it use a web developer analogy?
4. Does it avoid false certainty about product internals?
5. Does it include a small example?
6. Does it end with summary or questions?

Return:

- overall judgment
- strongest section
- confusing section
- factual caveat
- concrete edits
```

이 skill은 작지만 많은 것을 가르친다.

```text
description이 trigger를 만든다.
body는 workflow를 만든다.
output format은 review 품질을 안정시킨다.
scope가 좁을수록 잘 작동한다.
```

### 좋은 description 실험

같은 skill을 description만 바꿔본다.

나쁜 description:

```text
Review writing.
```

조금 나은 description:

```text
Review docs.
```

좋은 description:

```text
Review one markdown chapter for an intermediate web developer learning agent harness engineering. Use when the user asks for chapter feedback, review, or improvement suggestions.
```

모델은 skill implementation을 처음부터 다 읽는 것이 아니다.

metadata를 보고 “이 skill을 써야겠다”고 판단한다.

그러니 description은 광고 문구가 아니다.

runtime routing hint다.

### 언제 script를 넣을까?

skill에 script를 넣고 싶어질 수 있다.

하지만 처음부터 넣지 않아도 된다.

```text
instruction-only skill
→ 절차, 기준, 리뷰 rubric에 적합

script 포함 skill
→ deterministic check, file parsing, validation에 적합

reference 포함 skill
→ 긴 예시, 스타일 가이드, domain docs에 적합
```

예를 들어 chapter feedback skill은 처음엔 instruction-only로 충분하다.

나중에 link checker, heading checker, word count checker가 필요해지면 script를 넣으면 된다.

처음부터 넣지 않는다.

이건 Ponytail식으로도 맞다.

필요할 때 추가하면 된다.

## Lab 6. Mini Mode Plugin

여섯 번째 lab은 Mini Mode Plugin이다.

이 lab은 Ponytail과 Caveman을 보고 생긴 질문에서 출발한다.

```text
왜 어떤 것은 Hooks 화면에 뜨고,
왜 어떤 것은 Skill로만 보일까?
왜 같은 지시문도 plugin으로 만들면 더 강하게 느껴질까?
```

정답은 “마법”이 아니다.

packaging surface가 다르기 때문이다.

Codex plugin은 skills, hooks, app/MCP 연결, assets 등을 설치 가능한 단위로 묶는 distribution surface로 볼 수 있다. 실제 지원 필드와 schema는 Codex plugin 문서와 제품 버전에 맞춰 확인해야 한다.

### 무엇을 만들까?

Mini Mode Plugin은 아주 작은 mode를 하나 만든다.

예:

```text
name: microscope
목표: 답변 전에 항상 "관찰 대상 / 증거 / 추론 / 불확실성"으로 생각하게 만들기
```

처음부터 멋진 marketplace plugin을 만들 필요 없다.

학습용 구조만 보면 된다.

```text
microscope-plugin/
  .codex-plugin/
    plugin.json
  skills/
    microscope/
      SKILL.md
  hooks/
    hooks.json
  hooks/
    mode-reminder.js
```

여기서 관찰할 것은 기능이 아니라 surface다.

```text
SKILL.md만 있을 때
→ 사용자가 명시 호출하거나 description이 match될 때만 로드

SessionStart hook이 있을 때
→ 세션 시작 시 mode reminder 가능

UserPromptSubmit hook이 있을 때
→ 매 prompt마다 reminder 가능

plugin manifest에 hooks가 있을 때
→ 설치/신뢰 flow와 Settings > Hooks 표면에 연결 가능
```

### 작은 mode 예시

`SKILL.md`:

```md
---
name: microscope
description: Use when the user wants careful reasoning with explicit evidence and uncertainty.
---

Before answering, separate:

1. observation
2. evidence
3. inference
4. uncertainty

Keep the final answer concise.
```

이건 skill이다.

모델이 이 skill을 선택해야 body가 들어온다.

hook은 다르다.

`mode-reminder.js`는 매 prompt마다 짧은 reminder를 출력할 수 있다.

```text
Microscope active:
- separate evidence from inference
- name uncertainty when material
- keep final concise
```

이 output이 UserPromptSubmit 시점에 들어오면 mode가 더 지속적으로 느껴진다.

### 왜 중요한가?

여기서 배우는 것은 “plugin 만드는 법”보다 더 깊다.

```text
instruction
→ 무엇을 말할지

skill
→ 어떤 workflow를 필요할 때 로드할지

hook
→ 언제 runtime에 개입할지

plugin
→ 이 모든 것을 어떻게 설치/배포할지
```

Ponytail과 Caveman을 비교할 때 헷갈렸던 것도 이것이다.

둘 다 “모델 행동을 바꾸는 지시”처럼 보인다.

하지만 실제 효과는 어느 surface에 붙었는지에 따라 달라진다.

```text
AGENTS.md
→ 세션 시작 instruction chain에 포함

Skill
→ 선택될 때 full instructions 로드

Hook
→ lifecycle event마다 command 실행

Plugin
→ skill/hook/MCP/app을 묶어 설치 가능
```

Mini Mode Plugin은 이 차이를 손으로 확인하는 lab이다.

## 여섯 lab이 이어지는 방식

이 여섯 lab은 따로 노는 장난감이 아니다.

하나의 작은 harness로 합쳐질 수 있다.

```text
User prompt
→ Context Inspector가 layer report 생성
→ Memory Harness가 관련 memory 선택
→ Skill Authoring Lab의 skill metadata가 선택 후보로 들어감
→ Hook Playground가 lifecycle event 실행
→ Mini Mode Plugin이 mode reminder 주입
→ fake model 또는 실제 model 호출
→ tool call / message 생성
→ Tool Runner Lab이 tool call을 검증하고 observation 생성
→ observation 재주입
→ final answer
```

흐름으로 보면 이렇다.

```text
observe
→ inject
→ decide
→ execute
→ observe again
```

이게 agent loop다.

제품을 만들기 전에 이 loop를 작은 규모로 재현해보면, 나중에 진짜 제품을 만들 때 어디를 봐야 할지 안다.

## Lab notebook을 남겨라

이 repo 자체가 좋은 lab notebook이 될 수 있다.

실험할 때마다 결과를 기록한다.

```text
labs/
  001-context-inspector.md
  002-hook-playground.md
  003-tool-runner-observation.md
  004-memory-harness.md
  005-skill-authoring.md
  006-mini-mode-plugin.md
```

각 실험 기록은 길 필요 없다.

이 정도면 충분하다.

```md
# 001. Context Inspector

## 질문

모델 답변 전에 어떤 context layer가 들어가는지 볼 수 있을까?

## setup

- fake system instruction
- user prompt
- memory 3개
- tool schema 2개

## 관찰

- tool schema가 context의 대부분을 차지했다.
- 오래된 memory가 들어오면 답변이 틀어졌다.
- source id가 없으면 나중에 왜 들어왔는지 알 수 없었다.

## 배운 것

context는 내용뿐 아니라 출처와 이유가 필요하다.

## 다음 실험

memory conflict가 있을 때 warning을 만들기.
```

좋은 engineer는 결과만 모으지 않는다.

실패 이유를 모은다.

나중에 plugin이나 agent를 만들 때 이 기록이 설계 판단의 재료가 된다.

## Codex에서는 어떻게 보이는가

Codex를 공부할 때 이 lab들은 실제 표면과 연결된다.

```text
Context Inspector
→ AGENTS.md, skill metadata, tool schema, memory, hook output이 context에 영향을 주는 방식 이해

Hook Playground
→ SessionStart, UserPromptSubmit, PreToolUse, PostToolUse 같은 lifecycle event 이해

Tool Runner / Observation Lab
→ schema, runner validation, approval, error output, observation 품질 이해

Memory Harness
→ Codex memories와 AGENTS.md의 역할 차이 이해

Skill Authoring Lab
→ progressive disclosure와 skill description의 중요성 이해

Mini Mode Plugin
→ skill/hook/plugin packaging surface 차이 이해
```

공식 문서 기준으로도 이 표면들은 서로 경쟁하는 기능이 아니다.

```text
AGENTS.md
→ durable project guidance

Memories
→ prior threads에서 가져오는 useful context

Skills
→ reusable workflow

MCP
→ external tools/context provider 연결

Hooks
→ lifecycle event에 command 실행

Plugins
→ skills/apps/MCP/hook 등을 설치 가능한 bundle로 배포
```

중요한 것은 “어느 기능이 제일 세냐”가 아니다.

```text
어떤 scope인가?
언제 로드되는가?
실행성이 있는가?
배포 단위인가?
권한/신뢰 절차가 필요한가?
```

이 질문으로 surface를 고르는 것이 harness engineering이다.

## 흔한 오해

### 오해 1. “공부하려면 바로 완성형 agent를 만들어야 한다”

아니다.

처음부터 완성형 agent를 만들면 원인들이 섞인다.

학습 초반에는 작은 관찰 장치가 더 낫다.

```text
작은 lab
→ 원인이 선명함

큰 product
→ prompt/tool/memory/auth/UI 문제가 한꺼번에 섞임
```

### 오해 2. “진짜 API를 붙여야 실전이다”

항상 그렇지 않다.

처음에는 fake tool이 더 좋은 실험 장치다.

```text
fake tool
→ 입력/출력/에러/observation 설계에 집중

real API
→ OAuth, rate limit, network, billing, provider quirks가 섞임
```

진짜 API는 나중에 붙이면 된다.

먼저 tool boundary를 이해해야 한다.

### 오해 3. “memory는 많이 넣을수록 좋다”

아니다.

memory도 context budget을 쓰고, 오래되거나 틀릴 수 있다.

좋은 Memory Harness는 많이 넣는 장치가 아니라 잘 고르는 장치다.

### 오해 4. “plugin을 만들면 자동으로 강한 agent가 된다”

아니다.

plugin은 distribution unit이다.

강한 동작은 그 안에 들어 있는 skill, hook, MCP, instruction, runner 설계에서 나온다.

빈 plugin은 그냥 포장지다.

### 오해 5. “hook은 그냥 프롬프트를 더 넣는 기능이다”

아니다.

hook의 본질은 lifecycle timing이다.

```text
SessionStart
→ 시작할 때

UserPromptSubmit
→ 사용자 prompt마다

PreToolUse
→ tool 실행 직전

PostToolUse
→ tool 실행 직후

Stop
→ turn 종료 시점
```

무엇을 말할지도 중요하지만, 언제 말하는지가 hook의 핵심이다.

## 추천 진행 순서

처음 lab을 만든다면 이 순서가 좋다.

```text
1. Context Inspector
2. Hook Playground
3. Tool Runner / Observation Lab
4. Memory Harness
5. Skill Authoring Lab
6. Mini Mode Plugin
```

이유는 단순하다.

먼저 context를 볼 줄 알아야 한다.

그다음 runtime event를 본다.

그다음 tool 실행과 observation을 본다.

그다음 과거 정보를 가져오는 법을 본다.

그다음 reusable workflow를 만든다.

마지막으로 plugin packaging을 한다.

plugin은 멋있지만 마지막이 맞다.

포장하기 전에 내용물이 있어야 한다.

## 요약

```text
1. 처음부터 agent 제품을 만들기보다 관찰 장치를 먼저 만든다.
2. Context Inspector는 모델 입력 layer를 보이게 한다.
3. Hook Playground는 lifecycle event와 runtime 개입 시점을 보이게 한다.
4. Tool Runner / Observation Lab은 schema, runner validation, error output을 분리해서 보게 한다.
5. Memory Harness는 memory를 retrieval pipeline으로 보이게 한다.
6. Skill Authoring Lab은 reusable workflow와 progressive disclosure를 익히게 한다.
7. Mini Mode Plugin은 skill/hook/plugin surface 차이를 실험하게 한다.
8. 좋은 lab은 작고, 재현 가능하고, 관찰 결과가 남는다.
9. Harness engineering 실력은 “모델이 이상하다”를 구체적 원인으로 쪼개는 능력에서 나온다.
```

## 생각해볼 질문

1. 내가 지금 만들고 싶은 agent 제품은 어떤 harness 동작을 검증해야 가능한가?
2. 그 동작만 따로 떼어낸 가장 작은 lab은 무엇인가?
3. 모델 호출 없이 fake model/fake tool로 먼저 관찰할 수 있는 부분은 어디인가?
4. 어떤 context layer가 들어갔는지 나중에 재현할 수 있는가?
5. 실패 output과 event log를 남기고 있는가?
6. 이 규칙은 memory에 둘 것인가, AGENTS.md에 둘 것인가, skill에 둘 것인가, hook에 둘 것인가?
7. plugin으로 포장하기 전에 skill/hook/tool 자체가 충분히 작동하는가?
