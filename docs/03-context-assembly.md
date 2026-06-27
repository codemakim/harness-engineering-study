[← README](../README.md)

# 3장. 답변 전 실제로 무슨 일이 일어날까?

## 이 장에서 배울 것

1장에서는 모델이 “기억하는 것처럼 보이는 이유”를 배웠다.

```text
모델이 본다
→ context에 들어온 정보를 본다
```

2장에서는 에이전트가 모델 하나가 아니라 runtime이라는 점을 배웠다.

```text
모델은 판단하고,
하네스는 실행한다.
```

이제 둘을 합친다.

사용자가 질문 하나를 보내면 모델이 그 문장만 보고 답하는 것이 아니다. 답변 전에는 하네스가 여러 조각을 모아 “모델이 볼 수 있는 상태”를 만든다.

이 장의 중심 문장은 이것이다.

```text
좋은 답변은 좋은 질문만이 아니라,
좋은 context assembly에서 나온다.
```

## 먼저 직관으로 이해하기

사용자는 이렇게 한 줄만 보낸다.

```text
2장 이어서 3장 작성해볼까?
```

겉으로 보면 모델은 이 문장만 보고 답하는 것 같다. 하지만 실제로는 그렇지 않다.

이 요청이 의미를 가지려면 여러 배경이 필요하다.

```text
이 repo가 어디에 있는지
이미 1장과 2장이 작성되었는지
집필 계획이 무엇인지
레거시 3장이 어떤 내용이었는지
문서 톤이 어떤지
GitHub에 push하는 흐름이 있는지
```

사용자 질문은 짧지만, 하네스가 준비해야 하는 맥락은 꽤 길다.

웹 개발로 치면 controller에 도착한 request body만 보는 것과 비슷하다.

```json
{
  "message": "3장 작성해볼까?"
}
```

이 body만으로는 할 수 있는 일이 별로 없다. 하지만 middleware와 service layer가 다음을 붙여준다면 이야기가 달라진다.

```text
authenticated user
workspace path
git status
current branch
writing plan file
legacy chapter file
available tools
permission policy
```

그제야 “3장을 작성한다”는 일이 실제 작업이 된다.

LLM agent도 똑같다. user message는 시작점일 뿐이고, 답변 품질은 그 주변에 무엇이 조립되었는지에 크게 좌우된다.

## 정확한 개념: context assembly

Context assembly는 모델 호출 전에 하네스가 모델에게 보여줄 상태를 구성하는 과정이다.

여기서 context는 1장에서 정의한 넓은 의미다.

```text
모델의 출력을 조건화하는 모델 가시 상태 전체
```

반드시 하나의 긴 문자열일 필요는 없다. 구현에 따라 역할이 구분된 메시지 배열, tool schema, tool output, conversation state, 환경 메타데이터, 압축 요약, 캐시된 prefix 같은 구조가 섞일 수 있다.

개념적으로는 이런 흐름이다. 아래 순서는 실제 구현 순서를 엄밀히 나타낸 것이 아니라, 모델 호출 전에 어떤 종류의 정보가 모델 가시 상태에 참여할 수 있는지를 보여주는 개념적 흐름이다. 실제 runtime에서는 일부 layer가 별도 필드, 서버 측 conversation state, 캐시된 prefix, 세션 초기화 결과, tool result item 등으로 관리될 수 있다.

```text
사용자 입력
→ 대화 history / summary 선택
→ 현재 환경 정보 추가
→ 프로젝트 지시문 추가
→ 사용 가능한 tool schema 추가
→ 필요한 skill instruction 로드
→ hook output 반영
→ memory 검색 결과 반영
→ 모델 호출
→ tool call이면 실행 결과를 다시 추가
→ 모델 재호출
```

중요한 점은 이것이다.

```text
질문 하나가 모델에게 그대로 가는 것이 아니다.
하네스가 여러 layer를 조립한 뒤 모델에게 보낸다.
```

