[← README](../README.md)

# 2장. 에이전트는 왜 모델 하나가 아닐까?

## 이 장에서 배울 것

1장에서 우리는 모델을 이렇게 봤다.

```text
LLM = context를 보고 다음 답변 또는 tool call을 만드는 모델
```

이제 한 단계 더 간다.

많은 사람이 처음에는 “에이전트 = 더 똑똑한 모델”이라고 생각한다. 하지만 하네스 엔지니어링 관점에서는 다르게 봐야 한다.

```text
Agent = LLM + runtime
```

여기서 agent는 철학적·학술적으로 엄밀한 정의가 아니다. 업계에서도 어떤 사람은 tool을 쓰는 chatbot을 agent라고 부르고, 어떤 사람은 장기 목표, 계획, 자율 루프, 환경 조작까지 있어야 agent라고 부른다.

이 책에서는 하네스 엔지니어링 관점의 실용적 정의를 쓴다.

```text
Agent
= LLM이 다음 행동 후보를 만들고,
  runtime이 context 조립, tool 실행, 권한 확인,
  상태 관리, 결과 재주입을 담당하는 시스템
```

이 장의 중심 문장은 이것이다.

```text
모델은 판단하고, 하네스는 실행한다.
```

모델은 파일을 직접 읽지 않는다. shell을 직접 실행하지 않는다. 캘린더에 직접 일정을 만들지 않는다. 모델은 “이 도구를 이런 입력으로 호출하면 좋겠다”는 요청을 만들고, 실제 실행은 하네스가 한다.

일반적인 function calling에서는 그렇다. built-in tool처럼 플랫폼이 실행을 맡는 경우도 있지만, 그때도 실행 주체는 모델 파라미터 자체가 아니라 모델 바깥의 runtime/tool layer다.

이 구분을 이해하면 “AI agent를 만든다”는 말이 훨씬 선명해진다.

## 먼저 직관으로 이해하기

Chatbot과 agent는 겉보기에는 비슷하다. 둘 다 사용자 질문에 답한다. 하지만 중요한 차이가 있다.

```text
Chatbot
→ 주로 말로 답한다.

Agent
→ 말로 답할 수도 있고,
   도구를 호출하고,
   결과를 보고,
   다시 판단하고,
   필요하면 추가 행동을 한다.
```

예를 들어 사용자가 이렇게 묻는다.

```text
이 repo에서 README를 읽고 요약해줘.
```

파일 내용이 context에 없고, 파일 읽기 tool이나 retrieval도 붙지 않은 순수 응답형 chatbot이라면 이렇게 답할 수밖에 없다.

```text
파일 내용을 볼 수 없어서 요약할 수 없습니다.
```

반면 agent는 이런 루프를 돈다.

```text
1. README를 읽어야 한다고 판단
2. file read tool call 생성
3. 하네스가 실제 파일 읽기 실행
4. 파일 내용이 observation으로 돌아옴
5. 모델이 파일 내용을 보고 요약 작성
```

여기서 중요한 것은 2번과 3번의 차이다.

```text
모델이 tool call을 “생성”한다.
하네스가 tool call을 “실행”한다.
```

모델은 실행자가 아니라 다음 행동 후보를 제안하는 판단자다. 그리고 그 판단 자체도 검증 대상이다. 실제 실행 여부는 하네스의 schema validation, 권한, 정책, 사용자 승인, timeout, 비용 예산 같은 기준으로 결정된다.

## 정확한 개념: agentic loop

Agent runtime은 보통 반복 루프를 가진다.

```text
사용자 입력
→ context 조립
→ LLM 호출
→ 답변 또는 tool call 생성
→ tool call이면 하네스가 검증하고 실행
→ 실행 결과를 observation/tool result로 저장
→ 결과를 다시 context에 넣고 LLM 재호출
→ 최종 답변 또는 다음 tool call
```

이 반복을 agentic loop라고 부를 수 있다.

개념만 보이기 위해 중단 조건을 생략하면, 아주 단순한 의사코드는 이렇다.

```text
messages = build_context(user_input)

while true:
    model_output = call_model(messages, tools)

    if model_output is final_answer:
        return model_output

    if model_output is tool_call:
        result = run_tool_safely(model_output)
        messages.append(result)
        continue
```

현실의 agent runtime은 이보다 훨씬 복잡하다. 보통 `max turns`, `max tool calls`, timeout, cancellation, approval pause, guardrail failure, retry, streaming, tool schema validation, human approval, logging, cost budget, context compaction 같은 것이 붙는다.

하지만 뼈대는 단순하다.

```text
모델이 다음 행동을 고른다.
하네스가 행동을 실행한다.
결과가 다시 모델에게 들어간다.
```

## 웹 개발자 비유: Express 서버와 worker

