[← README](../README.md)

# 8장. Memory design: 모델이 기억하는 게 아니라 하네스가 가져온다

## 이 장에서 배울 것

1장에서 우리는 이렇게 말했다.

```text
LLM은 기억하는 것이 아니라 context를 읽는다.
```

8장에서는 이 말을 실제 설계 문제로 바꾼다.

처음 AI agent를 만들 때 가장 헷갈리는 말이 “memory”다. 말만 들으면 모델 안 어딘가에 사용자의 과거가 저장되어 있고, 모델이 필요할 때 꺼내 보는 것처럼 느껴진다.

하지만 하네스 엔지니어링 관점에서는 다르게 봐야 한다.

이 장의 중심 문장은 이것이다.

```text
Memory는 모델의 신비한 능력이 아니라,
하네스가 과거 정보를 고르고 요약해서 context에 다시 넣는 설계다.
```

모델은 최종 호출 순간에 들어온 context를 본다.

그 context 안에 과거 대화, 요약, 사용자 선호, 프로젝트 규칙, 검색된 문서, tool output이 들어 있으면 모델은 “기억하는 것처럼” 답한다.

반대로 사용자별·프로젝트별·현재 상태 정보가 context에 없고 tool로도 확인할 수 없다면, 모델은 그것을 안정적으로 알 수 없다.

## 먼저 직관으로 이해하기

웹 개발자는 이미 memory design을 해본 적이 있다.

예를 들어 로그인 상태를 생각해보자.

브라우저가 사용자를 “기억”하는 것처럼 보인다.

하지만 실제로는 보통 이런 구조다.

```text
request
→ cookie/session token 읽기
→ session store 조회
→ user row 조회
→ req.user에 붙임
→ controller가 req.user를 보고 응답
```

controller가 사용자를 신비롭게 기억하는 것이 아니다.

runtime이 매 request마다 상태를 다시 가져와서 request context에 붙인다.

LLM memory도 비슷하다.

```text
user prompt
→ 최근 대화 일부 선택
→ 장기 기억 검색
→ 관련 프로젝트 문서 선택
→ 요약 또는 원문 일부 조립
→ model context에 넣음
→ 모델이 그 context를 보고 답변
```

모델이 “기억한다”기보다, 하네스가 기억처럼 보이는 입력을 만들어준다.

이 관점이 잡히면 memory는 더 이상 마법이 아니다.

```text
어디에 저장할까?
무엇을 저장할까?
언제 검색할까?
무엇을 context에 넣을까?
오래된 기억은 어떻게 다룰까?
민감한 정보는 어떻게 막을까?
```

이런 평범하지만 중요한 시스템 설계 문제가 된다.

## 사용자의 Telegram + Node LLM 서버 경험으로 다시 보기

사용자는 예전에 Telegram으로 질문하거나 일정 등록을 요청하면, Node 서버가 받아서 LLM에 넘기는 시스템을 만들었다.

그 구조는 대략 이랬다.

```text
Telegram message 수신
→ Node server가 webhook/request 처리
→ 최근 대화 몇 건 선택
→ 장기 기억 후보 검색
→ 적합도 비교
→ 필요한 기억 파일을 tool로 읽음
→ tool schema와 호출 지침도 context에 포함
→ 현재 질문과 함께 LLM 호출
→ 답변 또는 일정 등록 action 수행
```

이건 이미 작은 agent harness다.

특히 memory 관점에서 아주 중요한 판단이 들어 있다.

```text
최근 대화 전체를 다 넣을 수 없음
→ 몇 건만 선택해야 함

장기 기억 전체를 다 넣을 수 없음
→ 관련 있는 것만 검색해야 함

기억 파일은 모델 안에 없음
→ tool 또는 서버 코드가 읽어와야 함

툴 사용법도 모델이 선천적으로 아는 게 아님
→ tool schema와 사용 지침을 context에 넣어야 함
```

즉 사용자가 체감한 “모델에는 자체 기억이 없다”는 감각은 맞다.

정확히는 이렇게 말할 수 있다.

