[← README](../README.md)

# 9장. Tool design: 모델에게 함수를 맡기는 법

## 이 장에서 배울 것

2장에서 우리는 이렇게 말했다.

```text
모델은 판단하고,
하네스는 실행한다.
```

9장은 이 문장을 tool 설계로 옮긴다.

LLM agent에서 tool은 “모델이 직접 실행하는 함수”처럼 보인다. 하지만 실제로는 조금 다르다.

이 장의 중심 문장은 이것이다.

```text
Tool schema는 모델에게 주는 API 문서이고,
tool runner는 실제 API 서버다.
```

모델은 tool schema를 읽고 “이 tool을 이런 인자로 호출하고 싶다”는 구조화된 요청을 만든다. 하지만 파일을 읽고, DB를 조회하고, HTTP 요청을 보내고, 로컬 command를 실행하는 것은 모델이 아니다.

그 실행은 하네스가 한다.

```text
model
→ tool name과 arguments 생성

harness
→ 권한 확인
→ 실제 함수 실행
→ result 수집
→ observation/context로 모델에게 다시 전달
```

그래서 tool design은 단순히 함수 하나 만드는 일이 아니다.

모델에게 어떤 API 문서를 보여줄지, 어떤 입력을 허용할지, 어떤 결과를 돌려줄지, 언제 호출하면 안 되는지 설계하는 일이다.

## 먼저 직관으로 이해하기

웹 개발자에게 tool schema는 Swagger/OpenAPI 문서와 비슷하다.

예를 들어 서버에 이런 API가 있다고 하자.

```http
GET /users/:id/orders
```

이 endpoint를 사람 개발자에게 설명하려면 문서가 필요하다.

```text
name:
  getUserOrders

description:
  특정 사용자의 최근 주문 목록을 가져온다.

parameters:
  userId: string, required
  limit: number, optional, default 20

returns:
  order id, status, total, createdAt

주의:
  결제 상세 정보나 카드 정보는 반환하지 않는다.
```

문서가 나쁘면 사람도 API를 잘못 쓴다.

```text
description이 모호함
→ 언제 써야 하는지 모름

parameter 이름이 애매함
→ userId인지 email인지 헷갈림

return shape이 불명확함
→ downstream 코드가 깨짐

권한/주의사항이 없음
→ 위험한 요청을 보냄
```

모델도 비슷하다.

Tool schema는 모델에게 보이는 API 문서다.

```text
모델은 함수 구현을 읽는 것이 아니라,
대개 name, description, input schema, tool instructions를 보고 판단한다.
```

따라서 tool이 잘못 호출된다면, “모델이 멍청하다” 전에 schema를 봐야 한다.

```text
이름이 너무 넓은가?
description이 모호한가?
input schema가 느슨한가?
언제 쓰지 말아야 하는지 안 적었는가?
output이 너무 크거나 불안정한가?
```

Tool design은 API design이다. 다만 클라이언트가 사람 개발자가 아니라 모델이라는 점이 다르다.

## Tool calling의 기본 흐름

OpenAI의 function calling 문서도 tool을 “모델이 외부 시스템의 데이터나 기능을 사용할 수 있게 하는 방법”으로 설명한다. function tool은 JSON Schema로 정의되고, 모델은 그 schema에 맞는 arguments를 생성한다.

중요한 점은 이것이다.

```text
모델은 tool call을 제안한다.
실행은 application 또는 harness가 한다.
```

흐름을 단순화하면 이렇다.

```text
1. 개발자가 tool 정의를 모델 호출에 포함한다.
2. 모델이 user prompt와 tool schema를 보고 tool call을 생성한다.
3. 하네스가 tool name과 arguments를 검증한다.
4. 하네스가 실제 함수를 실행한다.
5. 하네스가 tool result를 모델 context에 다시 넣는다.
6. 모델이 result를 보고 다음 답변 또는 다음 tool call을 만든다.
```

실제 agent runtime에서는 한 turn에 tool call이 0개, 1개, 또는 여러 개 나올 수 있다.

