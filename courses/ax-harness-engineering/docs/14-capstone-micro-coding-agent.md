[← README](../README.md)

# 14장. Capstone 2: micro coding agent 설계하기

## 이 장에서 할 일

13장에서는 `agent-tool-lint`를 만들기로 했다.

그 도구는 agent가 외부 세계를 만지는 손잡이를 검사한다.

이번 장은 그 다음 도전이다.

```text
micro coding agent
```

한 줄로 말하면:

```text
작은 모델에게 큰 코딩을 시키지 않고,
harness가 일을 작게 쪼개고,
검증 가능한 단위로만 코딩하게 만드는 coding agent.
```

여기서 중요한 단어는 `micro`다.

이 장의 목표는 “대형 모델처럼 알아서 다 하는 코딩 에이전트”를 만드는 것이 아니다.

작은 모델이 망하기 쉬운 부분을 인정하고, runtime이 그 약점을 감싸는 구조를 설계하는 것이다.

## 왜 이 과제를 하는가

코딩 에이전트는 멋있어 보인다.

사용자가 말한다.

```text
이 버그 고쳐줘.
```

agent가 repo를 읽고, 원인을 찾고, patch를 만들고, test를 돌리고, 결과를 보고한다.

겉으로 보면 간단하다.

하지만 실제로는 여러 일이 한꺼번에 섞여 있다.

```text
요구사항 이해
repo 탐색
관련 파일 선택
원인 추론
수정 범위 결정
patch 작성
test 선택
test 실행
실패 해석
재시도
작업 기록
```

대형 모델도 여기서 자주 넘어진다.

작은 모델은 더 쉽게 넘어진다.

특히 이런 곳에서 무너진다.

```text
긴 context에서 핵심 파일을 놓침
불필요한 파일을 많이 고침
큰 refactor를 시작함
test 없이 자신감 있게 끝냄
실패 로그를 보고 엉뚱한 곳을 고침
이전 판단을 잊고 같은 실수를 반복함
```

그래서 micro coding agent의 핵심은 모델을 더 믿는 것이 아니다.

모델이 한 번에 해야 하는 일을 줄이는 것이다.

```text
큰 문제 하나
→ 작은 ticket 여러 개

넓은 repo context
→ 필요한 파일 몇 개

큰 patch
→ 함수 하나 또는 파일 하나

막연한 성공
→ test와 trace
```

이건 1~12장에서 배운 하네스 관점을 그대로 적용하는 과제다.

## 이 과제가 연결하는 장

```text
1장
→ 모델은 기억하지 않고 context를 읽는다.

2장
→ agent는 LLM이 아니라 runtime loop다.

3장
→ 답변 전 context assembly가 있다.

8장
→ memory는 retrieval/injection pipeline이다.

9장
→ tool은 모델에게 주는 API 문서다.

11장
→ agent는 eval로 판단해야 한다.

12장
→ 위험한 행동은 boundary로 막아야 한다.
```

micro coding agent는 이 모든 것을 작게 묶는다.

특히 중요한 문장은 이것이다.

```text
작은 모델의 지능을 키우는 것이 아니라,
작은 모델이 틀릴 수 있는 자유도를 줄인다.
```

## 만들 것

처음 제품 형태는 CLI 또는 local harness다.

명령은 이런 느낌이면 충분하다.

```bash
micro-coder run ./fixtures/bug-001
```

또는:

```bash
micro-coder run --repo ./sample-project --task tasks/bug-001.md
```

입력:

```text
작은 repo
task description
allowed files
test command
model command 또는 model API 설정
```

출력:

```text
작업 plan
선택한 context pack
생성한 patch
실행한 test
성공/실패 결과
trace report
```

중요한 것은 “코드를 고쳤다”가 아니다.

중요한 것은 왜 그렇게 고쳤는지 관찰 가능해야 한다는 것이다.

```text
무슨 파일을 봤는가?
왜 그 파일을 골랐는가?
수정 범위는 어디까지였는가?
test는 무엇을 돌렸는가?
실패하면 다음 판단은 무엇이었는가?
```

## 제품이 아니라 실험 장치

이 과제는 처음부터 실사용 IDE agent를 만드는 일이 아니다.

처음 목표는 실험 장치다.

```text
작은 모델에게 어떤 제약을 걸면 코딩 성공률이 올라가는가?
```

이 질문에 답하는 장치면 된다.

그래서 v0.1은 멋진 UI가 없어도 된다.

