[← README](../README.md)

# 4장. 왜 지시문 넣는 방법이 이렇게 많을까?

## 이 장에서 배울 것

3장에서는 모델이 답하기 전에 여러 context layer가 조립된다는 것을 봤다.

이제 자연스럽게 이런 질문이 생긴다.

```text
그럼 그냥 프롬프트 하나에 다 쓰면 안 되나?
왜 AGENTS.md, Skill, Hook, Plugin, MCP처럼 표면이 여러 개인가?
```

처음엔 전부 “모델에게 뭔가 알려주는 방법”처럼 보인다. 하지만 실제로는 쓰임이 다르다.

이 장의 중심 문장은 이것이다.

```text
표면이 나뉘는 이유는 내용이 달라서만이 아니라,
scope, timing, execution, distribution이 다르기 때문이다.
```

짧게 말하면:

```text
AGENTS.md = durable project guidance
Skill = on-demand workflow manual
Hook = lifecycle command
Plugin = distribution unit
MCP/App = external capability
```

## 먼저 직관으로 이해하기

웹 앱을 만든다고 생각해보자.

모든 설정과 로직을 controller 파일 하나에 다 때려 넣을 수는 있다.

```text
auth rule
DB connection
request validation
business logic
background job
cron schedule
deployment config
API docs
```

작은 장난감이라면 돌아갈 수도 있다. 하지만 시스템이 커지면 곧 무너진다. 그래서 웹 개발에서는 역할을 나눈다.

```text
.env
→ 환경별 설정

middleware
→ 요청마다 검사할 공통 로직

service
→ 비즈니스 로직

job/worker
→ 비동기 작업

OpenAPI schema
→ API 계약

package
→ 배포 단위
```

Codex도 비슷하다. 모델에게 줄 지시나 능력을 한 군데에 다 넣지 않는다.

```text
repo 규칙은 AGENTS.md
특정 작업 절차는 Skill
이벤트마다 실행되어야 하는 검사는 Hook
남에게 설치시킬 묶음은 Plugin
외부 시스템 연결은 MCP/App
```

표면이 많아 보이는 것은 복잡하게 만들기 위해서가 아니다. 서로 다른 수명과 역할을 섞지 않기 위해서다.

## 정확한 개념: 네 가지 축

어떤 표면을 쓸지 고를 때는 네 가지 질문을 하면 된다.

```text
1. Scope
   어디까지 적용되는가?
   한 번의 답변, 한 repo, 한 사용자, 모든 설치 사용자?

2. Timing
   언제 모델에게 보이는가?
   세션 시작, 필요할 때, 매 prompt, tool 실행 전후?

3. Execution
   문서처럼 읽히는가, 실제 command가 실행되는가?

4. Distribution
   나만 쓰는가, repo 팀원이 쓰는가, 다른 사람이 설치하게 할 것인가?
```

이 네 축으로 보면 각 표면이 선명해진다.

| 표면 | Scope | Timing | Execution | Distribution |
|---|---|---|---|---|
| Prompt | 현재 요청 | 지금 | 없음 | 없음 |
| AGENTS.md | global/repo/folder | 세션 시작 instruction chain | 없음 | 파일 공유 |
| Skill | repo/user/plugin | 필요할 때 선택 | 선택적으로 script 사용 가능 | skill/plugin |
| Hook | config/plugin layer | lifecycle event | command 실행 | config/plugin |
| Plugin | 설치 단위 | 설치/활성화 후 | skills/hooks/MCP 제공 | 배포 단위 |
| MCP/App | 외부 시스템 연결 | tool 필요 시 | 외부 tool 실행 | server/app |

이 표를 외울 필요는 없다. 중요한 감각은 이것이다.

```text
정적 규칙인가?
작업 매뉴얼인가?
이벤트마다 실행돼야 하나?
배포해야 하나?
외부 시스템과 연결해야 하나?
```

## AGENTS.md: 프로젝트 헌법

`AGENTS.md`는 Codex가 세션 시작 시 instruction chain에 넣는 durable guidance다.

좋은 용도:

```text
repo 규칙
테스트 명령
코딩 컨벤션
리뷰 기준
작업 전 확인할 문서
팀의 금지 사항
```

예:

```md
# AGENTS.md

## Repository rules

- Use pnpm.
- Run `pnpm test` after changing TypeScript files.
- Do not add production dependencies without asking.
- Keep database migrations append-only.
```

이런 규칙은 매번 사용자가 프롬프트에 붙일 필요가 없다. repo의 기본 작업 방식이기 때문이다.

Codex의 discovery 규칙은 대략 이렇다.