```text
0개
→ tool 없이 바로 답변

1개
→ 단일 tool 실행 후 result 재주입

여러 개
→ 독립적이면 병렬 실행 가능
→ 순서 의존이 있으면 직렬 실행 필요
→ 일부 실패하면 partial result를 observation으로 설계해야 함
```

그래서 harness는 “모델은 항상 tool을 하나만 호출한다”고 가정하면 안 된다. 예를 들어 `search_docs`와 `get_current_weather`는 병렬로 실행해도 되지만, `search_customer` 결과의 `customerId`로 `get_customer_orders`를 호출해야 한다면 순서가 필요하다.

예를 들어:

```text
사용자:
이번 주 서울 날씨 알려줘.

모델:
get_weather({ location: "Seoul", days: 7 }) 호출 필요

하네스:
weather API 호출

tool result:
[{ day: "Mon", temp: ... }, ...]

모델:
이번 주 서울은 ...
```

여기서 weather API를 호출한 것은 모델이 아니다. 모델은 call intent와 arguments를 만들었고, 하네스가 실행했다.

이 분리를 놓치면 tool design이 위험해진다.

```text
모델이 실행한다고 생각함
→ 권한/검증/에러 처리 설계를 빼먹음

하네스가 실행한다고 생각함
→ tool runner, approval, sandbox, output contract를 설계함
```

## Tool schema는 무엇을 담아야 하는가

좋은 tool schema에는 최소한 다음이 있다.

```text
name
description
input schema
output contract
when to call
when not to call
side effect / permission note
error behavior
```

공식 API의 function tool schema는 주로 name, description, parameters(JSON Schema)로 표현된다. 하지만 하네스 설계 관점에서는 description 안에 call policy와 output expectation까지 함께 담는다고 생각하는 편이 좋다.

### Name: 모델이 고르는 메뉴 이름

tool name은 모델이 tool을 고를 때 보는 메뉴 이름이다.

나쁜 이름:

```text
handle
process
doThing
query
run
```

좋은 이름:

```text
get_customer_orders
search_internal_docs
create_calendar_event
read_project_file
send_slack_message
```

name은 짧지만 구체적이어야 한다.

```text
무엇을 하는가?
대상이 무엇인가?
읽기인가 쓰기인가?
```

특히 read/write 구분은 중요하다.

```text
get_invoice
→ 읽기

create_invoice
→ 쓰기

delete_invoice
→ 위험한 쓰기
```

모델에게도 이 차이가 보여야 한다.

### Description: 모델을 위한 usage guide

description은 사람이 보는 주석이 아니라 모델이 보는 usage guide다.

나쁜 description:

```text
Gets data.
```

조금 나은 description:

```text
Get customer data.
```

좋은 description:

```text
Use this to retrieve non-sensitive profile fields for one customer by customerId.
Do not use for payment details, authentication secrets, or bulk exports.
```

여기에는 네 가지가 들어 있다.

```text
언제 쓰는가
무엇을 입력으로 받는가
무엇을 돌려주는가
언제 쓰면 안 되는가
```

모델은 description을 보고 도구 선택을 한다. description이 느슨하면 tool 선택도 느슨해진다.

### Input schema: 모델이 채워야 하는 form

input schema는 모델이 채워야 하는 form이다.

예:

```json
{
  "type": "object",
  "properties": {
    "customerId": {
      "type": "string",
      "description": "Internal customer id, not email."
    },
    "limit": {
      "type": "integer",
      "minimum": 1,
      "maximum": 50,
      "default": 20
    }
  },
  "required": ["customerId"],
  "additionalProperties": false
}
```

좋은 schema는 모델이 실수할 여지를 줄인다.

```text
required
→ 반드시 필요한 값 표시

enum
→ 허용된 값 제한

minimum / maximum
→ 숫자 범위 제한

description
→ 필드 의미 설명

additionalProperties: false
→ 엉뚱한 필드 방지
```

가능하면 느슨한 string 하나로 모든 것을 받지 말아야 한다.

