[← README](../README.md)

# 13장. Capstone 1: agent-tool-lint 만들기

## 이 장에서 할 일

1~12장까지 우리는 agent harness를 이론적으로 배웠다.

이제 첫 번째 실습 과제를 잡는다.

이번 장의 목표는 실제 구현을 대신 해주는 것이 아니다.

미래의 내가 이 문서를 보고 직접 만들 수 있도록 과제 요구사항을 정확히 고정하는 것이다.

이번 과제는 이것이다.

```text
agent-tool-lint
```

한 줄로 말하면:

```text
LLM agent의 tool/function schema를 검사해
모호한 이름, 느슨한 입력, 누락된 승인 경계,
위험한 side effect를 찾아주는 CLI.
```

비유하면 에이전트용 ESLint다.

ESLint가 JavaScript 코드의 위험한 패턴을 잡아주듯, `agent-tool-lint`는 agent에게 노출되는 tool schema의 위험한 패턴을 잡아준다.

## 왜 이 과제를 하는가

에이전트에서 tool은 모델이 외부 세계를 만지는 손잡이다.

9장에서 우리는 이렇게 말했다.

```text
Tool schema는 모델에게 주는 API 문서이고,
tool runner는 실제 API 서버다.
```

그런데 많은 agent tool schema는 이렇게 생겼다.

```json
{
  "name": "send_email",
  "description": "Send an email.",
  "parameters": {
    "type": "object",
    "properties": {
      "to": { "type": "string" },
      "body": { "type": "string" }
    }
  }
}
```

사람이 보면 대충 이해할 수 있다.

하지만 모델에게는 이 schema가 API 문서다.

이 schema는 중요한 질문에 답하지 못한다.

```text
언제 호출해야 하는가?
언제 호출하면 안 되는가?
초안 작성에도 호출해도 되는가?
사용자 승인 없이 실제 전송해도 되는가?
recipient는 사용자가 직접 제공한 주소여야 하는가?
외부 전송 action인가?
실패하면 어떤 error를 반환하는가?
중복 전송 방지는 있는가?
```

그러면 모델은 사용자의 이런 말을 실제 전송 요청으로 오해할 수 있다.

```text
이렇게 보내면 어때?
```

좋은 agent는 runner에서 막아야 한다.

하지만 더 좋은 개발 도구는 그 전에 schema의 냄새를 알려준다.

```text
이 tool은 이름이 너무 넓다.
description에 when not to call이 없다.
write tool인데 approval boundary가 없다.
external side effect인데 recipient confirmation이 없다.
error contract가 없다.
idempotency 언급이 없다.
```

`agent-tool-lint`는 이 위험을 정적으로 검사한다.

## 이 과제가 연결하는 장

이 과제는 앞 장 세 개를 직접 구현하게 만든다.

```text
9장 Tool design
→ 좋은 tool schema란 무엇인가

11장 Evaluation design
→ good/bad fixture와 regression test로 검증

12장 Safety boundary
→ write/destructive/external action의 경계 검사
```

즉 이 과제는 “에이전트 만들기”가 아니다.

에이전트를 만드는 개발자를 위한 harness quality tool이다.

이게 오히려 첫 실습으로 좋다.

```text
범위가 작다.
외부 API가 필요 없다.
OAuth가 필요 없다.
데이터 유출 위험이 낮다.
테스트하기 쉽다.
문서의 철학을 바로 증명한다.
```

## 만들 것

처음 제품 형태는 CLI다.

명령:

```bash
agent-tool-lint ./tools.json
```

terminal output:

```text
✖ send_email
  Risk: external_side_effect
  Severity: high
  Issues:
  - Description is too vague for an external side-effect tool.
  - Missing explicit user approval boundary.
  - Missing "do not call for drafts or suggestions" boundary.
  - Parameters object should use additionalProperties: false.
  - Missing error contract.
  - Missing idempotency or duplicate-send prevention guidance.

⚠ create_calendar_event
  Risk: write
  Severity: medium
  Issues:
  - Write tool should have a dry-run or preview pair.
  - startAt should document ISO 8601 datetime with timezone.

✓ search_docs
  Risk: read
  Severity: none
```

JSON output:

```bash
agent-tool-lint ./tools.json --format json
```

