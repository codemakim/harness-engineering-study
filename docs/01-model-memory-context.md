[← README](../README.md)

# 1장. 모델은 왜 기억하는 것처럼 보일까?

## 이 장에서 배울 것

이 장의 목표는 하나다.

```text
모델이 “기억한다”는 느낌을
“context에 정보가 들어왔다”는 구조로 바꿔 이해하기.
```

AI를 처음 쓰면 모델이 사람처럼 기억하는 것처럼 보인다. 방금 말한 내용을 이어받고, 이전 대화의 뉘앙스를 따라오고, 어떤 때는 오래전 취향까지 아는 것처럼 답한다.

하지만 에이전트 하네스를 공부할 때는 먼저 이 감각을 내려놓는 편이 좋다. 모델을 사람처럼 기억하는 존재로 보면, 나중에 `AGENTS.md`, memory, tool, hook, skill이 전부 애매하게 섞인다.

기초 모델은 이렇게 잡자.

```text
LLM = 지금 주어진 context를 보고 다음 출력을 만드는 모델
```

조금 더 개발자스럽게 쓰면:

```text
model(context) -> next_message_or_tool_call
```

이 한 줄이 이 책의 첫 번째 기둥이다.

## 먼저 직관으로 이해하기

사용자가 이렇게 묻는다고 해보자.

```text
전에 말한 내 Telegram 봇 구조 기억나?
```

모델이 이렇게 답한다.

```text
응. Telegram webhook을 Node 서버로 받고,
최근 대화 몇 건과 장기 기억 검색 결과를 context에 붙인 다음,
LLM에 넘기는 구조였지.
```

겉으로 보면 모델이 “기억”한 것 같다.

하지만 실제로는 다음 중 하나가 일어났을 가능성이 높다.

```text
1. 같은 대화 thread 안에 이전 내용이 아직 남아 있었다.
2. 이전 대화가 summary로 압축되어 context에 들어왔다.
3. memory 파일이나 DB에서 관련 내용이 검색되어 context에 들어왔다.
4. 사용자가 만든 문서를 tool로 읽어 context에 넣었다.
```

즉 모델이 마음속 어딘가에 사용자의 Telegram 봇을 저장해둔 것이 아니라, 하네스가 관련 정보를 모델 입력에 다시 넣어준 것이다.

이 구분이 중요하다.

```text
모델이 기억했다
```

보다 더 정확한 말은:

```text
모델이 볼 수 있는 context 안에 그 정보가 있었다
```

이다.

## 정확한 개념

LLM은 context window 안의 정보를 보고 다음 token, 다음 문장, 다음 tool call을 만든다.

여기서 context란 단순히 “사용자가 방금 쓴 질문”만 뜻하지 않는다. 보통 여러 층의 정보가 함께 들어간다.

```text
system instruction
developer instruction
user message
conversation history
conversation summary
retrieved memory
tool descriptions
tool results
environment information
project instructions
hook output
```

모델은 이 전체 묶음을 보고 답한다.

그래서 같은 질문이라도 context가 다르면 답이 달라진다.

```text
질문:
“전에 말한 구조 기억나?”

context A:
아무 이전 정보 없음
→ 모델은 모른다고 하거나 추측해야 한다.

context B:
최근 대화에 Telegram 봇 구조가 들어 있음
→ 모델은 자연스럽게 이어서 답한다.

context C:
장기 기억 검색 결과에 Telegram 봇 구조가 들어 있음
→ 모델은 오래전 일을 기억하는 것처럼 답한다.
```

여기서 핵심은 “모델의 기억력”보다 “하네스가 어떤 context를 넣었는가”다.

## 웹 개발자 비유: stateless API와 session

웹 개발자에게 가장 좋은 비유는 HTTP request다.

HTTP 요청 하나만 보면 서버는 사용자를 기억하지 못한다.

```text
GET /me
```

이 요청 자체에는 “이 사람이 누구인지”가 없다. 그런데 실제 서비스에서는 `/me`가 잘 동작한다.

왜냐하면 request가 controller에 도착하기 전에 여러 일이 일어난다.

```text
브라우저가 cookie를 보냄
→ 서버 middleware가 session ID를 읽음
→ session store나 DB에서 user를 조회
→ request.user에 붙임
→ controller는 request.user를 보고 응답
```