나쁜 예:

```json
{
  "query": "string"
}
```

이건 편하지만 위험하다.

모델이 SQL, 자연어, ID, filter, pagination을 모두 `query`에 섞어 넣을 수 있다.

물론 `query: string`이 항상 나쁜 것은 아니다.

검색 tool이라면 자연어 query 하나가 오히려 가장 좋은 interface일 수 있다.

```text
search_docs({ query: "AGENTS.md load order" })
search_web({ query: "OpenAI function calling strict mode" })
```

문제는 검색이 아닌 action tool까지 느슨한 자연어 하나로 처리하는 것이다.

```text
send_email({ input: "민수한테 어제 얘기한 그거 보내줘" })
charge_credit_card({ input: "이번 달 요금 처리" })
delete_database_row({ input: "쓸모없는 것 지워" })
```

쓰기, 결제, 삭제, 외부 전송은 더 좁고 명시적인 schema가 필요하다.

좋은 예:

```json
{
  "customerId": "cus_123",
  "status": "paid",
  "limit": 20
}
```

schema가 구체적일수록 runner도 검증하기 쉽다.

여기서 production schema를 만들 때는 API의 strict mode 요구사항도 확인해야 한다.

예를 들어 OpenAI function calling에서 `strict: true`를 쓰면 schema 제약이 더 강해진다. 보통 object schema에는 `additionalProperties: false`가 필요하고, properties에 정의된 필드는 모두 `required`에 포함되어야 한다. 선택값은 API가 지원하는 방식에 맞춰 nullable union으로 표현하거나, runner 쪽 default로 처리하는 식의 설계가 필요하다.

즉 이 장의 예시는 개념 설명용이다. 실제 tool schema는 사용하는 API/runtime의 strict schema 규칙에 맞게 다듬어야 한다.

### Output contract: tool result도 API다

tool input만 설계하면 반쪽이다.

tool output도 설계해야 한다.

나쁜 output:

```text
raw HTTP response
거대한 JSON dump
불규칙한 문자열
에러 stack trace 전체
```

좋은 output:

```json
{
  "orders": [
    {
      "id": "ord_123",
      "status": "paid",
      "total": 42000,
      "currency": "KRW",
      "createdAt": "2026-07-01"
    }
  ],
  "hasMore": false
}
```

또는 모델이 읽기 쉽게 요약된 형태:

```text
Found 3 paid orders for customer cus_123.
- ord_123: 42,000 KRW, 2026-07-01
- ord_122: 18,000 KRW, 2026-06-28
No failed or refunded orders in result.
```

어느 쪽이 좋은지는 downstream 목적에 따라 다르다.

```text
다음 tool이 기계적으로 처리해야 함
→ structured JSON

모델이 사용자에게 설명해야 함
→ compact summary + key fields

감사/재현이 중요함
→ source id, query, timestamp 포함
```

중요한 것은 tool output도 context budget을 먹는다는 점이다.

거대한 output은 memory noise와 같다.

```text
tool result가 너무 큼
→ context window 압박
→ 중요한 instruction/decision이 밀림
→ 모델이 엉뚱한 부분에 attention 씀
```

좋은 tool은 필요한 만큼만 돌려준다.

### Error output: 실패도 observation이다

tool 실패도 모델에게는 다음 판단 재료다.

나쁜 error output:

```text
Error
```

또는:

```text
stack trace 300줄
```

이러면 모델은 무엇을 해야 할지 모른다.

좋은 error output은 실패를 작고 구조적으로 알려준다.

```json
{
  "ok": false,
  "errorCode": "AMBIGUOUS_DATETIME",
  "message": "Start time is missing a timezone.",
  "retryable": true,
  "userFixable": true,
  "partialResult": null,
  "suggestedNextStep": "Ask the user for timezone or use the user's configured timezone."
}
```

여기서 중요한 필드는 이런 것이다.