```json
{
  "summary": {
    "tools": 3,
    "errors": 1,
    "warnings": 2
  },
  "results": [
    {
      "tool": "send_email",
      "risk": "external_side_effect",
      "severity": "high",
      "issues": [
        {
          "rule": "missing-approval-boundary",
          "severity": "high",
          "message": "External side-effect tool must say it should only run after explicit user request and approval."
        }
      ]
    }
  ]
}
```

Markdown report:

```bash
agent-tool-lint ./tools.json --format markdown > TOOL_RISK_REPORT.md
```

```md
# Tool Risk Report

## Summary

- Tools: 3
- High: 1
- Medium: 1
- Low: 0

## High Risk Tools

### send_email

- Risk: external_side_effect
- Missing explicit approval boundary
- Missing recipient/body confirmation
- Missing error contract
- Missing idempotency or duplicate-send prevention guidance

## Recommendations

- Split `draft_email` and `send_email`.
- Add `requiresApprovalBeforeExecution`.
- Add structured error output.
```

## MVP 범위

처음부터 다 지원하지 않는다.

v0.1은 이것만 한다.

```text
입력
→ OpenAI function tool schema JSON
→ Responses-style / Chat Completions-style shape normalize

출력
→ terminal report
→ JSON report
→ Markdown report

검사
→ name rule
→ description rule
→ parameters rule
→ risk classification
→ approval boundary
→ error contract 존재 여부
→ strict mode 준비도 일부 warning

옵션
→ --format text|json|markdown
→ --fail-on low|medium|high|critical
```

v0.1에서 하지 않을 것:

```text
MCP 전체 schema 지원
OpenAPI parsing
VSCode extension
자동 수정
LLM 기반 review
복잡한 config system
웹 UI
```

나중에 필요하면 추가한다.

처음 목표는 “작지만 쓸 수 있는 CLI”다.

## 입력 포맷

처음에는 OpenAI function tool schema JSON만 지원한다.

다만 OpenAI tool schema는 API surface에 따라 모양이 다를 수 있다.

v0.1에서는 최소 두 형태를 normalize한다.

Responses-style:

```json
[
  {
    "type": "function",
    "name": "send_email",
    "description": "Send an email.",
    "parameters": {
      "type": "object",
      "properties": {
        "to": {
          "type": "string"
        },
        "subject": {
          "type": "string"
        },
        "body": {
          "type": "string"
        }
      },
      "required": ["to", "subject", "body"]
    }
  }
]
```

Chat Completions-style:

```json
[
  {
    "type": "function",
    "function": {
      "name": "send_email",
      "description": "Send an email.",
      "parameters": {
        "type": "object",
        "properties": {
          "to": {
            "type": "string"
          },
          "subject": {
            "type": "string"
          },
          "body": {
            "type": "string"
          }
        },
        "required": ["to", "subject", "body"]
      }
    }
  }
]
```

raw input shape는 provider/API surface마다 다를 수 있다.

그래서 rule이 raw input을 직접 보면 안 된다.

항상 adapter를 먼저 둔다.

```text
raw input
→ normalizeToolSchema()
→ internal ToolDefinition
→ lint rules
```

내부 표준 형태:

```ts
type ToolDefinition = {
  name: string;
  description: string;
  parameters: JsonSchemaObject;
  outputSchema?: JsonSchemaObject;
  metadata?: Record<string, unknown>;
};
```

처음부터 모든 provider를 지원하지 않는다.

하지만 내부 형태를 분리해두면 나중에 MCP/OpenAPI/LangChain adapter를 붙이기 쉽다.

중요한 점:

```text
lint rule은 raw JSON shape를 몰라도 된다.
lint rule은 normalizeToolSchema()가 만든 ToolDefinition만 본다.
```

## strict mode compatibility 주의

OpenAI strict mode compatibility를 완전히 검증하는 것은 v0.2로 미룬다.

하지만 v0.1에서도 다음은 warning으로 잡는다.

```text
additionalProperties: false 누락
required 누락
optional field를 nullable union으로 표현하지 않은 schema
```

이 검사는 보안 인증이 아니다.

`strict mode 준비도`를 알려주는 lint다.

즉:

```text
이 schema가 안전하다고 증명한다 → 아님
strict mode에서 문제가 될 수 있는 냄새를 알려준다 → 맞음
```

## Risk classification

린터는 먼저 tool의 위험도를 추정한다.

```text
read
dry_run
write
destructive
external_side_effect
financial
secret_access
admin
unknown
```

여기서 risk와 severity를 섞으면 안 된다.

```text
risk
→ tool의 성격

severity
→ 발견된 issue의 심각도
```

예를 들어 `send_email`은 risk가 `external_side_effect`다.

하지만 description, approval boundary, error contract, idempotency가 잘 되어 있으면 high issue가 없어야 한다.

반대로 `search_docs`는 보통 read tool이다.

하지만 secret source를 읽거나 민감한 raw result를 그대로 노출하면 issue severity가 올라갈 수 있다.

정리하면:

```text
위험한 tool이라고 항상 실패는 아니다.
위험한 tool인데 필요한 boundary가 빠졌을 때 high/critical issue가 된다.
```

이 분류는 완벽할 수 없다.

처음에는 heuristic이면 충분하다.

예:

```text
search_*
get_*
list_*
read_*
→ read

preview_*
validate_*
dry_run_*
→ dry_run

draft_*
→ dry_run candidate 또는 low-risk write candidate

create_*
update_*
write_*
save_*
→ write

delete_*
remove_*
destroy_*
drop_*
revoke_*
→ destructive

send_*
post_*
publish_*
notify_*
reply_*
→ external_side_effect

charge_*
refund_*
pay_*
transfer_*
→ financial

read_secret
get_token
export_credentials
→ secret_access
```

주의: `draft_*`는 항상 dry_run이 아니다.

local preview만 만들면 dry_run에 가깝다.

하지만 Gmail, Slack, Notion 같은 외부 서비스에 draft object를 실제로 생성하면 write action이다.

따라서 v0.1 heuristic은 `draft_*`를 무조건 dry_run으로 확정하지 않는다.

```text
draft_* + local/preview wording
→ dry_run, confidence medium

draft_* + gmail/slack/notion/external wording
→ write, confidence medium
```

classification이 틀릴 수 있으므로 report에는 confidence를 남긴다.

```json
{
  "tool": "send_email",
  "risk": "external_side_effect",
  "confidence": "high",
  "reason": "name starts with send_"
}
```

나중에는 config로 override할 수 있다.

하지만 v0.1에서는 heuristic + clear report면 충분하다.

## Rule 1. name 검사

나쁜 이름:

```text
run
query
process
handle
calendar
email
doThing
```

문제:

```text
무엇을 하는지 모름
read/write/destructive 구분이 안 됨
하나의 tool에 여러 intent가 섞일 가능성이 큼
```

좋은 이름:

```text
search_calendar_events
preview_calendar_event
create_calendar_event
draft_email
send_email
delete_email
```

lint rule:

```text
name-too-generic
```

실패 예:

```json
{
  "name": "calendar",
  "description": "Calendar stuff."
}
```

출력:

```text
✖ calendar
  rule: name-too-generic
  severity: high
  message: Tool name is too broad. Split read/write/destructive actions into separate tools.
```

## Rule 2. description 검사

description은 모델이 보는 usage guide다.

나쁜 description:

```text
Send email.
Calendar stuff.
Gets data.
```

좋은 description:

```text
Send an email only after the user explicitly asks to send it and has approved the recipient, subject, and body. Do not call for drafts, suggestions, summaries, or unapproved messages.
```

lint rules:

```text
description-too-vague
missing-when-not-to-call
missing-approval-boundary
missing-sensitive-action-warning
```

write/destructive/external_side_effect tool에서는 더 엄격하게 본다.

```text
read tool
→ when to call 정도면 충분할 수 있음

write tool
→ explicit user intent 필요

destructive tool
→ explicit confirmation 필요

external_side_effect
→ recipient/content/target approval 필요
```

출력 예:

```text
✖ send_email
  rule: missing-approval-boundary
  severity: high
  message: External side-effect tool must say it only runs after explicit user approval.
```

## Rule 3. input schema 검사

나쁜 schema:

```json
{
  "parameters": {
    "type": "object",
    "properties": {
      "input": { "type": "string" }
    }
  }
}
```