## 모델에게 들어갈 수 있는 layer들

아래 목록은 모든 제품과 모든 세션에서 항상 전부 들어간다는 뜻이 아니다. 제품 표면, 설정, 권한, 활성화된 plugin, 현재 대화 상태에 따라 달라진다.

그래도 agent harness를 이해하려면 이 layer들을 알아야 한다.

### 1. System instruction

가장 높은 수준의 지시다. 안전, 역할, 전체 제품 규칙 같은 것이 여기에 해당한다.

사용자는 보통 이 layer를 직접 보거나 수정하지 못한다.

### 2. Developer instruction

제품이나 실행 환경이 모델에게 주는 작업 방식이다.

여기서 system instruction과 developer instruction은 OpenAI API의 역할 구분과 제품 내부 instruction layer를 설명하기 위한 개념적 분류다. Codex 같은 제품에서는 `AGENTS.md`, hook output, 설정값, 권한 정책 등이 내부 instruction chain과 결합되어 모델 행동에 영향을 줄 수 있다.

예:

```text
도구 사용 규칙
파일 수정 방식
응답 스타일
현재 활성 모드
권한 정책
```

이 repo에서 사용 중인 Ponytail mode 같은 것도 이런 상위 지시와 결합되어 모델 행동에 영향을 줄 수 있다.

### 3. User message

사용자가 방금 입력한 메시지다.

많은 사람이 이것만 모델에게 간다고 생각하지만, 실제로는 여러 layer 중 하나다.

### 4. Conversation history / summary

현재 thread의 이전 대화나 압축 요약이다.

긴 대화에서는 모든 원문을 그대로 유지하기 어렵다. 그래서 일부는 요약되거나 압축될 수 있다. 이때 빠진 세부 정보는 나중에 정확히 복원된다고 기대하면 안 된다.

### 5. Environment context

현재 작업 환경에 관한 정보다.

예:

```text
cwd
shell
current date
timezone
filesystem permission
workspace root
```

이런 값은 모델이 상대 경로, 날짜 표현, shell 명령을 해석하는 데 도움을 준다.

다만 environment context가 보인다고 해서 모델이 실제로 모든 파일을 읽거나 shell을 실행할 수 있다는 뜻은 아니다. 실제 접근 가능 여부는 별도의 permission, sandbox, runtime 정책에 의해 제한된다.

### 6. Tool definitions

모델이 호출할 수 있는 tool의 이름, 설명, input schema다.

2장에서 말했듯 tool schema는 모델을 위한 API 문서다. 모델은 이 정보를 보고 “어떤 tool을 어떤 인자로 호출할지” 판단한다.

### 7. Tool outputs

Tool을 실행한 결과다.

예:

```text
파일 내용
shell stdout/stderr
검색 결과
API 응답
git status
```

Tool output은 모델이 외부 세계를 직접 본 것처럼 보이게 만든다. 하지만 실제로는 하네스가 실행한 결과가 다시 context에 들어온 것이다.

Tool output도 품질 관리 대상이다. 권위 있는 사실처럼 보일 수 있지만 실제로는 tool runner가 반환한 데이터다. 실패, partial result, stale result, 권한 오류, stdout/stderr 잘림, schema mismatch 같은 상태를 하네스가 명시적으로 모델에게 전달해야 한다.

### 8. AGENTS.md guidance

Codex는 시작 시 instruction chain을 만들며 `AGENTS.md` 계열 파일을 읽어 project/global guidance를 구성할 수 있다.

공식 문서 기준으로는 조금 더 구체적이다.

```text
Global scope
→ Codex home의 AGENTS.override.md가 있으면 그것을 우선
→ 없으면 AGENTS.md를 읽음

Project scope
→ project root에서 현재 working directory까지 내려오며 확인
→ 각 directory에서 AGENTS.override.md, AGENTS.md, fallback instruction file 순서

Merge order
→ root에서 current directory 방향으로 이어 붙임
→ 더 가까운 directory의 guidance가 뒤에 오므로 앞선 guidance를 덮는 효과를 낼 수 있음
```