```text
errorCode
→ runner와 모델이 분기할 수 있는 안정적 코드

retryable
→ 같은 call을 다시 시도할 가치가 있는지

userFixable
→ 사용자에게 물어보면 해결되는 문제인지

partialResult
→ 일부 성공한 결과가 있는지

suggestedNextStep
→ 모델이 다음 turn에서 취할 복구 방향
```

실패를 잘 설계하면 모델이 “망했습니다”에서 멈추지 않는다.

```text
시간대가 없음
→ 사용자에게 timezone 질문

rate limit
→ 잠시 후 retry 또는 더 작은 요청

권한 없음
→ 권한 필요하다고 설명

일부만 성공
→ 성공/실패를 나눠서 보고
```

성공 output만 API가 아니다. 실패 output도 API다.

## When to call / when not to call

tool description에는 “언제 호출할지”뿐 아니라 “언제 호출하지 말아야 할지”가 필요하다.

예를 들어 `send_slack_message` tool이 있다고 하자.

나쁜 description:

```text
Send a Slack message.
```

좋은 description:

```text
Send a Slack message to a channel or user after the user explicitly asks to send, post, notify, or reply.
Do not call for drafts, suggestions, summaries, or messages the user has not approved.
```

이 차이는 크다.

모델은 사용자의 “이렇게 보내면 어때?”라는 말을 실제 전송 요청으로 오해할 수 있다.

그래서 write tool에는 call boundary가 필요하다.

```text
read tool
→ 비교적 자동 호출 가능

write tool
→ 명시적 사용자 의도 필요

destructive tool
→ confirmation, approval, sandbox, policy 필요

external side-effect tool
→ 누구에게 무엇이 전송되는지 명확해야 함
```

Tool design에서 가장 위험한 실수는 모든 tool을 같은 위험도로 보는 것이다.

```text
search_docs
send_email
delete_database_row
charge_credit_card
```

이 네 가지는 모두 “tool”이지만 같은 tool이 아니다.

쓰기 tool에는 dry-run 또는 draft 단계를 따로 두는 편이 안전하다.

```text
draft_email
→ 보낼 내용을 만들지만 실제 전송하지 않음

send_email
→ 사용자 승인 후 실제 전송

preview_calendar_event
→ 일정 생성 전 제목/시간/참석자 확인

create_calendar_event
→ 실제 캘린더에 생성
```

모델이 초안을 만드는 것과 외부 세계에 영향을 주는 것은 다르다. 이 차이를 tool 이름과 approval policy에 드러내야 한다.

## Tool runner: 실제 보안 경계

Tool schema는 모델을 유도한다.

하지만 보안 경계는 runner에 있어야 한다.

```text
schema
→ 모델이 올바른 call을 만들도록 도움

runner validation
→ 실제 실행 전 검증

permission/approval
→ 위험한 실행 전 사용자 또는 정책 확인

sandbox
→ 실행 가능한 범위 제한

audit log
→ 무엇이 실행되었는지 기록
```

모델이 schema를 따라줄 것이라고 믿고 runner 검증을 생략하면 안 된다.

웹 API와 같다.

프론트엔드 form validation이 있어도 서버 validation은 필요하다.

```text
client validation
→ UX와 실수 방지

server validation
→ 실제 보안/무결성 경계
```

Tool schema는 client validation에 가깝다.

Tool runner validation은 server validation이다.

쓰기 tool은 idempotency도 생각해야 한다.

모델이 같은 tool call을 반복할 수도 있고, harness가 network error 후 retry할 수도 있다. 이때 같은 이메일이 두 번 가거나, 일정이 두 번 생기거나, 카드 결제가 두 번 일어나면 사고다.

```text
send_email
→ idempotency key, duplicate detection, draft/approval

create_calendar_event
→ 같은 title/startAt/endAt/attendees 중복 탐지

charge_credit_card
→ payment provider idempotency key 필수
```

특히 외부 side effect가 있는 tool은 “한 번만 실행됨”을 모델에게 기대하면 안 된다. 한 번만 실행되게 만드는 것은 runner의 책임이다.

## Tool result를 다시 context에 넣는 법

tool call은 한 번으로 끝나지 않는다.