```text
모델 호출 자체는 context-bound다.
기억처럼 보이는 것은 호출 전에 하네스가 조립한 입력이다.
```

이 장은 그 감각을 좀 더 체계화한다.

## Memory와 context window

모든 것을 context에 넣으면 가장 간단할 것 같지만, 그럴 수 없다.

모델에는 context window가 있다.

Codex 공식 문서도 thread의 모든 정보는 모델의 context window 안에 들어가야 한다고 설명한다. 긴 작업에서는 Codex가 자동으로 context를 compact할 수 있다. 즉 중요한 정보를 요약하고, 덜 중요한 정보를 버리면서 긴 작업을 이어간다.

이 말은 memory 설계에도 그대로 적용된다.

```text
저장된 기억이 많다
≠ 다 넣을 수 있다

다 넣을 수 없다
→ 고르고, 줄이고, 우선순위를 정해야 한다
```

memory design의 핵심은 저장보다 선택이다.

데이터베이스에 수천 개의 memory row를 저장하는 것은 쉽다. 어려운 것은 지금 질문에 맞는 5개를 고르는 것이다.

검색엔진과 비슷하다.

```text
indexing
→ 정보를 저장하고 검색 가능하게 만듦

retrieval
→ 지금 query와 관련 있는 후보를 찾음

ranking
→ 후보 중 context에 넣을 우선순위를 정함

compression
→ 너무 긴 내용을 요약하거나 잘라냄

injection
→ 최종 model context에 넣음
```

좋은 memory system은 “많이 저장하는 시스템”이 아니라 “필요한 순간에 필요한 만큼만 가져오는 시스템”이다.

## Memory의 네 가지 층

실전에서는 memory를 하나로 뭉뚱그리면 안 된다.

적어도 네 층으로 나눠서 생각하는 편이 좋다.

```text
1. recent history
2. thread summary / compaction
3. durable memory
4. external source retrieval
```

### 1. Recent history

가장 단순한 memory는 최근 대화다.

```text
최근 user message
최근 assistant answer
최근 tool call
최근 tool output
```

이건 정확도가 높다. 방금 일어난 일이기 때문이다.

하지만 비싸다. 원문 대화와 tool output은 금방 길어진다.

최근 history를 너무 많이 넣으면 context가 지저분해진다.

```text
장점: 최신, 구체적, 흐름 유지에 좋음
단점: 길어짐, 노이즈 많음, 오래된 중요한 정보가 밀릴 수 있음
```

### 2. Thread summary / compaction

긴 thread에서는 모든 대화를 계속 원문으로 들고 갈 수 없다.

그래서 요약이 필요하다.

```text
원문 대화 100개
→ 현재 목표, 결정사항, 변경 파일, 남은 작업만 요약
→ 다음 turn context에 넣음
```

Codex의 compaction도 이런 계열로 이해할 수 있다.

compaction은 단순 압축이 아니다.

좋은 compaction은 정보를 줄이되, 작업을 이어가는 데 필요한 결정과 상태는 남긴다.

나쁜 compaction은 분위기만 남기고 근거를 날린다.

```text
나쁜 요약:
사용자는 메모리 설계에 관심이 있음.

좋은 요약:
사용자는 Telegram+Node LLM 서버에서 최근 대화 N건과 장기 기억 후보를 선택해 context를 조립한 경험이 있음. 8장에서는 이 경험을 memory design 예시로 연결하기로 함.
```

둘 다 짧지만 품질이 다르다.

### 3. Durable memory

Durable memory는 thread를 넘어 이어지는 기억이다.

예:

```text
사용자는 한국어 설명을 선호한다.
이 repo는 intermediate 웹 개발자용 개론서 톤으로 쓴다.
피드백은 정확성 위주로 반영하고, 지나친 초심자용 축약은 피한다.
이 프로젝트의 repo path는 ...
```

Codex 공식 문서 기준으로도 Memories는 이전 thread의 유용한 context를 미래 작업으로 가져오기 위한 기능이다. 선호, 반복 workflow, tech stack, project convention, known pitfall 같은 stable context가 대상이다.

여기서 중요한 단어는 stable이다.

모든 것을 durable memory로 만들면 안 된다.