또한 combined guidance에는 크기 제한이 있다. 기본값은 32 KiB다.

이 layer는 repo 규칙, 테스트 명령, 작업 방식 같은 “프로젝트 헌법”에 가깝다.

### 9. Skill list and loaded skill instructions

Codex Skills는 처음부터 모든 `SKILL.md` 본문을 넣는 방식이 아니다.

공식 문서 기준으로 Codex는 skill의 name, description, path 같은 목록을 먼저 알고 있다가, 작업에 맞는 skill이 선택되면 그때 전체 `SKILL.md`를 읽는다. 이 방식을 progressive disclosure라고 부른다.

즉 두 단계가 있다.

```text
Skill list
→ 어떤 skill이 있는지 알려주는 얇은 layer

Loaded skill instructions
→ 선택된 skill의 실제 작업 지시문
```

따라서 skill의 `description`은 단순 설명문이 아니다. 어떤 상황에서 이 skill을 선택할지 결정하는 retrieval/selection trigger 역할을 한다. 너무 넓거나 모호하면 잘못 선택될 수 있고, 너무 좁으면 필요한 상황에도 선택되지 않을 수 있다.

### 10. Hook output

Hook은 lifecycle event에 실행되는 command다.

예:

```text
SessionStart
UserPromptSubmit
PreToolUse
PostToolUse
Stop
```

Hook output의 의미는 event마다 다르다. `SessionStart` 같은 hook의 stdout이나 `additionalContext`는 extra developer context로 들어갈 수 있다. 반면 `PreToolUse`에서는 plain text stdout이 context로 들어가지 않고, JSON 형태의 permission decision 등으로 tool 실행을 제어할 수 있다.

Ponytail처럼 `SessionStart`에서 mode instruction을 넣거나, `UserPromptSubmit`에서 mode 변경을 감지하는 구조가 여기에 해당한다.

### 11. Memory

Memory는 모델 내부에 마법처럼 저장된 것이 아니라, 제품이나 하네스가 저장하고 검색해서 다시 제공하는 정보다.

예:

```text
사용자 선호
이전 thread 요약
프로젝트 관련 메모
반복되는 작업 방식
```

Memory는 강력하지만 조심해야 한다. 오래되었거나 충돌하는 memory가 들어가면 모델이 틀린 확신을 가질 수 있다.

Memory retrieval은 단순 벡터 검색만 뜻하지 않는다. 제품에 따라 pinned memory, user profile, thread summary, semantic search, rule-based selection, 수동 저장 메모가 함께 사용될 수 있다. 중요한 것은 저장 위치보다 “어떤 정책으로 현재 turn에 주입되는가”다.

## 작은 예시: “3장 써줘” 요청이 처리되는 방식

지금 상황을 예로 들어보자.

사용자 입력:

```text
음 좋아. 그럼 이제 이어서 바로 3장 작성해볼까?
```

하네스가 준비해야 하는 것:

```text
repo path
→ /Users/jhkim/Documents/Codex/harness-engineering-study

writing plan
→ docs/00-writing-plan.md

legacy chapter
→ legacy/2026-06-25-pre-course-rewrite/docs/03-context-assembly.md

current chapters
→ docs/01-model-memory-context.md
→ docs/02-agent-runtime.md

git state
→ 현재 branch, 변경 여부

official docs
→ AGENTS.md, Skills, Hooks, Plugins 등 Codex 표면 확인
```

이것들이 없으면 모델은 “3장”이 무엇인지, 어떤 톤으로 써야 하는지, 어디에 저장해야 하는지 알 수 없다.

즉 좋은 답변은 사용자 질문 자체보다 다음 과정에 달려 있다.