VSCode extension도 필요 없다.

멀티 repo 지원도 필요 없다.

처음에는 fixture repo 몇 개와 CLI report면 충분하다.

## 작은 모델용 설계 원칙

작은 모델에게 큰 일을 맡기면 망한다.

그래서 harness가 이런 제한을 건다.

```text
한 번에 한 ticket
한 ticket은 한 파일 또는 한 함수 중심
patch 크기 제한
새 dependency 금지
큰 refactor 금지
파일 삭제 금지
test 없는 완료 금지
불확실하면 coding이 아니라 investigation으로 전환
```

이 제한은 모델을 괴롭히기 위한 것이 아니다.

성공 가능한 작업 단위로 줄이기 위한 것이다.

웹 개발로 비유하면:

```text
주니어에게 "결제 시스템 개선해줘"라고 맡기지 않는다.

"PaymentForm.tsx의 amount validation이 음수를 막지 못한다.
이 함수만 보고 failing test를 통과시켜라."

이렇게 ticket을 자른다.
```

micro coding agent도 같다.

모델에게 “repo 전체를 이해해”라고 하지 않는다.

하네스가 먼저 일감을 잘라준다.

단, v0.1에서는 이 “일감 자르기”도 자동화하지 않는다.

처음에는 사람이 자른 `task.md`를 입력으로 받는다.

자동 decomposition은 나중 실험이다.

## 기본 loop

v0.1의 loop는 이 정도면 된다.

```text
1. task.md 읽기
2. allowed files 확인
3. repo scan 보조 정보 만들기
4. local context pack 만들기
5. model patch proposal 받기
6. patch validation
7. patch apply
8. test run
9. trace write
10. stop or retry
```

flow로 보면:

```text
human-written task.md
  ↓
problem card
  ↓
small work item
  ↓
repo scan helper
  ↓
context pack
  ↓
model generates patch
  ↓
apply patch in sandbox
  ↓
run test
  ↓
trace result
```

이 loop에서 모델은 모든 것을 결정하지 않는다.

harness가 결정하는 것이 많다.

```text
어떤 파일까지 볼 수 있는가
patch가 얼마나 클 수 있는가
test 없이 끝낼 수 있는가
실패하면 몇 번 재시도할 수 있는가
destructive command를 실행할 수 있는가
```

이게 agent harness다.

중요한 제한:

```text
v0.1에서는 task decomposition을 자동화하지 않는다.
사람이 작성한 task.md와 allowedFiles를 입력으로 받는다.
harness는 context pack 생성, patch validation, test run, trace 기록에 집중한다.
자동 decomposition은 v0.3 이후의 실험이다.
```

## problem definition

사용자 요청은 대개 흐릿하다.

예:

```text
로그인이 가끔 안 돼. 고쳐줘.
```

이걸 그대로 작은 모델에게 주면 넓다.

먼저 problem card로 바꾼다.

```md
# Problem

Login fails when password contains leading or trailing spaces.

## Expected behavior

Password should be submitted exactly as typed.

## Current behavior

The login helper trims all string fields before submitting.

## Success condition

The failing test `auth.test.ts > preserves password whitespace` passes.

## Constraints

- Touch only `src/auth/normalizeLoginInput.ts`.
- Do not change API types.
- Do not add dependencies.
- Add or update one focused test.
```

problem card는 모델에게 주는 “작은 전장”이다.

전장을 좁히면 작은 모델도 싸울 수 있다.

## repo scan

repo scan은 처음부터 AI가 다 할 필요 없다.

v0.1에서는 단순한 정적 탐색이면 충분하다.

```text
파일명 검색
symbol grep
test 파일 검색
package script 확인
최근 error log에서 path 추출
```

하지만 v0.1의 repo scan은 allowedFiles를 자동 결정하지 않는다.

allowedFiles는 `task.md`가 제공한다.

repo scan은 보조 역할만 한다.

```text
allowedFiles 안의 snippet 추출
symbol 존재 확인
test 파일 존재 확인
test command 확인
```

예:

```bash
rg "normalizeLoginInput|login" src test
```

scan 결과는 모델에게 raw dump로 다 주지 않는다.

요약해서 준다.

````md
## Repo scan result

Likely relevant files:

1. `src/auth/normalizeLoginInput.ts`
   - contains `normalizeLoginInput`
   - trims all string fields

2. `test/auth.test.ts`
   - contains login normalization tests
   - missing whitespace preservation case