```text
Global scope
→ ~/.codex/AGENTS.override.md 우선
→ 없으면 ~/.codex/AGENTS.md

Project scope
→ project root에서 current working directory까지 내려오며 확인
→ 각 directory에서 AGENTS.override.md, AGENTS.md, fallback instruction file 순서

Merge order
→ root에서 current directory 방향으로 이어 붙임
→ 가까운 directory의 guidance가 뒤에 오므로 더 구체적인 규칙처럼 작동
```

주의할 점:

```text
AGENTS.md는 항상 유효한 규칙을 담기에 좋다.
하지만 매 프롬프트마다 파일을 다시 읽는 lifecycle command는 아니다.
```

세션 중간에 수정한 내용은 현재 실행 중인 세션에 바로 반영되지 않을 수 있다. 새 세션이나 재시작이 필요할 수 있다.

## Skill: 필요할 때 꺼내는 작업 매뉴얼

Skill은 특정 작업을 잘 수행하기 위한 재사용 가능한 workflow다.

좋은 용도:

```text
PDF 생성 절차
PR 리뷰 방식
데이터 분석 리포트 작성법
프론트엔드 테스트 절차
특정 framework 작업 규칙
```

Skill은 처음부터 모든 본문이 context에 들어가지 않는다. Codex는 먼저 skill의 name, description, file path 같은 얇은 목록을 알고 있다가, 작업에 맞는 skill이 선택되면 그때 `SKILL.md` 전체를 읽는다. 공식 문서는 이 방식을 progressive disclosure라고 부른다.

그래서 skill의 `description`은 아주 중요하다.

```text
description
→ 사람에게 보여주는 설명
→ 동시에 모델이 “언제 이 skill을 쓸지” 판단하는 trigger
```

너무 넓게 쓰면 아무 때나 튀어나온다. 너무 좁게 쓰면 필요할 때 선택되지 않는다.

예:

```md
---
name: pr-review
description: Use when reviewing a GitHub pull request for correctness, risk, tests, and actionable comments.
---

Review the PR by checking:

1. What changed
2. Whether tests cover the risk
3. Whether comments are actionable
4. Whether the implementation is simpler than alternatives
```

Skill은 “항상 지켜야 할 repo 규칙”보다 “특정 일을 할 때 꺼내는 절차서”에 가깝다.

## Hook: lifecycle에 꽂는 command

Hook은 instruction file이 아니다. 특정 lifecycle event에 실행되는 command다.

예:

```text
SessionStart
UserPromptSubmit
PreToolUse
PostToolUse
Stop
```

좋은 용도:

```text
세션 시작 때 동적 context 주입
매 prompt마다 상태 확인
slash command 감지
위험한 tool 사용 차단
tool 실행 후 로그 기록
conversation summary 저장
```

예를 들어 Ponytail은 hook을 사용해 세션 시작 시 mode instruction을 주입하고, 사용자 prompt마다 mode 변경 명령을 감지한다.

```text
SessionStart
→ Ponytail rules 주입

UserPromptSubmit
→ /ponytail ultra 같은 명령 감지
→ 상태 파일 갱신
```

Hook은 command를 실행할 수 있다. 그래서 trust/review가 필요하다.

```text
문서처럼 읽히는 것
→ AGENTS.md, Skill

실제로 실행되는 것
→ Hook
```

이 차이는 크다. Hook은 파일을 읽고, 상태를 쓰고, 외부 명령을 실행하고, 특정 tool 사용을 막을 수 있다. 그러므로 안전 모델과 승인 흐름이 필요하다.

또한 hook output은 event마다 의미가 다르다.

```text
SessionStart
→ stdout/additionalContext가 extra developer context로 들어갈 수 있음

PreToolUse
→ plain text stdout은 context로 들어가지 않음
→ JSON decision으로 tool 실행을 제어할 수 있음
```

이벤트마다 “출력이 context가 되는지, decision이 되는지, 로그에만 의미가 있는지”가 다르다. Hook을 만들 때는 event별 계약을 확인해야 한다.

## Plugin: 설치 가능한 묶음

Plugin은 기능 하나가 아니라 배포 단위다.

Plugin은 다음을 묶을 수 있다.

```text
skills/
hooks/
MCP config
assets
app mappings
manifest
```

최소 plugin은 skill 하나만 포함할 수도 있다.

```text
my-plugin/
  .codex-plugin/
    plugin.json
  skills/
    my-skill/
      SKILL.md
```