```text
무엇을 읽을지 선택
→ 읽은 내용을 context에 넣기
→ 기존 톤과 계획을 맞추기
→ 필요한 tool을 실행하기
→ 결과를 반영해 문서 작성하기
```

## Codex에서는 어떻게 보이는가

Codex에서 context assembly와 관련해 공식적으로 확인 가능한 표면들이 있다.

```text
AGENTS.md
→ global/project guidance를 구성한다.

Skills
→ skill 목록은 얇게 노출되고,
   선택된 skill의 SKILL.md가 필요할 때 로드된다.

Hooks
→ lifecycle event에 command를 실행하고,
   output을 통해 추가 context나 decision을 제공할 수 있다.

Plugins
→ skills, hooks, MCP 설정, assets 등을 설치 가능한 단위로 묶는다.

Memories
→ 이전 세션이나 사용자 관련 정보를 향후 context에 다시 제공하는 표면이 될 수 있다.

MCP / connectors / tools
→ 외부 시스템을 tool로 연결하고,
   실행 결과를 모델에게 다시 제공한다.
```

다만 여기서 조심해야 한다.

```text
공식 문서로 확인 가능한 것
≠
최종 모델 요청 payload의 모든 내부 구조를 그대로 볼 수 있다는 것
```

호스티드 제품의 최종 orchestration은 완전히 공개되어 있지 않을 수 있다. 반면 Codex CLI처럼 오픈소스인 표면에서는 `AGENTS.md`, 환경 context, compaction, skill, hook 처리의 상당 부분을 코드로 확인할 수 있다.

따라서 이 책에서는 이렇게 나눠서 본다.

```text
확인 가능한 표면
→ AGENTS.md, Skills, Hooks, Plugins, tool definitions, memory 기능

개념 모델
→ 이런 표면들이 모델 가시 상태를 구성해 답변에 영향을 준다
```

이 구분을 유지하면 과장하지 않으면서도 agent harness를 잘 이해할 수 있다.

Plugin 자체가 항상 모델 context에 직접 들어가는 것은 아니다. Plugin은 skills, hooks, MCP 설정, assets 등을 설치·배포하는 단위다. 실제 context assembly에는 그 plugin이 제공한 skill description, loaded `SKILL.md`, hook output, MCP tool definition 등이 참여한다.

## 웹 개발자 비유: request lifecycle

웹 서버에서 request는 controller로 바로 떨어지지 않는다.

```text
HTTP request
→ load balancer
→ auth middleware
→ session middleware
→ validation
→ rate limit
→ feature flag
→ controller
→ service
→ DB/API
→ response
```

Controller에서 보는 request는 이미 여러 layer를 거친 결과다.

LLM도 비슷하다.

```text
user message
→ instruction layers
→ history / summary
→ environment context
→ tool definitions
→ project guidance
→ skill instructions
→ hook output
→ memory retrieval
→ model call
→ tool output
→ model call
→ final answer
```

웹 개발자가 controller만 보고 장애를 분석하면 놓치는 것이 많다. AI agent도 모델 답변만 보면 놓치는 것이 많다.

## 흔한 오해

### 오해 1. “내가 쓴 질문만 모델이 본다”

그렇지 않다.

사용자 질문은 중요한 layer지만, 유일한 layer는 아니다. 상위 instruction, conversation history, tool schema, project guidance, hook output, memory 등이 함께 영향을 줄 수 있다.

그래서 모델 답변이 이상할 때는 질문만 다시 쓰는 것보다, 어떤 context layer가 들어갔는지 확인해야 한다.

### 오해 2. “context는 그냥 긴 프롬프트 문자열이다”

입문 단계에서는 그렇게 생각해도 어느 정도 맞지만, 엄밀히는 부족하다.

Tool schema는 구조화된 JSON Schema일 수 있고, tool result는 별도 message item일 수 있으며, conversation state는 서버에 저장되어 이어질 수 있다.