Suggested test command:

```bash
npm test -- auth.test.ts
```
````

작은 모델에게 repo 전체를 먹이는 것은 비싸고 위험하다.

필요한 조각만 context pack으로 만든다.

## task decomposition

큰 요구사항은 작은 work item으로 쪼갠다.

하지만 v0.1에서는 이 단계를 harness가 자동으로 하지 않는다.

fixture 작성자 또는 사람이 미리 쪼개둔다.

나쁜 work item:

```text
로그인 버그 고치기
```

좋은 work item:

```text
`normalizeLoginInput`이 password field를 trim하지 않도록 수정하고,
기존 username trim 동작은 유지하는 test를 추가하라.
```

work item은 다음 필드를 가진다.

```ts
type WorkItem = {
  id: string;
  title: string;
  hypothesis: string;
  allowedFiles: string[];
  forbiddenActions: string[];
  successCommand: string;
  maxPatchLines: number;
};
```

v0.1에서는 이 type을 코드로 꼭 만들 필요는 없다.

markdown 파일이어도 된다.

`hypothesis`도 모델이 자동 추론한 원인이 아니다.

v0.1에서는 fixture 작성자가 제공한 작업 가설이다.

자동 원인 추론은 나중 실험으로 미룬다.

중요한 것은 구조다.

## local context pack

context pack은 모델에게 넘기는 작은 작업 폴더다.

구성:

```text
problem card
work item
관련 source snippet
관련 test snippet
수정 제약
출력 형식
```

예:

````md
# Context Pack

## Task

Fix password whitespace normalization.

## Allowed files

- `src/auth/normalizeLoginInput.ts`
- `test/auth.test.ts`

## Source

```ts
export function normalizeLoginInput(input: LoginInput): LoginInput {
  return Object.fromEntries(
    Object.entries(input).map(([key, value]) => [
      key,
      typeof value === "string" ? value.trim() : value
    ])
  ) as LoginInput;
}
```

## Test command

```bash
npm test -- auth.test.ts
```

## Rules

- Keep username trim behavior.
- Do not trim password.
- Do not add dependencies.
- Return a unified diff only.
````

작은 모델에게 필요한 것은 “많은 정보”가 아니다.

정확히 필요한 정보다.

## patch proposal

모델에게 자유 서술을 시키지 않는다.

출력 형식을 제한한다.

```text
Return only a unified diff.
Do not explain.
Do not edit files outside allowedFiles.
Do not add dependencies.
```

왜 설명을 줄이는가?

작은 모델은 설명하다가 자기 patch와 모순되는 말을 만들 수 있다.

처음에는 patch만 받는 편이 낫다.

설명은 trace에서 harness가 기록한다.

```text
input context hash
model output
patch apply result
test result
```

## patch validation

patch를 바로 믿지 않는다.

apply 전에 검사한다.

```text
allowedFiles 밖 파일을 건드렸는가?
maxPatchLines를 넘었는가?
package.json을 수정했는가?
lockfile을 수정했는가?
delete file이 있는가?
binary file이 있는가?
```

v0.1에서는 단순 규칙이면 충분하다.

```text
허용 파일 밖 수정 → reject
patch line 수 초과 → reject
dependency 추가 → reject
```

작은 모델에게 “하지 마”라고 말하는 것보다, runtime에서 막는 것이 안전하다.

단, patch validation은 correctness를 보장하지 않는다.

이건 boundary check다.

test가 부족하면 잘못된 patch도 통과할 수 있다.

## test run

코딩 agent는 test 없이 완료하면 안 된다.

v0.1에서는 하나의 test command만 있어도 된다.

```bash
npm test -- auth.test.ts
```

test 결과는 trace에 남긴다.

```json
{
  "workItem": "bug-001-password-whitespace",
  "testCommand": "npm test -- auth.test.ts",
  "exitCode": 0,
  "passed": true
}
```

실패하면 모델에게 전체 로그를 다 주지 않는다.

처음에는 tail과 핵심 error만 준다.

```text
test failed
error summary
relevant stack line
previous patch
allowed files
```

로그도 context다.

너무 많이 주면 모델은 또 헤맨다.

## retry policy

무한 재시도는 안 된다.

v0.1:

```text
maxAttempts = 2
```

1회 실패 후에는 repair context를 만든다.

```md
# Repair Context

Previous patch failed.

## Error

Expected password to preserve whitespace.
Received password was trimmed.

## Constraint

Do not change username behavior.
Touch only allowed files.
Return unified diff only.
```

