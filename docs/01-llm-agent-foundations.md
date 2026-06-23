[← README](../README.md)

# 0. LLM과 에이전트의 기본 동작 원리

이 장은 나머지 모든 내용의 바닥이다. `AGENTS.md`, Skill, Hook, Plugin, Memory, MCP를 이해하려면 먼저 “모델”과 “에이전트 하네스”를 분리해서 봐야 한다.

사용자는 이전에 직접 Telegram 기반 LLM 서버를 만든 경험이 있다.

대략 이런 구조였다.

```text
Telegram 채팅
→ Node 서버가 webhook/request 수신
→ 최근 대화 몇 건 조회
→ 장기 기억 파일/DB에서 관련 내용 검색
→ 필요한 tool 설명과 input/output schema 준비
→ 현재 질문과 함께 context 구성
→ LLM 호출
→ 필요하면 tool 실행
→ tool 결과를 다시 LLM에 넣음
→ 최종 답변 또는 일정 등록
```

이 경험은 Codex를 이해하는 데 아주 좋은 출발점이다. Codex도 본질은 비슷하다. 차이는 직접 만든 Node 서버보다 훨씬 더 큰 context builder, tool runner, permission layer, plugin system, lifecycle hook system을 제품 안에 가지고 있다는 점이다.

### 0.1 모델은 무엇인가

가장 단순하게 보면 LLM은 이런 함수처럼 생각할 수 있다.

```text
model(context) → next_message_or_tool_call
```

모델은 “지금 들어온 context”를 보고 다음 출력을 만든다. 그 출력은 일반 텍스트 답변일 수도 있고, tool call 요청일 수도 있다.

중요한 감각:

```text
모델 자체는 네 프로젝트 파일을 직접 읽지 않는다.
모델 자체는 shell command를 직접 실행하지 않는다.
모델 자체는 네 장기 기억 DB를 자동으로 뒤지지 않는다.
모델 자체는 세션 밖의 일을 마법처럼 계속 기억하지 않는다.
```

그렇게 보이는 이유는 하네스가 필요한 정보를 context로 넣어주기 때문이다.

예:

```text
최근 대화가 context에 들어감
→ 모델이 기억하는 것처럼 답함

memory 검색 결과가 context에 들어감
→ 모델이 오래전 일을 아는 것처럼 답함

tool schema가 context에 들어감
→ 모델이 어떤 tool을 호출해야 할지 판단함

tool output이 다시 context에 들어감
→ 모델이 파일을 읽은 것처럼 그 결과를 사용함
```

즉 “기억”처럼 보이는 많은 것은 실제로는 “적절한 context 주입”이다.

### 0.2 에이전트는 무엇인가

에이전트는 모델 하나가 아니다. 모델을 둘러싼 실행 루프다.

```text
사용자 입력
→ context 조립
→ LLM 호출
→ 답변 또는 tool call 생성
→ tool call이면 하네스가 실제 실행
→ 실행 결과를 다시 context에 추가
→ 다시 LLM 호출
→ 최종 답변
```

여기서 역할이 갈린다.

```text
LLM
= 판단하고, 다음 말을 만들고, 어떤 tool이 필요할지 제안함

하네스
= context를 모으고, tool을 실제 실행하고, 권한을 검사하고,
   실행 결과를 다시 모델에게 전달함
```

웹 개발자 관점으로 비유하면:

```text
LLM = pure function에 가까운 inference engine
Agent harness = server runtime + middleware + router + DB access + job runner + permission layer
```

모델이 똑똑한 것은 맞다. 하지만 제품으로서의 “에이전트”는 모델만으로 되지 않는다. 모델 주변에 실행 환경이 있어야 한다.

### 0.3 Codex에서 한 번의 요청은 어떻게 흘러가나

정확한 내부 직렬화 형식은 제품 내부 구현이므로 그대로 볼 수는 없다. 하지만 동작 모델은 이렇게 이해하면 된다.

```text
1. 사용자가 메시지를 보냄

2. Codex가 현재 thread/session 상태를 준비
   - 이전 대화 또는 요약
   - 현재 작업 디렉토리, shell, 날짜, timezone 같은 환경 정보
   - system/developer/user instruction
   - 사용 가능한 tool 목록과 schema
   - skill 목록과 description
   - plugin이 제공하는 instruction/hook/MCP 정보
   - AGENTS.md에서 로드된 project/global guidance
   - memory summary 또는 관련 memory

3. UserPromptSubmit hook이 있으면 실행될 수 있음
   - 예: Ponytail mode tracker
   - user prompt 검사
   - 상태 파일 갱신
   - 추가 context 출력

4. LLM 호출
   - 모델이 답변을 만들거나 tool call을 요청

5. tool call이 있으면 Codex가 실행
   - shell command
   - 파일 읽기/수정
   - apply_patch
   - web search
   - browser/chrome/computer control
   - Gmail/Slack/GitHub/Notion 같은 connector
   - MCP tool

6. tool 결과가 다시 conversation context에 들어감

7. 모델이 결과를 보고 다음 행동 결정
   - 추가 tool call
   - 설명
   - 최종 답변

8. Stop/PostToolUse/PostCompact 같은 hook이 실행될 수 있음
```

짧게:

```text
모델은 판단한다.
Codex는 실행한다.
실행 결과는 다시 모델에게 먹인다.
```

### 0.4 Codex가 매 요청에 함께 줄 수 있는 것들

사용자가 보는 것은 “내 질문 본문”뿐이지만, 모델은 보통 그 외의 많은 레이어를 같이 받는다.

개념적으로는 이런 층들이 있다.