웹 개발자 관점에서는 모델 하나를 전체 애플리케이션으로 보면 안 된다. 모델은 runtime 안의 한 컴포넌트다.

비유하면 이렇다.

```text
LLM
≈ business logic을 제안하는 brain

Agent harness
≈ Express/Nest 서버 + middleware + worker + permission layer + logger
```

Express 서버를 생각해보자.

```text
request 수신
→ auth middleware
→ validation
→ controller
→ service
→ DB/API 호출
→ response
```

여기서 controller 함수 하나만 보고 “이게 전체 서비스다”라고 말하지 않는다. 서비스가 제대로 동작하려면 middleware, DB client, queue, auth, error handler, logger가 필요하다.

Agent도 마찬가지다.

```text
user input
→ context assembly
→ model call
→ tool validation
→ permission check
→ tool runner
→ observation storage
→ model call
→ final response
```

모델은 중요하지만 전체가 아니다.

## 작은 예시: 일정 등록 agent

사용자가 이렇게 말한다.

```text
내일 오전 9시에 병원 예약 리마인드 해줘.
```

Agent가 이 일을 처리하려면 적어도 세 단계가 필요하다.

### 1. 모델의 판단

모델은 context와 tool schema를 보고 이렇게 판단한다.

```text
이건 일반 질문이 아니라 일정 등록 요청이다.
calendar_create_event tool이 필요하다.
날짜는 현재 날짜 기준 내일이다.
시간은 오전 9시다.
제목은 병원 예약 리마인드 정도가 적절하다.
```

모델이 만들 수 있는 것은 이런 tool call이다.

```json
{
  "tool": "calendar_create_event",
  "arguments": {
    "title": "병원 예약 리마인드",
    "start_time": "2026-06-28T09:00:00+09:00",
    "end_time": "2026-06-28T09:10:00+09:00",
    "timezone": "Asia/Seoul"
  }
}
```

### 2. 하네스의 실행

하네스는 이 tool call을 그대로 믿고 무조건 실행하면 안 된다.

해야 할 일이 있다.

```text
input schema 검증
timezone 확인
권한 확인
중복 일정 가능성 확인
외부 Calendar API 호출
성공/실패 결과 기록
```

즉 실제 일정 등록은 모델이 아니라 하네스가 한다.

### 3. 결과 재주입

Calendar API가 성공하면 결과가 돌아온다.

```json
{
  "ok": true,
  "event_id": "evt_123",
  "starts_at": "2026-06-28T09:00:00+09:00"
}
```

이 결과가 다시 모델에게 들어간다. 그러면 모델은 사용자에게 이렇게 답할 수 있다.

```text
내일 오전 9시에 “병원 예약 리마인드” 일정으로 등록했어.
```

여기서 모델은 Calendar API를 직접 호출하지 않았다. 모델은 호출할 도구와 인자를 제안했고, 하네스가 실행했다.

## Codex에서는 어떻게 보이는가

Codex도 같은 구조로 볼 수 있다.

사용자가 이렇게 묻는다.

```text
이 파일 읽고 문제점 찾아줘.
```

모델은 파일 내용을 아직 모를 수 있다. 그러면 tool call을 요청한다.

```text
파일 읽기
검색
shell command
git status
apply_patch
web search
browser interaction
connector 호출
```

구체적인 tool 목록과 권한 모델은 Codex CLI, IDE extension, web, cloud task, connector 설정, 조직 정책에 따라 달라질 수 있다. 여기서는 Codex를 코딩 작업용 agent harness로 이해하기 위한 공통 구조만 본다.

Codex 하네스는 실제로 tool을 실행한다. 그리고 결과를 다시 모델에게 준다.

예:

```text
모델:
README.md를 읽어야겠다.

Codex:
파일 읽기 실행.

tool result:
README.md 내용.

모델:
내용을 읽고 요약하거나 수정 계획 작성.
```

또 다른 예:

```text
모델:
이 변경은 파일 수정이 필요하다.

Codex:
apply_patch 실행.

tool result:
패치 성공/실패.

모델:
성공 여부를 보고 다음 단계 결정.
```

이때 Codex는 단순 실행기만은 아니다. sandbox, approval, hook, workspace 규칙, tool schema 같은 runtime layer도 함께 가진다.

그래서 Codex는 “모델이 들어 있는 채팅창”이라기보다, 코딩 작업을 위한 agent harness에 가깝다.

## Tool call은 실행이 아니라 요청이다

이 장에서 가장 헷갈리기 쉬운 지점이다.

모델이 tool call을 만들었다고 해서 그 순간 외부 세계가 바뀐 것은 아니다.

```text
tool call 생성
≠
tool 실행 완료
```

Tool call은 요청이다. 실제 실행은 하네스가 맡는다.

모델이 만든 tool arguments는 신뢰할 수 있는 실행 명령이 아니라 검증 대상 데이터다.