2회 실패하면 멈춘다.

실패도 결과다.

```text
이 모델은 이 fixture를 해결하지 못했다.
실패 지점은 patch generation이었다.
context pack은 충분했는가?
test error는 명확했는가?
```

이 질문을 남기는 것이 실험이다.

## trace report

micro coding agent의 산출물은 patch만이 아니다.

trace가 더 중요하다.

예:

```json
{
  "taskId": "bug-001-password-whitespace",
  "model": "local-small-coder",
  "modelCommand": "ollama run local-small-coder",
  "temperature": 0.2,
  "promptVersion": "micro-coder-v0.1",
  "contextPackHash": "sha256:abc123...",
  "fixtureVersion": "bug-001-v1",
  "attempts": 2,
  "result": "failed",
  "failureStage": "patch_generation",
  "filesProvided": [
    "src/auth/normalizeLoginInput.ts",
    "test/auth.test.ts"
  ],
  "filesChanged": [
    "src/auth/normalizeLoginInput.ts"
  ],
  "patchLines": 14,
  "testCommand": "npm test -- auth.test.ts",
  "notes": [
    "model changed username behavior despite constraint"
  ]
}
```

이 trace가 있어야 eval을 할 수 있다.

```text
모델이 멍청했다
```

이 말은 분석이 아니다.

더 좋은 분석은 이것이다.

```text
repo scan은 맞았다.
context pack도 관련 파일을 포함했다.
하지만 patch generation에서 constraint를 어겼다.
따라서 다음 실험은 output validation 또는 smaller work item이 필요하다.
```

## MVP 범위

v0.1은 작게 간다.

입력:

```text
fixture repo
task markdown
test command
allowed files
model command
```

기능:

```text
task 읽기
allowedFiles 확인
repo scan 보조 정보 만들기
context pack 만들기
model 호출하기
unified diff 받기
patch validate하기
patch apply하기
test 실행하기
trace JSON 쓰기
```

하지 않을 것:

```text
IDE integration
multi-repo support
parallel agents
long-term memory
autonomous backlog handling
automatic dependency install
production sandbox
web dashboard
automatic task decomposition
automatic relevant file selection
model-driven tool orchestration
```

v0.1은 실험 장치다.

멋진 제품이 아니다.

## fixture 설계

처음 fixture는 작아야 한다.

```text
fixtures/
  js-utils/
    bug-001-password-whitespace/
    bug-002-date-timezone/
    bug-003-array-empty-case/
    bug-004-url-validation/
    bug-005-error-message/
```

각 fixture는 같은 구조를 가진다.

```text
README.md
task.md
repo/
expected.patch
expected-trace.json
```

task 예:

````md
# bug-003-array-empty-case

## Problem

`average([])` returns `NaN`.

## Expected

`average([])` should return `0`.

## Allowed files

- `src/math.ts`
- `test/math.test.ts`

## Test command

```bash
npm test -- math.test.ts
```
````

이런 작은 bug 5개면 충분하다.

처음부터 real-world repo를 넣으면 변수가 너무 많다.

## eval 기준

성공 기준은 “한 번 성공했다”가 아니다.

최소 기준:

```text
5개 fixture 중 4개 이상 해결
허용 파일 밖 수정 0회
dependency 추가 0회
test 없이 완료 0회
maxPatchLines 초과 0회
실패한 fixture는 failureStage가 기록됨
```

비교 실험도 한다.

```text
baseline
→ 전체 task + 관련 파일 dump를 모델에게 한 번에 줌

micro harness
→ problem card + work item + context pack + patch validation + test
```

보고 싶은 것은 이것이다.

```text
micro harness가 성공률을 올리는가?
아니면 단지 더 느리게 실패하는가?
```

둘 다 배움이다.

## 작은 모델 실패 유형

trace에는 실패 유형을 남긴다.

```text
context_selection_failure
→ 필요한 파일을 context pack에 넣지 못함

instruction_following_failure
→ allowed files, output format, dependency 금지를 어김

patch_generation_failure
→ patch 의도는 보이지만 코드 생성 자체가 틀림

diff_format_failure
→ unified diff 형식을 지키지 못함

patch_apply_failure
→ diff처럼 보이지만 실제 apply 실패

test_command_failure
→ 테스트 command 자체가 잘못됐거나 환경 문제

timeout_failure
→ 모델 호출 또는 test run이 시간 초과

no_change_failure
→ patch를 냈지만 실제 변경이 없음

test_reasoning_failure
→ test failure를 잘못 해석함

over_editing_failure
→ 필요 이상으로 큰 수정 또는 refactor를 함

regression_failure
→ targeted test는 통과했지만 기존 test가 깨짐
```