Hook이 있는 plugin이라면 manifest에서 hook 파일을 가리켜야 한다.

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "skills": "./skills/",
  "hooks": "./hooks/hooks.json"
}
```

중요한 점:

```text
Plugin 자체가 항상 context에 들어가는 것은 아니다.
Plugin이 제공한 skill description, loaded SKILL.md,
hook output, MCP tool definition 등이 context assembly에 참여한다.
```

즉 plugin은 “context 조각”이라기보다 “context 조각과 실행 능력을 설치 가능하게 묶은 package”다.

## MCP/App: 외부 세계로 나가는 통로

MCP나 App/Connector는 모델에게 외부 시스템을 tool로 연결하는 표면이다.

좋은 용도:

```text
GitHub issue/PR 읽기
Gmail 검색
Slack 메시지 조회/작성
Notion 문서 생성
Google Drive 파일 읽기
사내 API 호출
```

MCP/App은 단순 instruction이 아니다. 외부 데이터와 외부 action을 연결한다.

```text
AGENTS.md
→ “이 repo에서는 이렇게 일해라”

Skill
→ “이 작업은 이 절차로 해라”

Hook
→ “이 lifecycle event에 이 command를 실행해라”

MCP/App
→ “이 외부 시스템의 tool을 사용할 수 있다”
```

외부 action이 들어가므로 권한, 인증, 감사 로그, 사용자 승인 같은 문제가 중요해진다.

## 작은 예시: “항상 짧게 답해줘”는 어디에 둘까?

이제 같은 요구를 여러 표면에 넣어보자.

요구:

```text
항상 짧게 답해줘.
```

### 1. 한 번만 필요하다면 prompt

```text
이번 답변은 짧게 해줘.
```

현재 요청에만 적용하면 된다.

### 2. 모든 repo에서 개인 취향으로 쓰고 싶다면 global AGENTS.md

```md
# ~/.codex/AGENTS.md

- Prefer concise answers unless the user asks for detail.
```

개인 기본값으로 적합하다.

### 3. 이 repo에서만 작업 스타일로 쓰고 싶다면 repo AGENTS.md

```md
# AGENTS.md

- Keep implementation notes short.
- Prefer direct file links over long explanations.
```

팀이나 프로젝트 규칙이라면 여기가 맞다.

### 4. 특정 작업에서만 짧은 review를 원한다면 Skill

```md
---
name: concise-review
description: Use when the user asks for a short, actionable code review.
---

Return only actionable findings. One line per issue.
```

작업별 workflow라면 skill이 맞다.

### 5. 매 prompt마다 mode를 강하게 유지하고 싶다면 Hook

```text
UserPromptSubmit
→ 현재 mode 파일 확인
→ active면 short-answer reminder 주입
```

대화 중 스타일이 자꾸 흐려진다면 hook이 더 강하다.

### 6. 남들이 설치하게 만들고 싶다면 Plugin

```text
short-answer-plugin/
  .codex-plugin/plugin.json
  skills/
  hooks/
```

배포가 목표라면 plugin으로 묶는다.

같은 “짧게 답해줘”라도 scope와 timing이 다르면 위치가 달라진다.

## Codex에서는 어떻게 보이는가

Codex 공식 표면을 이 장 관점으로 다시 정리하면 이렇다.

```text
Prompt
→ 지금 이 turn에만 강하게 작용하는 요청

AGENTS.md
→ global/project/folder 단위 durable guidance

Skill
→ 필요할 때 선택되어 로드되는 workflow

Hook
→ lifecycle event에 실행되는 command

Plugin
→ skills/hooks/MCP/assets를 설치 가능한 단위로 묶은 package

MCP/App
→ 외부 시스템의 data/action을 tool로 제공
```

이들은 서로 대체 관계가 아니라 조합 관계다.

예를 들어 Ponytail 같은 mode는 여러 표면을 함께 쓴다.

```text
Skill
→ Ponytail의 행동 원칙 설명

Hook
→ SessionStart/UserPromptSubmit에서 mode 유지

Plugin
→ skill과 hook을 설치 가능한 package로 묶음
```

그래서 Ponytail은 단순 prompt보다 강하게 느껴진다. 문구 하나가 아니라, distribution + lifecycle + instruction이 결합되어 있기 때문이다.

## 흔한 오해

### 오해 1. “그냥 AGENTS.md에 다 넣으면 되지 않나?”

작은 개인 규칙이라면 가능하다. 하지만 모든 것을 AGENTS.md에 넣으면 문제가 생긴다.

```text
특정 작업에만 필요한 절차가 항상 context를 차지함
동적으로 상태를 확인할 수 없음
tool 사용 전 차단 같은 실행 로직을 할 수 없음
남에게 설치 가능한 package로 관리하기 어려움
```

AGENTS.md는 프로젝트 헌법이지, 모든 자동화의 쓰레기통이 아니다.

### 오해 2. “Skill은 AGENTS.md랑 같은 것 아닌가?”

둘 다 instruction을 담을 수 있지만 로드 방식이 다르다.

```text
AGENTS.md
→ 세션 시작 시 durable guidance