하네스는 보통 다음을 확인한다.

```text
이 tool이 존재하는가?
입력이 schema에 맞는가?
사용자 승인이 필요한가?
권한상 허용되는가?
외부 상태를 바꾸는가?
timeout이나 retry 정책은 무엇인가?
실패하면 모델에게 어떤 결과를 돌려줄 것인가?
```

이 구분이 있어야 안전한 agent를 만들 수 있다.

모델에게 “삭제해”라는 판단을 맡길 수는 있어도, 실제 삭제를 무조건 실행하게 해서는 안 된다. 하네스가 권한과 안전장치를 가져야 한다.

실행 결과를 부르는 용어도 문맥마다 조금 다르다. ReAct 계열 설명에서는 observation이라고 부르는 일이 많고, OpenAI API 문맥에서는 tool output, function call output, tool result 같은 표현을 쓴다. 이 책에서는 독자가 흐름을 이해하기 쉬운 쪽으로 병기한다.

## 흔한 오해

### 오해 1. “Agent는 모델에게 목표만 주면 알아서 끝까지 한다”

실제로는 runtime이 잘 설계되어 있어야 한다.

모델에게 목표만 던져주면 이런 문제가 생긴다.

```text
필요한 tool이 없음
tool 설명이 애매함
권한이 없음
실패 결과를 해석 못 함
무한히 반복함
잘못된 상태를 사실로 믿음
외부 시스템을 위험하게 변경함
```

좋은 agent는 목표만 가진 모델이 아니라, 좋은 tool, 좋은 context, 좋은 중단 조건, 좋은 권한 모델을 가진 runtime이다.

### 오해 2. “모델이 tool을 쓰면 믿어도 된다”

Tool call은 모델의 제안이다. 검증된 사실이 아니다.

모델은 잘못된 인자를 만들 수 있고, 없는 파일을 읽으려 할 수 있고, 날짜를 잘못 해석할 수 있다.

그래서 하네스는 schema validation, 권한 확인, 에러 처리, 결과 재주입을 해야 한다.

### 오해 3. “Agent 개발은 프롬프트 작성이다”

프롬프트는 중요하다. 하지만 전부는 아니다.

Agent 개발에서 중요한 것은 더 넓다.

```text
context assembly
state management
tool design
permission model
runtime loop
error handling
observability
memory retrieval
evaluation
```

프롬프트는 agent runtime의 한 부품이다.

## 왜 중요한가

Agent를 모델 하나로 보면 문제를 엉뚱하게 고친다.

예를 들어 agent가 일정 등록에 실패했다.

초보자는 이렇게 생각할 수 있다.

```text
모델이 멍청해서 실패했다.
더 좋은 모델을 써야 한다.
```

하지만 실제 원인은 다를 수 있다.

```text
calendar tool schema가 애매했다.
timezone이 context에 없었다.
Calendar API 권한이 만료됐다.
tool error를 모델에게 제대로 돌려주지 않았다.
중복 일정 확인이 없었다.
사용자 승인 단계가 빠졌다.
```

이 경우 모델 교체보다 runtime 수정이 정답이다.

좋은 agent 개발자는 모델만 보지 않는다. 전체 루프를 본다.

```text
입력
context
model decision
tool call
permission
execution
observation
next decision
final answer
```

## 요약

```text
1. Agent는 모델 하나가 아니라 LLM + runtime이다.
2. 이 책에서 agent는 LLM이 행동 후보를 만들고 runtime이 실행·검증·상태 관리를 맡는 시스템을 뜻한다.
3. 모델은 실행자가 아니라 행동 후보를 제안하는 판단자이며, 그 판단도 검증 대상이다.
4. Tool call은 실행 완료가 아니라 실행 요청이다.
5. 하네스는 tool validation, 권한 확인, 실행, 결과 재주입을 담당한다.
6. Codex는 코딩 작업을 위한 agent harness로 볼 수 있다. 단, 구체적인 tool과 권한은 제품 표면과 설정에 따라 달라진다.
7. Agent 개발은 프롬프트 작성이 아니라 runtime 설계다.
```

## 생각해볼 질문

1. 내가 만든 Telegram LLM 서버에서 모델이 한 일과 Node 서버가 한 일은 각각 무엇이었나?
2. 일정 등록 실패가 생긴다면 모델 문제와 runtime 문제를 어떻게 나눠서 볼 수 있을까?
3. 내가 만든 tool call 중 실제 실행 전에 사용자 승인이 필요한 것은 무엇인가?
4. Codex가 shell command나 file patch를 실행할 때, 모델과 하네스의 역할은 어떻게 나뉘는가?
5. “에이전트를 만든다”를 “모델에게 목표를 준다”가 아니라 “runtime loop를 설계한다”로 바꾸면 무엇을 더 설계해야 할까?