```text
좋은 durable memory:
사용자가 반복해서 선호하는 작업 방식
프로젝트에서 계속 유지되는 규칙
자주 틀리는 경로 또는 setup 주의점

나쁜 durable memory:
오늘만 쓰는 임시 파일명
이미 끝난 일회성 TODO
검증 안 된 추측
민감한 secret
오래되면 위험한 상태
```

Durable memory는 강력하지만 위험하다.

오래된 memory는 stale해진다.

한 번 잘못 저장된 memory는 이후 thread에서 계속 잘못된 가정으로 돌아올 수 있다.

### 4. External source retrieval

때로는 memory보다 원본 source가 낫다.

예:

```text
GitHub issue
Slack thread
Google Doc
Notion spec
local source file
database row
official documentation
```

memory는 요약이고, source는 근거다.

요약은 빠르지만 왜곡될 수 있다.

원본은 길지만 정확하다.

좋은 하네스는 둘을 구분한다.

```text
memory
→ "지난번에 이 API는 billing v2를 쓴다고 했음"

source retrieval
→ 실제 billing spec 문서 열기
→ 현재 API field 확인
→ 그 결과를 context에 넣기
```

고위험 정보일수록 memory만 믿으면 안 된다.

법, 가격, API spec, 보안 정책, 최신 제품 동작은 원본을 확인해야 한다.

External source retrieval은 엄밀히 말하면 durable memory 자체라기보다 memory system이 함께 사용하는 source-of-truth retrieval layer다. 하지만 최종적으로 retrieved source snippet도 context assembly에 참여하므로 memory design과 함께 설계해야 한다.

## Memory pipeline: 저장보다 중요한 것은 선택

memory system은 보통 이런 pipeline으로 생각할 수 있다.

```text
capture
→ 무엇을 기억 후보로 잡을까?

extract
→ 원문에서 durable fact를 뽑을까?

store
→ 어디에 저장할까?

retrieve
→ 지금 질문과 관련 있는 후보를 어떻게 찾을까?

rank
→ 무엇을 context에 먼저 넣을까?

inject
→ 어떤 형식으로 모델에게 보여줄까?

verify
→ 이 기억을 믿어도 될까?

expire/update
→ 오래된 기억을 어떻게 폐기하거나 갱신할까?
```

웹 개발자에게는 캐시 설계와 닮았다.

캐시도 그냥 저장하면 끝이 아니다.

```text
cache key
TTL
invalidation
source of truth
stale read
write-through / write-back
manual purge
```

memory도 같다.

```text
memory key
relevance score
freshness
source evidence
scope
privacy
manual review
```

“기억을 많이 넣으면 좋아진다”는 생각은 캐시에 TTL 없이 모든 것을 영원히 넣는 것과 비슷하다. 처음엔 편해 보이지만 나중엔 stale bug가 된다.

## Memory에 들어가면 좋은 것과 나쁜 것

memory 후보를 판단할 때는 이 질문을 한다.

```text
미래 thread에서 다시 쓸 가능성이 높은가?
상대적으로 오래 안정적인가?
사용자가 반복해서 말하지 않아도 되면 이득인가?
틀리면 피해가 큰가?
원본 source를 다시 확인해야 하는 종류인가?
민감한 정보인가?
```

### 좋은 memory 후보

```text
사용자의 선호:
한국어로 설명하되 기술 용어는 정확히 유지한다.

프로젝트 톤:
이 repo는 intermediate 웹 개발자용 개론서다. 과도한 초심자용 축약은 피한다.

반복 workflow:
각 장 작성 후 링크 검증, commit, push를 한다.

프로젝트 구조:
새 원고는 docs/에 두고 README 목차에 연결한다.

자주 틀리는 점:
서브에이전트에 전체 대화 context를 넘기면 PM처럼 행동할 수 있으니, 이 프로젝트에서는 직접 작성한다.
```

### 나쁜 memory 후보