controller 입장에서는 사용자를 “아는 것처럼” 보인다. 하지만 controller가 사용자를 기억한 게 아니다. middleware가 request에 user 정보를 붙여준 것이다.

LLM도 비슷하게 볼 수 있다.

```text
사용자 질문
→ 하네스가 대화 history를 붙임
→ memory store에서 관련 기록을 검색해 붙임
→ tool schema와 project instruction을 붙임
→ 모델 호출
→ 모델은 context를 보고 답함
```

비유를 나란히 놓으면 이렇다.

| 웹 서버 | LLM 에이전트 |
|---|---|
| HTTP request | user message |
| cookie/session ID | thread ID / user ID / memory key |
| session middleware | context assembler |
| DB lookup | memory retrieval |
| `request.user` | retrieved user/project context |
| controller | model inference |

그래서 “모델이 기억한다”는 말은 초보 단계에서는 편한 표현이지만, 시스템을 만들 때는 위험한 표현이다.

더 정확한 질문은 이것이다.

```text
이 요청에서 모델에게 어떤 context가 붙었지?
```

## 작은 예시: 직접 만든 Telegram LLM 서버

사용자는 이미 비슷한 구조를 직접 만든 적이 있다. Telegram으로 질문하면 Node 서버가 받고, 필요한 context를 조립해서 LLM에 넘기는 구조다.

대략 이런 흐름이었다.

```text
Telegram 채팅
→ Node 서버가 webhook/request 수신
→ 최근 대화 몇 건 조회
→ 장기 기억 파일/DB에서 관련 내용 검색
→ 적합도 비교로 넣을 기억 선택
→ tool 설명과 input/output schema 추가
→ 현재 질문과 함께 LLM 호출
→ 필요하면 tool 실행
→ 답변 또는 일정 등록
```

이건 단순 장난감이 아니라, 에이전트 하네스의 핵심을 이미 건드린 구조다.

예를 들어 사용자가 이렇게 물었다고 하자.

```text
내일 오전에 병원 예약 리마인드 해줘.
```

Node 서버는 모델에게 질문만 넘기지 않는다.

아마 이런 정보를 함께 넣었을 것이다.

```text
현재 날짜/시간
사용자의 timezone
최근 대화
사용자의 일정 관련 선호
일정 등록 tool의 설명
일정 등록 tool의 input schema
```

그러면 모델은 이런 판단을 할 수 있다.

```text
이건 일반 답변이 아니라 일정 등록 요청이다.
calendar_create_event tool을 호출해야 한다.
날짜는 내일 오전이다.
timezone은 사용자 기본 timezone을 써야 한다.
```

이때 모델이 “일정 등록 기능을 원래 알고 있었다”고 보면 안 된다. 하네스가 tool 설명을 context에 넣었기 때문에 모델이 그 tool을 사용할 수 있었던 것이다.

## Codex에서는 어떻게 보이는가

Codex도 원리는 같다. 다만 사용자가 직접 만든 Node 서버보다 더 많은 context surface를 가진다.

한 번의 질문이 들어오면 Codex는 개념적으로 이런 것들을 준비할 수 있다.

```text
현재 대화 thread
이전 대화 summary
현재 작업 디렉토리
shell / OS / 날짜 / timezone
사용 가능한 tool 목록과 schema
Skill 목록과 description
선택된 Skill의 SKILL.md
AGENTS.md guidance
Hook output
Memory
방금 실행한 tool 결과
```

정확한 내부 payload 형태는 제품 내부 구현이므로 그대로 볼 수 없다. 하지만 공부할 때 중요한 것은 이 관점이다.

```text
내 질문만 모델에게 가는 것이 아니다.
Codex 하네스가 조립한 여러 context layer가 함께 간다.
```

그래서 Codex를 이해하려면 “모델이 똑똑하다”에서 멈추면 안 된다.

이렇게 물어야 한다.

```text
이번 답변에 영향을 준 instruction은 무엇인가?
AGENTS.md가 로드되었는가?
어떤 skill이 선택되었는가?
hook이 추가 context를 넣었는가?
tool 결과가 다시 모델에게 들어갔는가?
memory가 사용되었는가?
```