```text
System instructions
→ 안전, 역할, 전체 제품 규칙 같은 최상위 지시

Developer instructions
→ Codex로서 어떻게 일할지, 도구 사용 규칙, 파일 수정 규칙, 현재 모드

User message
→ 사용자가 방금 쓴 질문

Conversation history / summary
→ 이 thread의 이전 대화 일부 또는 압축 요약

Environment context
→ cwd, shell, 날짜, timezone, filesystem permission 등

Tool definitions
→ 호출 가능한 tool 이름, input schema, 사용 지침

Skill list
→ 사용 가능한 skills의 name, description, path

Loaded skill instructions
→ 선택된 skill의 SKILL.md 내용

AGENTS.md guidance
→ 세션 시작 때 로드된 global/project instruction chain

Hook output
→ lifecycle hook이 stdout으로 낸 additionalContext, decision 등

Memory
→ 관련 memory summary나 검색된 기억 정보

Tool outputs
→ shell/file/web/connector 실행 결과
```

이걸 네 Telegram 서버와 나란히 놓으면 이렇게 된다.

| 네가 만든 Telegram LLM 서버 | Codex |
|---|---|
| 최근 대화 select | thread history / summary |
| 장기 기억 검색 | memory system / files / external stores |
| tool schema 직접 구성 | Codex tool definitions / MCP tools |
| Node server가 tool 실행 | Codex tool runner |
| prompt builder | Codex context assembly |
| 일정 등록 API/tool | connector / MCP / local tool |
| 권한 로직 직접 구현 | sandbox / approvals / trust model |
| system prompt 직접 작성 | system/developer/AGENTS/skill/hook instruction layers |

그래서 Codex를 공부한다는 것은 “내가 직접 만들었던 LLM 앱 서버가 제품 수준에서는 어떻게 확장되는가”를 공부하는 것과 비슷하다.

### 0.5 기억은 어디에 있는가

모델 안에 사용자별 기억 DB가 들어 있다고 생각하면 헷갈린다. 더 좋은 모델은 긴 context를 더 잘 다루고, 단서를 더 잘 연결하고, 요약된 정보를 더 잘 복원한다. 하지만 원리는 여전히 context 기반으로 보는 편이 안전하다.

기억처럼 작동하는 것들의 실제 위치:

```text
1. 현재 thread history
2. 압축된 conversation summary
3. memory 파일/DB/vector store
4. AGENTS.md 같은 지속 instruction 파일
5. Notion/Gmail/Calendar/Slack/GitHub 같은 외부 app data
6. 사용자가 만든 문서나 프로젝트 파일
7. tool이 방금 읽어온 출력
```

그래서 이 문서 자체도 중요하다.

```text
/Users/jhkim/Documents/Codex/harness-engineering-study/README.md
```

이 파일은 모델 내부 기억이 아니다. 하지만 다음 세션에서 “이 README 읽고 이어가자”라고 하면, Codex가 파일을 읽고 그 내용을 다시 context로 가져올 수 있다. 즉 외부 persistent memory 역할을 한다.

### 0.6 `AGENTS.md`, Skill, Hook, Plugin은 전부 context 조립 장치다

이 문서의 나머지 개념들은 전부 이 기본 원리 위에 있다.

```text
AGENTS.md
= 시작 시 context에 들어가는 project/global instruction chunk

Skill
= task에 맞을 때 로드되는 workflow instruction chunk

Hook
= 특정 lifecycle event에서 실행되어 동적으로 만드는 context/decision chunk

Plugin
= skill/hook/MCP/assets를 설치 가능한 단위로 묶은 package

MCP/App
= 외부 시스템을 tool로 연결하는 통로

Memory
= 과거 정보를 검색해 현재 context에 넣는 장치
```

그래서 “Caveman과 Ponytail이 어떻게 모델 행동을 바꾸는가?”라는 질문도 결국 이렇게 해석된다.

```text
어떤 문구를
어느 lifecycle에서
얼마나 반복적으로
어떤 우선순위와 상태 조건으로
모델 context에 넣는가?
```

문구 자체도 중요하지만, 하네스 엔지니어링에서는 “문구를 넣는 위치와 타이밍”이 결정적으로 중요하다.

### 0.7 하네스 엔지니어링의 진짜 질문

하네스 엔지니어링은 멋진 프롬프트 한 문장을 쓰는 일이 아니다. 모델 주변의 실행 환경을 설계하는 일이다.

항상 물어야 할 질문:

```text
1. 현재 모델에게 실제로 들어가는 context는 무엇인가?
2. 누가 그 context를 고르는가?
3. context는 세션 시작 때 한 번 정해지는가, 매 턴 갱신되는가?
4. 어떤 정보는 memory에서 검색되고, 어떤 정보는 고정 instruction인가?
5. tool schema는 모델에게 어떻게 설명되는가?
6. 모델이 tool call을 만들면 누가 실제로 실행하는가?
7. tool 결과는 어떤 형식으로 다시 모델에게 돌아오는가?
8. 위험한 tool call은 누가 막는가?
9. 사용자의 승인/trust는 어느 레이어에서 필요한가?
10. 대화가 길어져 context가 압축될 때 무엇이 살아남는가?
```

이 질문을 계속 붙잡으면 된다. 웹 개발자로 비유하면, 처음에는 HTTP request/response와 middleware를 이해하는 순간 서버가 보이기 시작한다. AI agent도 비슷하다. 처음에는 “모델이 답한다”로 보이다가, 어느 순간 “아, 이건 context, tools, memory, permissions, lifecycle을 가진 runtime이구나”로 보이기 시작한다.

그때부터 `AGENTS.md`, Hook, Skill, Plugin은 각각 이상한 마법 기능이 아니라, 에이전트 runtime을 구성하는 부품으로 보인다.

---