```text
오늘만 필요한 임시 판단:
방금 7장 피드백을 기다리는 중.

검증 안 된 추측:
Caveman에는 hook 코드가 전혀 없다.

민감 정보:
API key, access token, private customer data.

자주 바뀌는 정보:
현재 최신 모델, 가격, 정책, 제품 UI 위치.

원본 확인이 필수인 정보:
법률, 보안 정책, 결제 조건, public API schema.
```

좋은 memory는 다음 thread의 friction을 줄인다.

나쁜 memory는 다음 thread의 hallucination을 강화한다.

## Stale memory: 오래된 기억은 버그다

memory에서 가장 위험한 것은 “틀린 기억”보다 “한때 맞았던 기억”이다.

예를 들어:

```text
이 프로젝트의 7장은 아직 작성 전이다.
```

이 문장은 어제는 맞았을 수 있다.

하지만 오늘 7장이 작성되어 push되었다면 틀렸다.

memory system이 이걸 계속 가져오면 모델은 과거 상태에서 작업한다.

웹 개발로 치면 stale cache다.

```text
DB에는 주문 상태가 shipped
cache에는 주문 상태가 pending
controller는 cache를 믿고 잘못 응답
```

memory도 source of truth가 필요하다.

```text
repo 상태
→ git status, file system, GitHub

공식 제품 동작
→ official docs/manual

사용자 선호
→ 최근 명시적 지시

작업 진행 상태
→ 현재 파일과 commit log
```

그래서 하네스는 memory를 “정답”으로 넣으면 안 된다.

더 안전한 형태는 이렇다.

```text
Memory says: 7장은 미작성일 수 있음.
Before acting: check docs/07-caveman-case-study.md and git log.
```

좋은 memory는 model에게 결론만 주는 것이 아니라 확인할 경로를 준다.

## Source discipline: 기억과 근거를 분리하기

memory design에서 가장 좋은 습관은 “기억”과 “근거”를 분리하는 것이다.

나쁜 형태:

```text
사용자는 항상 subagent를 싫어한다.
```

좋은 형태:

```text
이 repo 작업에서는 subagent가 7장 작성 지시를 잘못 해석한 적이 있다.
다음 장 작성은 root agent가 직접 수행하는 편이 안전하다.
근거: 7장 작업 중 subagent가 6장 보정 흐름을 반복하려고 했음.
```

차이는 크다.

첫 번째는 성급한 일반화다.

두 번째는 scope와 근거가 있다.

좋은 memory는 보통 이런 형태를 가진다.

```text
fact
scope
source/evidence
last verified
confidence
expiry or refresh rule
```

예:

```text
fact: 이 repo의 새 원고는 docs/에 작성한다.
scope: harness-engineering-study repo
source: docs/00-writing-plan.md
last verified: 2026-06-29
confidence: high
refresh: 새 장 작성 전 writing plan 확인
```

이 정도 metadata가 있으면 memory가 훨씬 덜 위험해진다.

## Codex에서는 어떻게 보이는가

Codex 공식 문서 기준으로 Memories는 기본적으로 꺼져 있을 수 있고, 설정에서 켜거나 config feature flag로 켤 수 있다.

Memory가 켜지면 Codex는 이전 thread에서 유용한 context를 local memory file로 만들 수 있다. 대상은 stable preference, recurring workflow, tech stack, project convention, known pitfall 같은 것들이다.

중요한 공식적 경고도 있다.

```text
항상 적용되어야 하는 team guidance는 AGENTS.md나 checked-in documentation에 둔다.
Memories는 helpful local recall layer이지, 반드시 적용되어야 하는 rule의 유일한 source가 아니다.
```

이 구분은 아주 중요하다.

```text
AGENTS.md
→ repo와 함께 version control되는 규칙

Memory
→ local user/Codex home 아래 저장되는 recall layer

Hook
→ lifecycle에서 실행되는 command

Skill
→ reusable workflow
```

Codex memory 파일은 Codex home 아래에 저장된다. 기본값은 `~/.codex`이고, memory 관련 파일은 `~/.codex/memories/` 아래에 있다.

이 파일들은 generated state다. 필요하면 문제 해결이나 공유 전 검토를 위해 볼 수 있지만, 수동 편집을 primary control surface로 삼는 것은 권장되지 않는다.