이 질문들이 하네스 엔지니어링의 시작이다.

## 흔한 오해

### 오해 1. “프론티어 모델은 자체 기억이 있으니까 괜찮다”

모델이 강할수록 긴 context를 잘 읽고, 단서를 잘 연결하고, 요약된 정보를 잘 복원한다. 그래서 더 기억력이 좋아 보인다.

하지만 시스템 설계 관점에서는 여전히 이렇게 보는 편이 안전하다.

```text
기억처럼 보이는 것 = 현재 context에 들어온 정보를 잘 사용한 결과
```

모델 능력이 좋아질수록 context를 잘 쓰는 것이지, 내 서비스의 DB나 파일을 허락 없이 자동으로 뒤지는 것은 아니다.

### 오해 2. “대화를 많이 넣으면 무조건 좋아진다”

그렇지 않다.

너무 적게 넣으면 필요한 배경이 빠진다. 하지만 너무 많이 넣으면 중요한 정보가 묻힌다.

```text
context 부족
→ 모델이 모름

context 과다
→ 모델이 중요한 것을 못 찾음

stale context
→ 모델이 오래된 정보를 사실처럼 사용함
```

좋은 하네스는 “많이 넣는 시스템”이 아니라 “지금 필요한 것을 잘 고르는 시스템”이다.

### 오해 3. “모델이 틀리면 모델만 바꾸면 된다”

가끔은 더 좋은 모델이 답이다. 하지만 많은 문제는 context 문제다.

```text
필요한 기억이 안 들어갔다.
tool 설명이 애매했다.
현재 날짜/timezone이 없었다.
오래된 memory가 들어갔다.
사용자 의도를 구분할 instruction이 없었다.
```

이런 경우 모델만 바꿔도 잠깐 나아질 수 있지만, 근본 해결은 하네스 설계다.

## 왜 중요한가

이 장을 이해하면 질문 방식이 바뀐다.

예전 질문:

```text
왜 모델이 이걸 기억 못 하지?
왜 모델이 이상한 답을 하지?
```

좋은 질문:

```text
모델이 봐야 할 정보가 context에 들어갔나?
너무 많은 정보가 들어가서 묻히진 않았나?
검색된 memory가 최신인가?
tool 설명이 호출 조건을 잘 알려주고 있나?
결과를 다시 모델에게 명확히 돌려줬나?
```

이 관점이 있어야 뒤의 장들이 이해된다.

```text
AGENTS.md
→ 어떤 규칙을 기본 context로 넣을 것인가

Skill
→ 어떤 작업 매뉴얼을 필요할 때만 넣을 것인가

Hook
→ 어떤 lifecycle 순간에 context를 동적으로 만들 것인가

Plugin
→ 이런 context/tool/hook 묶음을 어떻게 배포할 것인가

Memory
→ 어떤 과거 정보를 지금 질문에 맞게 가져올 것인가
```

## 요약

```text
1. LLM은 지금 주어진 context를 보고 답한다.
2. 기억처럼 보이는 것은 대개 history, summary, memory, tool result가 context에 들어왔기 때문이다.
3. 웹 서버에서 session middleware가 request.user를 붙이듯, agent harness는 LLM request에 필요한 정보를 붙인다.
4. 좋은 agent는 많은 정보를 넣는 시스템이 아니라, 지금 필요한 정보를 잘 고르는 시스템이다.
5. 하네스 엔지니어링의 첫 질문은 “모델이 뭘 봤는가?”다.
```

## 생각해볼 질문

1. 내가 만든 LLM 앱에서 “기억”이라고 부른 기능은 실제로 어디에 저장되어 있었나?
2. 최근 대화, 장기 기억, tool 결과 중 어떤 것이 context에 들어갔는지 로그로 확인할 수 있었나?
3. 모델이 틀린 답을 했을 때, 모델 문제가 아니라 context 조립 문제였던 사례가 있었나?
4. 지금 사용하는 Codex 답변에서 어떤 context layer가 영향을 주었을지 추측해볼 수 있나?
5. “모델이 기억한다” 대신 “하네스가 무엇을 넣었나?”라고 질문하면 어떤 설계가 달라질까?