대부분 agent loop에서는 tool result가 다시 모델에게 들어간다.

```text
model
→ tool call 생성

harness
→ tool 실행

tool result
→ observation으로 context에 추가

model
→ result를 보고 답하거나 다음 tool call 생성
```

여기서 observation 설계가 중요하다.

나쁜 observation:

```text
Command succeeded.
```

모델이 무엇이 성공했는지 모른다.

또 다른 나쁜 observation:

```text
전체 5MB 로그 dump
```

모델이 중요한 신호를 찾기 어렵다.

좋은 observation:

```text
Command: npm test
Result: failed
Exit code: 1
Key error:
  auth.test.ts:42 expected 401, received 200
Relevant files:
  src/auth/middleware.ts
  tests/auth.test.ts
```

tool result는 모델의 다음 판단 재료다.

따라서 “정확하고 작게”가 좋다.

## Codex에서는 어떻게 보이는가

Codex에서 tool은 여러 표면으로 나타난다.

```text
내장 tool
→ 파일 읽기, 파일 수정, shell command, apply_patch 등

MCP tool
→ 외부 MCP server가 제공하는 tool

plugin-bundled MCP
→ plugin manifest가 제공하는 MCP server/tool

connector/app tool
→ Slack, Gmail, GitHub, Google Drive 같은 외부 시스템 연결

Codex MCP server
→ 다른 agent가 Codex를 tool처럼 호출
```

Codex manual 기준으로 MCP는 models를 tools와 context provider에 연결하는 표준 방법이다. Codex는 MCP server의 `instructions`를 읽고, server-wide guidance를 tools와 함께 사용한다. MCP server는 tool list를 제공하고, Codex는 tool policy, approval mode, enabled/disabled tools 같은 설정으로 사용 범위를 조절할 수 있다.

다만 Codex의 MCP 관련 표면은 CLI, plugin, app-server, connector 환경에 따라 제공 방식과 명칭이 달라질 수 있다. 여기서는 MCP를 “외부 tool provider를 연결하는 표준 경로”라는 개념으로 다룬다.

이것도 지금까지 배운 구조와 같다.

```text
MCP server
→ tool provider

tool schema
→ 모델이 볼 API 문서

Codex/harness
→ tool call 실행, permission, result 재주입

config.toml / plugin manifest
→ tool discovery와 policy
```

즉 MCP는 “모델에게 외부 세계를 붙이는 표준 연결 방식”이다.

하지만 연결되었다고 아무 tool이나 막 써야 하는 것은 아니다.

```text
enabled_tools
disabled_tools
approval_mode
tool_timeout
server instructions
```

이런 설정이 tool design의 일부다.

## OpenAI API에서는 어떻게 보이는가

OpenAI API의 function calling에서도 원리는 같다.

function tool은 JSON Schema로 정의된다. 모델은 schema를 보고 arguments를 생성한다. application은 그 arguments로 실제 함수를 실행하고, 결과를 다시 모델에게 전달한다.

OpenAI의 Tools 문서는 built-in tools, function calling, tool search, remote MCP servers 같은 방식으로 모델 capabilities를 확장할 수 있다고 설명한다. 중요한 것은 모두 같은 원리를 공유한다는 점이다.

```text
모델에게 capability 설명
→ 모델이 call intent와 arguments 생성
→ runtime이 실행
→ result가 다시 context로 들어감
```

Structured Outputs와 function calling도 구분해야 한다.

```text
Structured Outputs
→ 모델의 최종 답변 형식을 schema로 제한하고 싶을 때

Function calling
→ 모델이 외부 데이터나 action을 사용해야 할 때
```

둘 다 JSON Schema를 쓸 수 있지만 목적이 다르다.

```text
응답 모양을 통제하고 싶은가?
→ structured output

외부 시스템을 호출해야 하는가?
→ tool/function calling
```

이 구분을 못 하면 모든 문제를 tool로 풀려고 하거나, 반대로 외부 action이 필요한 문제를 structured JSON 출력으로만 처리하려고 한다.