이 분류가 있어야 다음 개선이 보인다.

```text
context_selection_failure가 많다
→ repo scan/context pack 개선

instruction_following_failure가 많다
→ patch validation 강화

test_reasoning_failure가 많다
→ repair context 축소/정리
```

## tool 설계

micro coding agent도 tool이 필요하다.

하지만 v0.1 tool은 적어야 한다.

정확히 말하면, v0.1에서는 모델이 직접 tool call을 고르지 않는다.

모델은 unified diff만 만든다.

`apply_patch`, `run_test`, `write_trace`는 harness가 deterministic하게 실행하는 internal operation이다.

나중에 모델 주도 tool calling으로 확장할 수 있지만, 처음에는 orchestration을 모델에게 맡기지 않는다.

```text
read_file
search_repo
apply_patch
run_test
write_trace
```

각 tool은 경계가 있어야 한다.

```text
read_file
→ allowed root 안에서만 읽기

search_repo
→ repo 안에서만 검색

apply_patch
→ allowedFiles 밖 수정 거부

run_test
→ allowlisted command만 실행

write_trace
→ trace directory에만 쓰기
```

여기서 12장의 safety boundary가 그대로 나온다.

코딩 agent라고 해서 아무 명령이나 실행하면 안 된다.

## command safety

test command도 위험할 수 있다.

처음에는 allowlist만 허용한다.

```text
npm test -- ...
pnpm test -- ...
python -m pytest ...
```

v0.1에서는 다음을 금지한다.

```text
rm
curl
wget
ssh
chmod
sudo
git push
npm install
```

완벽한 sandbox는 v0.1 목표가 아니다.

하지만 위험한 명령을 그냥 실행하지 않는 최소 경계는 필요하다.

## context pack 크기 제한

작은 모델에게 context를 많이 주면 좋을 것 같지만 꼭 그렇지 않다.

작은 모델은 긴 context에서 신호를 잃을 수 있다.

v0.1에서는 단순 제한을 둔다.

```text
maxFiles = 3
maxTotalChars = 12000
maxSnippetCharsPerFile = 5000
```

이 숫자는 정답이 아니다.

실험용 knob다.

trace에 남긴다.

```json
{
  "contextPack": {
    "files": 2,
    "chars": 8430
  }
}
```

나중에 성공률과 비교한다.

## output format

모델 output은 처음부터 단순하게 제한한다.

```text
unified diff only
```

나쁜 출력:

````text
여기 수정하면 됩니다:
```ts
...
```
````

좋은 출력:

```diff
diff --git a/src/math.ts b/src/math.ts
--- a/src/math.ts
+++ b/src/math.ts
@@
-  return sum(values) / values.length;
+  return values.length === 0 ? 0 : sum(values) / values.length;
```

unified diff는 검증하기 쉽다.

patch 적용 실패도 명확하다.

작은 모델은 unified diff 형식을 자주 깨뜨릴 수 있다.

v0.1에서는 이를 자동 repair하지 않는다.

그냥 `diff_format_failure`로 기록한다.

diff repair나 function-body replacement는 v0.2 이후 실험이다.

## 구현 단계

실제 구현할 때는 이 순서로 간다.

```text
1. fixture repo 하나 만들기
2. task.md format 정하기
3. context pack 생성기 만들기
4. model command를 실행하는 wrapper 만들기
5. unified diff 파서/검증 만들기
6. patch apply
7. test command 실행
8. trace JSON 쓰기
9. fixture 5개로 늘리기
10. baseline과 비교하기
```

Ponytail 원칙:

```text
처음부터 agent framework 쓰지 말기.
처음부터 web UI 만들지 말기.
처음부터 multi-model router 만들지 말기.
처음부터 vector DB 붙이지 말기.
```

필요하면 나중에 붙인다.

처음에는 파일, subprocess, JSON이면 된다.

## 완료 체크리스트