Codex memory generation은 즉시 동기적으로 일어나는 저장 버튼이 아니다. 공식 문서 기준으로 Codex는 active 또는 short-lived session을 건너뛸 수 있고, thread가 충분히 idle해질 때까지 기다릴 수 있으며, rate limit이 낮으면 background memory generation을 스킵할 수 있다. generated memory fields에서는 secret을 redact하지만, Codex home이나 memory artifact를 공유하기 전에는 사용자가 직접 검토해야 한다.

Codex에는 thread 단위 memory control도 있다. `/memories`를 통해 현재 thread가 기존 memory를 사용할지, 나중에 memory generation input으로 쓰일지 조절할 수 있다.

이것도 하네스 관점에서는 자연스럽다.

```text
global setting
→ memory 기능 자체를 켤지

thread-level setting
→ 이 thread에서 memory를 쓰거나 생성할지

memory files
→ local recall data

context assembly
→ 관련 memory가 선택되어 model context에 들어감
```

## Chronicle은 무엇이 다른가

Codex 문서에는 Chronicle도 나온다.

Chronicle은 screen context를 이용해 memory를 보강하는 기능이다. macOS의 Screen Recording/Accessibility permission을 요구하고, 현재는 opt-in research preview로 설명된다.

Chronicle의 세부 동작과 지원 범위는 research preview 상태와 제품 표면에 따라 달라질 수 있다. 여기서는 memory source가 thread를 넘어 screen context까지 확장될 수 있다는 설계 포인트만 본다.

이 기능은 좋은 예시다.

왜냐하면 memory source가 대화 thread만이 아니라 화면 context까지 확장되기 때문이다.

```text
일반 memory
→ 이전 thread에서 나온 유용한 context

Chronicle
→ 최근 화면 context를 바탕으로 memory 생성
```

하지만 이만큼 privacy/security risk도 커진다.

화면에는 secret, 고객 정보, private 문서, 회의 내용, 채팅이 있을 수 있다.

그래서 Chronicle은 권한, opt-in, pause, local storage, rate limit 같은 운영 문제가 함께 따라온다.

이 사례가 보여주는 것은 하나다.

```text
memory source가 강해질수록,
privacy와 source discipline도 같이 강해져야 한다.
```

## 작은 예시: memory retrieval pseudo-code

아주 단순한 memory retrieval을 pseudo-code로 써보자.

```ts
type Memory = {
  id: string;
  text: string;
  scope: 'global' | 'repo' | 'thread';
  source: string;
  updatedAt: string;
  confidence: 'low' | 'medium' | 'high';
};

async function buildContext(userPrompt: string, repoPath: string) {
  const recent = await loadRecentMessages({ limit: 8 });
  const candidates = await searchMemories(userPrompt, { repoPath });
  const ranked = rankByRelevanceAndFreshness(candidates);

  const selected = ranked
    .filter(memory => memory.confidence !== 'low')
    .slice(0, 5);

  return {
    messages: [
      ...systemAndDeveloperInstructions(),
      ...formatMemories(selected),
      ...formatRecentHistory(recent),
      { role: 'user', content: userPrompt },
    ],
  };
}
```

이 예시는 일부러 단순하다.

실제 API나 제품에서는 memory가 반드시 독립된 chat message로 들어가는 것은 아니다. developer context, retrieved context block, tool output, server-side conversation state, compressed summary 등 여러 형태로 들어갈 수 있다. 여기서는 개념을 보여주기 위해 `messages` 배열처럼 표현했다.

실제 시스템에서는 더 많은 판단이 필요하다.

```text
memory가 너무 오래되었는가?
현재 repo와 scope가 맞는가?
원본 source를 다시 확인해야 하는가?
secret이 포함되어 있지 않은가?
같은 memory가 여러 번 중복되지 않는가?
현재 user prompt와 정말 관련 있는가?
```

그래도 핵심은 보인다.

```text
memory file 자체가 답변을 만들지 않는다.
memory retrieval이 context를 만들고,
모델은 그 context를 읽는다.
```

## 흔한 오해