Skill
→ 작업에 맞을 때 선택되어 전체 본문 로드
```

항상 필요한 규칙은 AGENTS.md, 필요할 때 꺼낼 절차는 Skill이 좋다.

### 오해 3. “Hook은 더 강한 prompt다”

Hook은 prompt가 아니라 command다.

Hook은 context를 주입할 수도 있지만, 그보다 넓은 일을 할 수 있다.

```text
상태 파일 읽기/쓰기
prompt 검사
tool 사용 차단
로그 기록
외부 command 실행
```

그래서 hook은 강력하지만 trust/review가 필요하다.

### 오해 4. “Plugin을 만들면 그 자체가 agent다”

아니다.

Plugin은 배포 단위다. Plugin 안에 skill만 있을 수도 있고, hook만 있을 수도 있고, MCP 설정이 있을 수도 있다. Agent처럼 행동하는지는 plugin이 제공하는 instruction, hook, tool, runtime 설계에 달려 있다.

## 선택 기준

무언가를 어디에 둘지 헷갈리면 아래 질문을 한다.

```text
1. 이번 요청에만 필요한가?
   → Prompt

2. repo나 사용자 기본 규칙인가?
   → AGENTS.md

3. 특정 작업을 할 때만 필요한 절차인가?
   → Skill

4. 특정 lifecycle event마다 실행되어야 하나?
   → Hook

5. 남이 설치할 수 있게 묶어야 하나?
   → Plugin

6. 외부 시스템의 data/action이 필요한가?
   → MCP/App/Connector
```

더 짧게:

```text
정적 규칙 → AGENTS.md
작업 매뉴얼 → Skill
이벤트 실행 → Hook
배포 묶음 → Plugin
외부 연결 → MCP/App
```

## 왜 중요한가

표면을 잘못 고르면 에이전트가 지저분해진다.

예:

```text
매번 검사해야 하는 보안 정책을 AGENTS.md에만 적음
→ 모델이 잊거나 무시할 수 있고, 실제 차단은 못 함

특정 작업에서만 필요한 긴 절차를 AGENTS.md에 넣음
→ 모든 작업 context를 불필요하게 차지함

개인 실험용 hook을 plugin으로 과하게 포장함
→ 배포/승인/유지 비용만 늘어남

남에게 설치시킬 workflow를 로컬 AGENTS.md로만 관리함
→ 재사용과 공유가 어려움
```

좋은 하네스 엔지니어링은 “무엇을 말할까”뿐 아니라 “그 말을 어느 표면에 둘까”를 설계한다.

이 설계가 맞아야 뒤의 Ponytail/Caveman case study도 제대로 보인다.

```text
Ponytail은 왜 Hooks에 뜨는가?
Caveman은 왜 Codex Hooks에 안 떴는가?
AGENTS.md로 넣은 Caveman 규칙과 plugin hook은 왜 다른가?
```

이 질문들의 답은 모두 이 장의 표면 구분에서 나온다.

## 요약

```text
1. Codex의 여러 표면은 모두 “프롬프트 넣는 방법”이 아니라 scope, timing, execution, distribution이 다른 장치다.
2. AGENTS.md는 global/repo/folder 단위 durable guidance다.
3. Skill은 필요할 때 선택되어 로드되는 workflow manual이다.
4. Hook은 lifecycle event에 실행되는 command이며 trust/review가 필요하다.
5. Plugin은 skills/hooks/MCP/assets를 설치 가능한 단위로 묶는 distribution unit이다.
6. MCP/App은 외부 시스템의 data/action을 tool로 연결하는 표면이다.
7. 좋은 설계는 같은 내용을 무조건 한 파일에 넣지 않고, 수명과 역할에 맞는 표면에 둔다.
```

## 생각해볼 질문

1. 내 프로젝트의 기본 규칙 중 AGENTS.md에 들어가야 할 것은 무엇인가?
2. 특정 작업에서만 필요한 절차 중 Skill로 빼면 좋은 것은 무엇인가?
3. 매번 실행되어야 하거나 실제 차단이 필요한 규칙 중 Hook이 필요한 것은 무엇인가?
4. 내가 만든 workflow를 남이 설치하게 만들려면 Plugin으로 묶어야 할 것은 무엇인가?
5. 지금까지 “프롬프트”라고 뭉뚱그려 생각했던 것 중 사실은 서로 다른 표면에 있어야 할 것은 무엇인가?