```text
- [ ] fixture repo가 있다.
- [ ] task.md를 읽는다.
- [ ] allowedFiles를 지킨다.
- [ ] context pack을 만든다.
- [ ] model command를 호출한다.
- [ ] unified diff만 받는다.
- [ ] patch validation을 한다.
- [ ] patch apply를 한다.
- [ ] test command를 실행한다.
- [ ] trace JSON을 쓴다.
- [ ] 실패 시 failureStage를 기록한다.
- [ ] 5개 fixture 결과를 비교한다.
- [ ] README에 limitation을 쓴다.
```

## README 작성법

README 첫 문장은 이렇게 쓰면 된다.

영어:

```text
A tiny coding-agent harness that helps small models solve code tasks by decomposing work into file-scoped patches, constrained context packs, patch validation, tests, and traces.
```

한국어:

```text
작은 모델이 큰 코딩 작업에서 무너지지 않도록 작업을 파일/함수 단위로 쪼개고, context pack, patch 검증, test, trace로 제한하는 coding-agent harness.
```

README 구조:

```text
1. What is this?
2. Why small models need a harness
3. How the loop works
4. Fixture format
5. Usage
6. Trace example
7. Evaluation results
8. Safety boundaries
9. Limitations
10. Roadmap
```

limitation은 꼭 써야 한다.

```text
이 도구는 production coding agent가 아니다.
작은 fixture 중심의 실험 harness다.
모델 성능을 보장하지 않는다.
repo scan heuristic은 틀릴 수 있다.
test가 부족하면 잘못된 patch도 통과할 수 있다.
command sandbox는 제한적이다.
```

## 이력서 / 포트폴리오 기록법

나쁜 기록:

```text
코딩 에이전트 만들었습니다.
```

좋은 기록:

```text
Built a micro coding-agent harness for small local models. The harness decomposes tasks into file-scoped work items, assembles constrained context packs, validates unified diffs against allowed files, runs targeted tests, and records trace data for fixture-based evaluation.
```

한국어:

```text
작은 로컬 모델을 위한 micro coding-agent harness를 구현했습니다. 큰 코딩 작업을 파일 단위 work item으로 분해하고, 제한된 context pack을 구성하며, unified diff를 allowed files 기준으로 검증하고, targeted test와 trace 기반 평가를 수행하도록 설계했습니다.
```

포트폴리오 글 구조:

```text
Problem
→ 작은 모델은 넓은 repo context와 큰 patch에서 쉽게 무너진다.

Design
→ problem card, work item, context pack, patch validation, test, trace loop.

Safety
→ allowed files, command allowlist, dependency 금지, patch size limit.

Evaluation
→ 5개 fixture, baseline vs micro harness 비교.

Result
→ 성공률, 실패 유형, 평균 attempts, boundary violation 수.

What I learned
→ agent 성능은 모델만이 아니라 task framing과 runtime boundary에 크게 좌우된다.
```

## 확장 과제

v0.2:

```text
fixture 20개로 확장
failureStage dashboard
context pack 크기별 비교
test failure repair prompt 개선
diff repair 또는 function-body replacement 실험
```

v0.3:

```text
real open-source small repo fixture
multi-model comparison
coverage-based test selection
simple benchmark report
automatic task decomposition
automatic relevant file selection
```

v0.4:

```text
IDE integration
human approval UI
longer task planning
parallel investigation agent
```

하지만 확장은 나중이다.

처음 질문은 하나다.

```text
작은 모델에게 큰 코딩을 시키지 않고,
harness가 일을 작게 쪼개면,
성공률과 안전성이 실제로 좋아지는가?
```

## 마지막 기준

이 과제를 끝냈다고 해서 완성형 코딩 에이전트를 만든 것은 아니다.

하지만 중요한 경험을 얻는다.

```text
agent loop를 직접 설계해봤다.
context pack을 직접 만들어봤다.
patch boundary를 runtime에서 막아봤다.
test와 trace로 agent 행동을 평가해봤다.
```

이 경험은 “AI로 코딩해봤다”와 다르다.

이건 코딩 에이전트가 왜 어려운지, 그리고 어디를 설계해야 하는지 몸으로 이해하는 일이다.

13장은 agent가 쓰는 tool의 손잡이를 검사하는 과제였다.

14장은 agent가 코드를 고치는 loop 자체를 작게 만드는 과제다.

둘을 합치면 시야가 바뀐다.

```text
좋은 모델 찾기
→ 좋은 runtime 만들기

프롬프트 더 잘 쓰기
→ 실패가 작게 나도록 harness 설계하기
```

이 방향이 agent harness engineering의 핵심이다.