## 작은 예시: 일정 등록 tool

사용자가 예전에 만든 Telegram 일정 등록 agent를 떠올려보자.

나쁜 tool:

```json
{
  "name": "calendar",
  "description": "Calendar stuff",
  "parameters": {
    "type": "object",
    "properties": {
      "input": { "type": "string" }
    }
  }
}
```

이 tool은 너무 넓다.

모델은 `input`에 무엇을 넣어야 할지 애매하다.

읽기인지 쓰기인지도 불명확하다.

좋은 tool은 나뉜다.

```text
search_calendar_events
create_calendar_event
update_calendar_event
delete_calendar_event
```

예를 들어 `create_calendar_event`는 이렇게 설계할 수 있다.

```json
{
  "name": "create_calendar_event",
  "description": "Create a calendar event only after the user clearly asks to schedule, add, or register an event. Do not call for drafts or when date/time is ambiguous.",
  "parameters": {
    "type": "object",
    "properties": {
      "title": {
        "type": "string",
        "description": "Short event title visible on the calendar."
      },
      "startAt": {
        "type": "string",
        "description": "ISO 8601 datetime with timezone, e.g. 2026-07-01T19:00:00+09:00."
      },
      "endAt": {
        "type": "string",
        "description": "ISO 8601 datetime with timezone. Must be after startAt."
      },
      "timezone": {
        "type": "string",
        "description": "IANA timezone such as Asia/Seoul."
      },
      "attendees": {
        "type": "array",
        "items": { "type": "string" },
        "description": "Email addresses explicitly provided by the user."
      }
    },
    "required": ["title", "startAt", "endAt", "timezone"],
    "additionalProperties": false
  }
}
```

여기서 중요한 점:

```text
create_calendar_event
→ write action임이 name에 드러남

only after user clearly asks
→ call boundary가 description에 있음

date/time ambiguous면 호출 금지
→ 모델이 물어봐야 할 상황을 알려줌

ISO 8601 + timezone
→ 시간 해석 오류 줄임

attendees explicitly provided
→ 모델이 임의로 참석자 추가하지 않게 함
```

하지만 schema만으로 충분하지 않다.

runner에서도 검증해야 한다.

```ts
function runCreateCalendarEvent(args, user) {
  assertUserCanWriteCalendar(user);
  assertValidIsoDate(args.startAt);
  assertValidIsoDate(args.endAt);
  assert(args.endAt > args.startAt);
  assert(args.timezone === user.timezone || user.allowedTimezones.includes(args.timezone));
  assertNoUnknownAttendees(args.attendees);
  assertNotDuplicateEvent(args, user);

  return calendarApi.createEvent(args, {
    idempotencyKey: buildIdempotencyKey(user.id, args)
  });
}
```

Tool schema는 모델을 도와주고, runner validation은 시스템을 지킨다.

둘 다 필요하다.

실제 write tool이라면 `idempotencyKey`나 duplicate detection도 고려해야 한다. 사용자가 “일정 등록해줘”라고 했을 때 agent loop, retry, 네트워크 오류 때문에 같은 일정이 두 번 생기면 안 된다.

## Tool을 쪼갤까, 합칠까?

Tool design에서 흔한 고민이 있다.

```text
큰 tool 하나?
작은 tool 여러 개?
```

정답은 “모델이 안전하게 고를 수 있는 단위”다.

너무 큰 tool:

```text
manage_calendar(input: string)
```

문제:

```text
읽기/쓰기/삭제가 섞임
권한 판단 어려움
input parsing이 runner로 밀림
모델이 의도를 잘못 넣을 수 있음
```

너무 작은 tool:

```text
set_event_title
set_event_start
set_event_end
set_event_timezone
commit_event
```

문제:

```text
tool call 수 증가
중간 상태 관리 필요
실패 지점 증가
모델이 orchestration을 과하게 해야 함
```

좋은 단위:

```text
사용자 의도와 side effect boundary가 맞는 단위
```

예:

```text
search_calendar_events
create_calendar_event
update_calendar_event
delete_calendar_event
```

읽기와 쓰기, 삭제를 분리한다.

권한과 approval도 tool 단위로 걸기 쉽다.

다만 항상 함께 호출되는 연속 단계라면 tool을 지나치게 쪼개지 말고 runner 내부 로직으로 합치는 편이 낫다. 반대로 읽기, 쓰기, 삭제처럼 위험도나 승인 정책이 다르면 별도 tool로 분리해야 한다.

## 흔한 오해

### 오해 1. “Tool schema만 있으면 모델이 알아서 잘 쓴다”

아니다.

schema는 모델에게 주는 문서다. 문서가 좋아도 서버 검증은 필요하다.

```text
schema
→ call을 잘 만들게 유도

runner
→ 실제 안전과 무결성 보장
```

### 오해 2. “Tool description은 대충 써도 된다”

아니다.

description은 모델의 tool selection에 직접 영향을 준다.

특히 `when not to call`이 없으면 모델은 필요 이상으로 tool을 호출할 수 있다.

### 오해 3. “Tool output은 자세할수록 좋다”

아니다.

tool output도 context budget을 쓴다.

필요한 핵심 결과, source id, error summary를 작게 돌려주는 편이 낫다.

### 오해 4. “Structured output과 tool calling은 같다”

아니다.

둘 다 schema를 쓰지만 목적이 다르다.

```text
structured output
→ 답변 형식 제어

tool calling
→ 외부 시스템 호출
```

### 오해 5. “Tool이 연결되면 권한 문제는 끝난다”

아니다.

tool이 연결되면 권한 문제가 시작된다.

읽기, 쓰기, 삭제, 외부 전송, 결제, 개인정보 접근은 모두 다른 정책이 필요하다.

## 왜 중요한가

Tool design은 agent의 손과 발을 설계하는 일이다.

모델은 말과 판단을 잘한다.

하지만 실제 세계를 바꾸는 것은 tool이다.

```text
파일 수정
메시지 전송
일정 등록
결제 실행
DB row 삭제
브라우저 조작
GitHub PR 작성
```

이런 작업은 모두 위험도와 실패 비용이 다르다.

좋은 tool design은 모델에게 능력을 주면서도 사고를 줄인다.

```text
명확한 name
구체적 description
좁은 input schema
예측 가능한 output contract
when not to call
runner validation
permission / approval
small observation
audit trail
```

Tool을 잘 설계하면 agent는 똑똑해진다.

Tool을 대충 설계하면 agent는 자신감 있게 위험해진다.

## 요약

```text
1. Tool schema는 모델에게 주는 API 문서다.
2. 모델은 tool name과 arguments를 생성하고, 하네스가 실제 실행한다.
3. 좋은 tool은 name, description, input schema, output contract, when not to call을 가진다.
4. Tool schema는 runner validation과 permission을 대체하지 않는다.
5. Tool output과 error output은 다음 모델 판단을 위한 observation이므로 작고 복구 가능해야 한다.
6. 읽기, 쓰기, 삭제, 외부 전송 tool은 위험도가 다르다.
7. write/destructive/external side-effect tool은 approval, audit, idempotency를 고려해야 한다.
8. Structured output은 답변 형식 제어이고, tool calling은 외부 시스템 호출이다.
9. Tool design은 agent의 capability boundary를 설계하는 일이다.
```

## 생각해볼 질문

1. 내가 만들 tool은 읽기 tool인가, 쓰기 tool인가, destructive tool인가?
2. tool name만 보고 모델이 의도를 정확히 이해할 수 있는가?
3. description에 when to call과 when not to call이 모두 있는가?
4. input schema가 너무 느슨하지 않은가?
5. runner는 모델 arguments를 다시 검증하는가?
6. tool result는 다음 모델 판단에 필요한 만큼만 작게 돌아오는가?
7. 실패 output은 모델이 복구할 수 있을 만큼 구조적인가?
8. 이 tool은 사용자 승인, sandbox, audit log, idempotency가 필요한가?