문제:

```text
모든 의도가 input 하나에 섞임
runner가 자연어 parsing을 떠안음
read/write/delete 구분이 흐려짐
검증하기 어려움
```

lint rules:

```text
loose-input-string
missing-additional-properties-false
missing-required-fields
missing-field-description
enum-candidate-is-free-string
datetime-missing-timezone-guidance
dangerous-field-missing-description
```

`query: string`은 항상 나쁜 것이 아니다.

검색 tool에서는 자연스럽다.

```json
{
  "name": "search_docs",
  "parameters": {
    "type": "object",
    "properties": {
      "query": { "type": "string" }
    },
    "required": ["query"],
    "additionalProperties": false
  }
}
```

문제는 write/destructive action을 자연어 `input` 하나로 받는 것이다.

strict mode 준비도와도 연결된다.

v0.1에서는 다음을 warning으로 본다.

```text
object schema인데 additionalProperties: false가 없음
properties가 있는데 required가 비어 있음
optional처럼 보이는 field가 nullable 표현 없이 빠져 있음
```

이건 “안전하지 않다”는 최종 판정이 아니다.

strict schema로 가기 전에 손봐야 할 가능성이 있다는 신호다.

## Rule 4. output contract 검사

tool output은 다음 모델 판단의 observation이다.

나쁜 output:

```text
Success.
```

또는:

```text
raw HTTP response dump
5MB log
stack trace 전체
```

좋은 output:

```json
{
  "ok": true,
  "action": "preview_calendar_event",
  "summary": "Preview created for 병원 예약 on 2026-07-02 15:00 Asia/Seoul.",
  "requiresApprovalBeforeWrite": true,
  "preview": {
    "title": "병원 예약",
    "startAt": "2026-07-02T15:00:00+09:00",
    "endAt": "2026-07-02T16:00:00+09:00"
  }
}
```

v0.1에서는 output schema를 강제하지 않아도 된다.

대신 다음 중 하나를 찾는다.

```text
outputSchema
returns
metadata.outputContract
description 안의 Returns / Output / Error 설명
```

lint rules:

```text
missing-output-contract
missing-ok-field
missing-source-or-timestamp-for-read-tool
output-may-leak-sensitive-data
```

주의할 점이 있다.

OpenAI function tool schema는 기본적으로 input parameters 중심이다.

따라서 `outputSchema`나 `errorContract`가 표준 필드로 항상 존재한다고 가정하면 안 된다.

v0.1에서는 다음처럼 보수적으로 적용한다.

```text
read-only pure query tool
→ missing-output-contract는 info/warning

write/destructive/external_side_effect/financial tool
→ missing-output-contract는 medium 이상 가능

민감정보, source, timestamp가 중요한 read tool
→ output contract와 source/timestamp를 더 강하게 본다.
```

## Rule 5. error contract 검사

나쁜 error:

```text
Failed.
```

좋은 error:

```json
{
  "ok": false,
  "errorCode": "AMBIGUOUS_DATETIME",
  "message": "Start time is missing a timezone.",
  "retryable": true,
  "userFixable": true,
  "suggestedNextStep": "Ask the user for timezone."
}
```

lint rules:

```text
missing-error-contract
missing-error-code
missing-retryable-field
missing-user-fixable-field
missing-suggested-next-step
raw-stack-trace-risk
```

v0.1에서는 schema를 완벽히 해석하지 않아도 된다.

우선 error contract 존재 여부를 본다.

```text
description에 error behavior가 있는가?
metadata.errorContract가 있는가?
output schema에 ok:false path가 있는가?
```

error contract도 risk-aware severity로 적용한다.

```text
read-only pure query tool
→ missing-error-contract는 warning 정도

write/destructive/external_side_effect tool
→ missing-error-contract는 high 가능

financial/destructive tool
→ error contract와 retry/idempotency 설명 누락은 critical 가능
```

이렇게 해야 false positive가 줄어든다.

린터가 너무 시끄러우면 개발자는 끈다.

## Rule 6. approval boundary 검사

위험한 tool은 approval boundary가 있어야 한다.

대상:

```text
write
destructive
external_side_effect
financial
secret_access
admin
```

검사:

```text
description에 explicit user request/approval 조건이 있는가?
destructive tool에 confirmation 조건이 있는가?
external side effect에 target/content 확인이 있는가?
financial tool에 idempotency와 confirmation이 있는가?
metadata에 approvalRequired가 있는가?
```

출력 예:

```text
✖ delete_calendar_event
  rule: destructive-tool-needs-confirmation
  severity: critical
  message: Destructive tool must require explicit confirmation before execution.
```

## Rule 7. idempotency / duplicate prevention 검사

쓰기 tool은 같은 요청이 두 번 실행될 수 있다.

```text
모델이 같은 tool call을 반복함
harness가 network error 후 retry함
사용자가 같은 요청을 다시 보냄
```

따라서 이런 tool은 idempotency가 중요하다.

```text
send_email
create_calendar_event
charge_credit_card
create_invoice
post_slack_message
```

lint rules:

```text
missing-idempotency
missing-duplicate-prevention
write-tool-needs-dry-run-or-preview
```

v0.1에서는 다음을 찾는다.

```text
idempotencyKey field
duplicate detection note
dry-run / preview pair
confirmation step
```

## 구현 단계

실제 구현할 때는 이 순서로 간다.

```text
1. package scaffold
2. CLI argument parsing
3. JSON file load
4. normalizeToolSchema()
5. risk classification
6. rule runner
7. text/json/markdown reporter
8. --fail-on option
9. fixtures
10. regression tests
```

처음에는 dependency를 많이 넣지 않는다.

가능하면 Node.js 표준 라이브러리와 작은 CLI parser 정도로 충분하다.

Ponytail 원칙:

```text
새 dependency는 정말 필요할 때만.
rule은 단순 function으로.
class hierarchy 만들지 말기.
config system은 v0.2로 미루기.
```

## Fixture 설계

테스트는 fixture 중심으로 만든다.

```text
fixtures/
  good/
    search-docs.json
    preview-calendar-event.json
    search-docs-query-string.json
    send-email-with-approval-and-error-contract.json
    create-calendar-with-preview-and-idempotency.json
  bad/
    vague-send-email.json
    loose-calendar-input.json
    destructive-delete-no-confirmation.json
    charge-card-no-idempotency.json
  edge/
    draft-email-local-preview.json
    draft-email-external-service-write.json
    query-string-write-tool-bad.json
```

각 fixture는 예상 issue를 가진다.

예:

```json
{
  "fixture": "bad/vague-send-email.json",
  "expectedIssues": [
    "description-too-vague",
    "missing-approval-boundary",
    "missing-error-contract",
    "missing-idempotency"
  ]
}
```

이것이 11장의 golden task 역할을 한다.

bad를 잘 잡는 것만큼 중요한 것이 있다.

좋은 schema를 괜히 때리지 않는 것이다.

예:

```text
approval boundary가 명확하고,
recipient/content confirmation이 있고,
error contract가 있고,
idempotency guidance가 있는 send_email
→ high severity issue가 없어야 한다.
```

이런 fixture가 있어야 린터가 “겁주는 장난감”이 아니라 실제 개발 도구가 된다.

## 평가 기준

완료는 “CLI가 실행된다”가 아니다.

다음 기준을 만족해야 한다.

```text
1. bad send_email fixture를 high severity로 잡는다.
2. good search_docs fixture는 통과한다.
3. delete_* tool에 confirmation boundary가 없으면 critical로 잡는다.
4. charge_* tool에 idempotency 언급이 없으면 high 이상으로 잡는다.
5. loose input string을 write/destructive tool에서 잡는다.
6. --format json이 machine-readable report를 출력한다.
7. --format markdown이 GitHub README/PR에 붙일 수 있는 report를 출력한다.
8. --fail-on high가 high 이상 issue에서 non-zero exit code를 반환한다.
9. fixture 기반 regression test가 있다.
10. good send_email fixture는 high severity issue를 내지 않는다.
11. draft_email local preview와 external service write를 다르게 분류한다.
12. README에 limitation이 명시되어 있다.
```

## 완료 체크리스트