### 오해 1. “Memory가 켜져 있으면 모델이 다 기억한다”

아니다.

Memory가 켜져 있어도 모든 과거가 항상 context에 들어가는 것은 아니다.

저장, 검색, 선택, 주입 과정이 있다.

모델은 최종 context에 들어온 것만 본다.

### 오해 2. “Memory는 AGENTS.md를 대체한다”

아니다.

항상 지켜야 하는 repo/team rule은 `AGENTS.md`나 checked-in documentation에 둬야 한다.

Memory는 local recall layer다.

```text
AGENTS.md
→ durable, version-controlled, team-visible guidance

Memory
→ local, generated, selective recall
```

둘은 경쟁이 아니라 역할이 다르다.

### 오해 3. “Memory는 많을수록 좋다”

아니다.

memory가 많아질수록 retrieval, ranking, stale filtering이 중요해진다.

쓸모없는 memory는 context noise가 된다.

틀린 memory는 hallucination seed가 된다.

### 오해 4. “요약은 원문과 같다”

아니다.

요약은 압축된 해석이다.

요약에는 누락과 왜곡이 생긴다.

그래서 중요한 판단은 가능하면 source를 다시 확인해야 한다.

### 오해 5. “Vector DB를 붙이면 memory 문제가 해결된다”

아니다.

Vector DB는 retrieval 도구일 뿐이다.

memory design에는 여전히 다음 문제가 남는다.

```text
무엇을 저장할까?
어떤 scope로 저장할까?
언제 stale하다고 볼까?
어떤 source를 근거로 삼을까?
얼마나 context에 넣을까?
민감한 정보는 어떻게 막을까?
```

도구가 아니라 정책이 핵심이다.

## 왜 중요한가

Memory design을 이해하면 agent가 훨씬 덜 신비롭게 보인다.

```text
모델이 기억한다
```

가 아니라:

```text
하네스가 기억 후보를 저장한다.
하네스가 현재 prompt에 맞는 기억을 찾는다.
하네스가 일부를 context에 넣는다.
모델이 그 context를 보고 답한다.
```

이렇게 보면 설계할 수 있다.

그리고 설계할 수 있으면 고칠 수 있다.

```text
모델이 자꾸 예전 결정을 믿는다
→ stale memory 문제일 수 있음

모델이 매번 같은 설명을 요구한다
→ durable memory 후보가 빠졌을 수 있음

모델이 엉뚱한 프로젝트 규칙을 적용한다
→ scope/ranking 문제일 수 있음

모델이 민감 정보를 떠올린다
→ capture/redaction/privacy 문제일 수 있음

모델이 공식 API를 틀린다
→ memory가 아니라 source retrieval을 해야 할 문제일 수 있음
```

이것이 하네스 엔지니어링이다.

모델을 탓하기 전에 context assembly를 본다.

## 요약

```text
1. Memory는 모델 내부 기억이 아니라 하네스의 저장/검색/주입 설계다.
2. 모델은 최종 context에 들어온 정보만 본다.
3. recent history, thread summary, durable memory, external retrieval은 역할이 다르다.
4. context window 때문에 모든 memory를 넣을 수 없고, 선택과 압축이 필요하다.
5. 좋은 memory는 stable preference, recurring workflow, project convention, known pitfall이다.
6. 나쁜 memory는 stale fact, secret, 일회성 상태, 검증 안 된 추측이다.
7. AGENTS.md는 team-visible durable guidance이고, Memory는 local recall layer다.
8. Memory는 source of truth를 대체하지 않는다. 중요한 정보일수록 memory보다 source를 확인해야 한다.
```

## 생각해볼 질문

1. 내가 “기억”이라고 부르는 정보는 recent history, summary, durable memory, external source 중 어디에 가까운가?
2. 이 정보는 미래 thread에서도 안정적으로 맞을까, 아니면 금방 stale해질까?
3. 이 memory가 틀렸을 때 피해가 큰가?
4. memory만 넣어도 되는가, 아니면 원본 source를 다시 확인해야 하는가?
5. 내 agent harness는 memory의 scope, freshness, source evidence를 어떻게 기록할 것인가?