이 책에서 context는 더 넓은 뜻이다.

```text
모델의 출력을 조건화하는 모델 가시 상태 전체
```

### 오해 3. “좋은 context는 많이 넣는 것이다”

많이 넣는 것보다 잘 고르는 것이 중요하다.

좋은 context assembly는 다음을 다룬다.

```text
selection
→ 무엇을 넣을지

ordering
→ 어디에 둘지

compression
→ 얼마나 줄일지

freshness
→ 최신 정보인지

provenance
→ 어디서 온 정보인지

priority
→ 어떤 layer를 더 신뢰할지

conflict
→ 충돌하면 어떻게 표시하거나 해결할지

budgeting
→ 토큰 예산을 어떻게 나눌지
```

이건 1장에서 본 memory 설계와도 이어진다.

### 오해 4. “공식 문서에 있는 모든 기능이 항상 내 세션에 들어간다”

아니다.

기능은 제품 표면, 설정, 설치된 plugin, 조직 정책, 권한, 현재 작업 상태에 따라 달라진다.

예를 들어 skill은 선택되어야 전체 본문이 로드된다. hook은 설정되고 trust되어야 실행된다. memory도 기능과 설정에 따라 달라진다. connector도 연결되어 있어야 쓸 수 있다.

## 왜 중요한가

Context assembly를 이해하면 디버깅 질문이 바뀐다.

나쁜 질문:

```text
모델이 왜 이래?
```

좋은 질문:

```text
모델이 어떤 instruction을 봤지?
필요한 파일을 읽었나?
skill이 로드됐나?
AGENTS.md가 적용됐나?
hook이 추가 context를 넣었나?
memory가 오래된 건 아닌가?
tool output이 모델에게 다시 들어갔나?
```

Agent의 문제는 모델 자체보다 context assembly 문제인 경우가 많다.

```text
필요한 context가 빠짐
오래된 context가 들어감
충돌하는 instruction이 들어감
tool schema가 애매함
tool result가 불충분함
너무 많은 정보가 묻힘
```

이 장을 이해하면 뒤의 `AGENTS.md`, Skill, Hook, Plugin이 왜 나뉘어 있는지도 자연스럽게 보인다.

각각은 모두 “context assembly에 참여하는 표면”이다. 다만 쓰임과 타이밍이 다르다.

## 요약

```text
1. 사용자의 질문 하나가 모델에게 그대로 가는 것이 아니다.
2. 하네스는 history, summary, environment, tool schema, project guidance, skill, hook, memory, tool output 같은 layer를 조립한다.
3. context는 단순 문자열이 아니라 모델의 출력을 조건화하는 모델 가시 상태 전체다.
4. Codex의 AGENTS.md, Skills, Hooks, Plugins는 context assembly에 참여하는 확인 가능한 표면이다.
5. 최종 내부 payload와 모든 orchestration은 제품 표면에 따라 다를 수 있으므로, 확인 가능한 표면과 개념 모델을 구분해야 한다.
6. 좋은 context assembly는 많이 넣는 것이 아니라 필요한 정보를 고르고, 배치하고, 압축하고, 출처·최신성·우선순위를 관리하는 것이다.
```

## 생각해볼 질문

1. 내가 최근에 받은 이상한 LLM 답변은 질문 문제가 아니라 context assembly 문제였을 가능성이 있을까?
2. 내가 만든 Telegram LLM 서버에서는 어떤 정보를 항상 넣고, 어떤 정보를 검색해서 넣었나?
3. 지금 Codex 답변에는 어떤 layer가 영향을 주고 있을까?
4. 내 프로젝트에서 `AGENTS.md`에 넣을 정보와 Skill로 분리할 정보는 어떻게 다를까?
5. context를 많이 넣는 대신 잘 고르려면 어떤 로그나 관찰 도구가 필요할까?