```text
- [ ] CLI command가 실행된다.
- [ ] JSON schema 파일을 읽는다.
- [ ] 여러 tool array를 처리한다.
- [ ] risk classification을 출력한다.
- [ ] 최소 8개 lint rule이 있다.
- [ ] text report가 읽을 만하다.
- [ ] json report가 안정적인 shape를 가진다.
- [ ] markdown report를 생성한다.
- [ ] --fail-on 옵션이 있다.
- [ ] good/bad/edge fixture가 있다.
- [ ] regression test가 있다.
- [ ] README에 사용 예시가 있다.
- [ ] README에 false positive / false negative 한계가 있다.
```

## README 작성법

README 첫 문장은 짧게 쓴다.

영어:

```text
Lint LLM agent tool schemas for unsafe names, vague descriptions, weak parameters, missing approval boundaries, and risky side effects.
```

한국어:

```text
LLM agent의 tool/function schema를 검사해 모호한 이름, 느슨한 입력, 누락된 승인 경계, 위험한 side effect를 찾아주는 CLI.
```

README 구조:

```text
1. What is this?
2. Why tool schemas matter
3. Install
4. Usage
5. Example input
6. Example output
7. Rules
8. CI usage
9. Limitations
10. Roadmap
```

중요한 것은 limitation이다.

이 도구는 보안 인증 도구가 아니다.

security scanner도 아니다.

정확한 포지션은 schema linter다.

```text
정적 schema lint 도구다.
runner validation을 대체하지 않는다.
approval policy를 대체하지 않는다.
실제 tool implementation을 분석하지 않는다.
heuristic risk classification은 틀릴 수 있다.
false positive와 false negative가 있다.
```

이 limitation을 써야 신뢰가 생긴다.

## 이력서 / 포트폴리오 기록법

나쁜 기록:

```text
AI agent tool 만들었습니다.
```

좋은 기록:

```text
Built agent-tool-lint, a CLI that statically analyzes LLM agent tool schemas for unsafe names, vague descriptions, weak JSON schemas, missing approval boundaries, and risky side effects. Added fixture-based regression tests and CI-ready severity reporting.
```

한국어:

```text
LLM agent의 tool/function schema를 정적으로 분석하는 CLI를 구현했습니다. 모호한 tool name, 느슨한 JSON schema, 누락된 approval boundary, error contract, idempotency 위험을 탐지하고 fixture 기반 regression test와 CI용 severity report를 제공했습니다.
```

포트폴리오 글 구조:

```text
Problem
→ agent tool schema가 모델의 API 문서인데 너무 쉽게 허술해진다.

Why it matters
→ 허술한 schema는 잘못된 tool call, approval bypass, 외부 side effect 사고로 이어질 수 있다.

Design
→ risk classification + rule-based lint + severity + report.

Safety
→ write/destructive/external_side_effect rule을 엄격하게 둔다.

Evaluation
→ good/bad fixtures와 regression test로 rule을 검증한다.

Result
→ N개 fixture 통과, high severity issue 감지, markdown/json report 생성.

What I learned
→ agent는 prompt보다 runtime boundary와 observation 설계가 중요하다.
```

## 확장 과제

v0.2:

```text
idempotency rule 강화
dry-run/preview pair 자동 탐지
strict mode compatibility check
severity config
CI GitHub Action 예시
```

v0.3:

```text
MCP tool schema 일부 지원
OpenAPI에서 tool schema 추출
rule config 파일
ignore rule comment
SARIF output
GitHub Code Scanning 연동
```

v0.4:

```text
VSCode extension
웹 report viewer
LLM-assisted rule explanation
tool runner metadata lint
```

확장할 때도 원칙은 같다.

```text
작게 유지한다.
정적 lint 도구임을 잊지 않는다.
runner/sandbox/approval을 대체한다고 말하지 않는다.
```

## 마지막 기준

이 과제를 끝냈다고 해서 “완성형 agent 개발자”가 되는 것은 아니다.

하지만 중요한 선을 넘는다.

```text
에이전트를 말로 설명하는 사람
→ 에이전트 개발 도구를 만든 사람
```

작은 차이 같지만 크다.

이 도구는 agent가 외부 세계를 만지는 손잡이를 검사한다.

그 손잡이를 안전하게 보는 눈이 생기면, 다음 과제인 micro coding agent도 훨씬 덜 위험하게 설계할 수 있다.
