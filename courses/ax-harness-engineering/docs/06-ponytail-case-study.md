[← README](../README.md)

# 6장. Ponytail은 왜 강하게 작동하는가?

## 이 장에서 배울 것

5장까지 우리는 Codex의 지시문 표면을 큰 그림으로 봤다.

```text
AGENTS.md = durable project guidance
Skill = on-demand workflow manual
Hook = lifecycle command
Plugin = distribution unit
```

이제 첫 번째 실제 사례로 Ponytail을 본다.

Ponytail은 겉으로 보면 성격 프롬프트처럼 보인다.

```text
간단하게 해.
YAGNI를 지켜.
표준 라이브러리를 먼저 써.
불필요한 추상화를 만들지 마.
```

그런데 사용자가 직접 관찰한 것처럼, Ponytail은 단순히 GitHub 저장소를 plugin input에 넣었다고 끝나는 것이 아니다. Codex 설정의 Hooks 화면에 나타나고, 세션 시작 때 실행되며, 대화 중 `/ponytail` 같은 명령도 감지한다.

이 장의 중심 문장은 이것이다.

```text
Ponytail이 강하게 작동하는 이유는 좋은 문구 때문만이 아니라,
그 문구를 lifecycle에 다시 꽂는 runtime 구조 때문이다.
```

조금 더 개발자스럽게 말하면:

```text
Ponytail = instructions + plugin manifest + lifecycle hooks + state file
```

여기서 중요한 것은 “좋은 프롬프트냐?”가 아니라 “언제, 어떤 경로로, 어떤 상태를 읽고, 어떤 context를 다시 넣느냐?”다.

## 먼저 직관으로 이해하기

웹 앱에서 다크 모드를 만든다고 생각해보자.

가장 단순한 구현은 페이지 어딘가에 이렇게 써두는 것이다.

```text
이 사용자는 dark mode를 좋아함.
```

하지만 이것만으로는 실제 제품의 다크 모드가 되지 않는다.

진짜 다크 모드에는 보통 이런 구조가 있다.

```text
초기 로드
→ 저장된 preference 읽기
→ html class나 theme token 적용

사용자 toggle
→ preference 저장
→ 현재 화면 즉시 변경

다음 방문
→ 저장된 preference 다시 읽기
→ 같은 mode 복원
```

Ponytail도 비슷하다.

단순히 한 번 “lazy senior developer처럼 행동해”라고 말하는 것과, Ponytail plugin이 runtime에 붙어서 mode를 관리하는 것은 다르다.

```text
SessionStart
→ 현재 Ponytail 기본 mode 확인
→ 상태 파일에 mode 기록
→ Ponytail instruction을 context로 주입

UserPromptSubmit
→ 사용자의 `/ponytail full`, `/ponytail ultra`, `normal mode` 감지
→ 상태 파일 갱신
→ mode 변경 context 출력

SubagentStart
→ 부모 thread의 Ponytail 상태 확인
→ subagent에도 같은 instruction 주입
```

즉 Ponytail은 “문장”이 아니라 “작은 mode runtime”이다.

## 로컬에서 확인한 구조

이 장은 사용자의 현재 설치본 기준으로 확인했다.

```text
/Users/jhkim/.codex/plugins/cache/ponytail/ponytail/4.8.3/
```

따라서 아래의 파일 경로와 코드 조각은 Ponytail `4.8.3` 현재 설치본 기준이다. Ponytail 버전이나 Codex plugin schema가 바뀌면 파일명, manifest 필드, hook output shape은 달라질 수 있다.

핵심 파일은 많지 않다.

```text
.codex-plugin/plugin.json
hooks/claude-codex-hooks.json
hooks/ponytail-activate.js
hooks/ponytail-mode-tracker.js
hooks/ponytail-subagent.js
hooks/ponytail-runtime.js
hooks/ponytail-instructions.js
hooks/ponytail-config.js
skills/ponytail/SKILL.md
```

이름만 보면 역할이 거의 드러난다.

| 파일 | 역할 |
|---|---|
| `.codex-plugin/plugin.json` | Codex plugin manifest. skills와 hooks 위치를 선언한다. |
| `hooks/claude-codex-hooks.json` | 어떤 lifecycle event에서 어떤 command를 실행할지 선언한다. |
| `ponytail-activate.js` | 세션 시작 때 mode를 켜고 instruction을 출력한다. |
| `ponytail-mode-tracker.js` | 사용자 prompt에서 mode 변경 명령을 감지한다. |
| `ponytail-subagent.js` | subagent 시작 때 Ponytail instruction을 다시 주입한다. |
| `ponytail-runtime.js` | 상태 파일 읽기/쓰기와 Codex용 hook output 형식을 담당한다. |
| `ponytail-instructions.js` | `SKILL.md`에서 현재 mode에 맞는 instruction을 만든다. |
| `ponytail-config.js` | 기본 mode, config 경로, deactivation command를 처리한다. |
| `skills/ponytail/SKILL.md` | Ponytail의 실제 행동 규칙 본문이다. |

이 구성이 중요한 이유는 하나다.

```text
Ponytail의 “행동 변화”는 한 파일의 프롬프트가 아니라,
manifest → hook discovery → command 실행 → state 저장 → context 주입
으로 이어지는 흐름에서 나온다.
```

## Plugin manifest: 왜 Hooks 화면에 보였나

Ponytail의 Codex plugin manifest에는 이런 필드가 있다.

```json
{
  "name": "ponytail",
  "version": "4.8.3",
  "skills": "./skills/",
  "hooks": "./hooks/claude-codex-hooks.json",
  "interface": {
    "capabilities": ["Instructions", "Lifecycle hooks"]
  }
}
```

여기서 핵심은 `hooks`다.

```text
"hooks": "./hooks/claude-codex-hooks.json"
```

Codex는 plugin을 설치할 때 manifest를 보고 이 plugin이 어떤 capability를 제공하는지 알 수 있다. `skills`가 있으면 skill 목록에 들어갈 수 있고, `hooks`가 있으면 hook definition을 발견할 수 있다.

다만 manifest의 세부 필드와 `interface.capabilities` 같은 표시용 metadata는 plugin schema와 버전에 따라 달라질 수 있다. 여기서는 Ponytail `4.8.3` 설치본에서 확인한 manifest를 사례로 보는 것이 안전하다.

그래서 Ponytail은 Settings > Hooks에 나타난다.

반대로 어떤 repository에 좋은 `SKILL.md`나 `AGENTS.md`가 있더라도, Codex plugin manifest에서 hooks 경로를 선언하지 않으면 “plugin-bundled hook”으로 발견될 수 없다.

이 지점이 사용자가 처음 헷갈렸던 부분이다.

```text
GitHub repo를 plugin input에 넣었다
≠ Codex가 모든 파일을 lifecycle hook으로 실행한다
```

Plugin은 배포 단위다. 그 안에서 무엇이 Codex에 노출되는지는 manifest가 결정한다.

## Hook file: 언제 어떤 command를 실행하나

Ponytail의 hook declaration은 `hooks/claude-codex-hooks.json`에 있다.

핵심 event는 세 가지다.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/ponytail-activate.js\"; exit 0",
            "timeout": 5,
            "statusMessage": "Loading ponytail mode..."
          }
        ]
      }
    ],
    "SubagentStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/ponytail-subagent.js\"; exit 0",
            "timeout": 5,
            "statusMessage": "Loading ponytail mode..."
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/ponytail-mode-tracker.js\"; exit 0",
            "timeout": 5,
            "statusMessage": "Tracking ponytail mode..."
          }
        ]
      }
    ]
  }
}
```

이 선언은 이렇게 읽으면 된다.

```text
SessionStart
→ 세션이 startup/resume/clear/compact로 시작될 때
→ ponytail-activate.js 실행

UserPromptSubmit
→ 사용자가 prompt를 보낼 때
→ ponytail-mode-tracker.js 실행

SubagentStart
→ subagent가 시작될 때
→ ponytail-subagent.js 실행
```

여기서 `timeout: 5`도 중요하다. 5장에서 봤듯 hook은 runtime lifecycle에 끼어드는 command다. 느린 hook은 대화 UX를 직접 느리게 만든다. Ponytail은 mode 주입과 상태 갱신만 하므로 5초 제한으로 충분하다.

작지만 좋은 설계다.

```text
세션 시작 시 한 번 주입
대화 입력마다 가벼운 command 감지
subagent 시작 시 누락 방지
```

이 정도면 “mode 유지”에는 충분하다. 굳이 매 tool call마다 끼어들 필요는 없다.

## SessionStart: mode를 켜고 instruction을 넣는다

`ponytail-activate.js`는 SessionStart에서 실행된다.

역할은 크게 세 가지다.

```text
1. default mode 결정
2. 상태 파일에 현재 mode 기록
3. Ponytail instruction을 hook output으로 출력
```

default mode는 `ponytail-config.js`에서 결정한다.

우선순위는 다음과 같다.

```text
1. PONYTAIL_DEFAULT_MODE 환경 변수
2. config file의 defaultMode
3. 기본값 full
```

이 구조는 웹 앱의 설정 우선순위와 비슷하다.

```text
runtime env
→ user config
→ app default
```

mode가 `off`가 아니면 `ponytail-activate.js`는 `setMode(mode)`를 호출한다. 실제 상태 파일 처리는 `ponytail-runtime.js`가 한다.

Codex 환경에서는 상태 파일 위치가 `PLUGIN_DATA` 아래로 바뀐다.

```js
const STATE_FILE = '.ponytail-active';
const isCodex = !isCopilot && Boolean(process.env.PLUGIN_DATA);

let stateDir = getClaudeDir();
if (isCodex) stateDir = process.env.PLUGIN_DATA;

const statePath = path.join(stateDir, STATE_FILE);
```

즉 Ponytail은 “모델이 자기 상태를 기억한다”고 믿지 않는다. runtime data directory에 작은 파일을 써서 mode를 기억한다.

이 점이 하네스 엔지니어링적으로 중요하다.

```text
mode persistence = model memory가 아니라 runtime state다
```

그 다음 `getPonytailInstructions(mode)`로 실제 instruction text를 만든다. 이 함수는 `skills/ponytail/SKILL.md`를 읽고 현재 intensity level에 맞게 일부 내용을 필터링한다.

결과적으로 SessionStart 시점에 모델은 이런 추가 context를 받는다.

```text
PONYTAIL MODE ACTIVE — level: full

You are a lazy senior developer...
...
```

여기까지 보면 Ponytail은 단순 프롬프트와 비슷해 보일 수 있다. 하지만 차이는 다음 단계에서 생긴다.

## UserPromptSubmit: mode command를 대화 중에 감지한다

Ponytail은 대화 중 사용자가 mode를 바꿀 수 있다.

```text
/ponytail lite
/ponytail full
/ponytail ultra
/ponytail off
normal mode
stop ponytail
```

이 기능은 모델이 “알아서 기억”하는 것이 아니다. `UserPromptSubmit` hook에서 `ponytail-mode-tracker.js`가 stdin으로 들어온 prompt를 읽고 직접 파싱한다.

이 설계는 `UserPromptSubmit`의 event contract와도 잘 맞는다. 5장에서 본 것처럼 `UserPromptSubmit`은 matcher가 사용되지 않는다. 그래서 Ponytail도 matcher로 `/ponytail` prompt만 고르는 대신, command 내부에서 prompt JSON을 직접 읽고 필요한 경우만 처리한다.

핵심 흐름은 이렇다.

```text
stdin JSON 읽기
→ data.prompt 추출
→ /ponytail 명령인지 확인
→ mode 결정
→ 상태 파일 setMode(mode) 또는 clearMode()
→ 필요하면 hook output으로 mode changed context 출력
```

대략 이런 패턴이다.

```js
const data = JSON.parse(input.replace(/^\uFEFF/, ''));
const prompt = (data.prompt || '').trim().toLowerCase();

if (/^[/@$]ponytail/.test(prompt)) {
  ...
  setMode(mode);
  writeHookOutput(
    'UserPromptSubmit',
    mode,
    'PONYTAIL MODE CHANGED — level: ' + mode,
  );
}
```

여기서 중요한 감각은 이것이다.

```text
slash command는 모델에게 “잘 해석해줘”라고 맡긴 기능이 아니다.
hook command가 사용자 prompt를 runtime event로 받아 파싱하는 기능이다.
```

물론 최종적으로 mode changed message는 다시 모델 context에 들어갈 수 있다. 하지만 mode 변경의 1차 처리는 JavaScript 코드가 한다.

이것은 웹 개발자가 익숙한 패턴이다.

```text
사용자 입력
→ route/middleware에서 command 파싱
→ server-side state 변경
→ 다음 render/request에 반영
```

Ponytail도 똑같다.

```text
사용자 prompt
→ UserPromptSubmit hook에서 command 파싱
→ .ponytail-active 변경
→ 다음 model context에 mode instruction 반영
```

## SubagentStart: 부모 thread의 mode를 자식에게 전달한다

Ponytail 4.8.3에는 `SubagentStart` hook도 있다.

주석이 핵심을 잘 설명한다.

```text
SessionStart context is parent-thread only and never reaches subagents,
so without this every Task-spawned agent runs ponytail-unaware.
```

즉 부모 thread에서 SessionStart로 들어간 Ponytail instruction이 subagent에게 자동으로 전달된다고 가정하면 안 된다. 이 말은 모든 부모 context가 어떤 경우에도 subagent로 전달되지 않는다는 일반 법칙이 아니다. Ponytail 관점에서는 SessionStart에서 주입한 mode instruction이 subagent에 자동 보장되지 않으므로, `SubagentStart`에서 다시 주입하는 설계가 필요하다는 뜻이다.

그래서 `ponytail-subagent.js`는 상태 파일을 읽는다.

```js
const mode = readMode();

if (!mode || mode === 'off') {
  process.exit(0);
}

writeHookOutput('SubagentStart', mode, getPonytailInstructions(mode));
```

이 설계는 아주 좋은 교재다.

우리가 앞 장에서 배운 사실이 실제 문제로 나타난다.

```text
context는 “전체 우주에 자동 전파되는 공기”가 아니다.
각 runtime boundary마다 다시 넣어야 한다.
```

Subagent는 새로운 agent execution boundary다. 부모의 대화 분위기나 mode를 그대로 안다고 생각하면 안 된다. 그래서 Ponytail은 상태 파일을 공유 storage로 쓰고, SubagentStart hook에서 instruction을 다시 주입한다.

웹 개발로 비유하면 이렇다.

```text
부모 request의 req.locals.theme
≠ background worker가 자동으로 아는 theme

공유 DB/cache/config에 저장
→ worker 시작 시 다시 읽기
```

Ponytail의 subagent hook은 바로 이 역할을 한다.

## Hook output: Codex에 맞춰 JSON으로 말한다

Ponytail은 여러 플랫폼을 지원한다. 그래서 hook output도 플랫폼별로 다르게 쓴다.

Codex 환경은 `PLUGIN_DATA` 환경 변수로 감지한다.

```js
const isCopilot = Boolean(process.env.COPILOT_PLUGIN_DATA);
const isCodex = !isCopilot && Boolean(process.env.PLUGIN_DATA);
```

Codex일 때 `writeHookOutput`은 대략 이런 JSON을 stdout으로 쓴다.

```js
const output = { systemMessage: `PONYTAIL:${mode.toUpperCase()}` };
if (context) {
  output.hookSpecificOutput = {
    hookEventName: event,
    additionalContext: context,
  };
}
process.stdout.write(JSON.stringify(output));
```

5장에서 본 것처럼 hook stdout은 event마다 의미가 다르다. Ponytail은 Codex가 이해할 수 있는 hook output shape으로 `additionalContext`를 넣는다.

여기서 한 가지 주의할 점이 있다.

`systemMessage`라는 필드명이 코드에 있다고 해서 이것이 OpenAI API의 최상위 system message와 동일하다고 단정하면 안 된다. 여기서는 Codex hook output contract 안에서 Ponytail 상태를 표시하기 위한 필드로 보는 것이 안전하다.

우리가 공부해야 할 포인트는 필드 이름 자체보다 구조다.

```text
hook command가 stdout으로 structured output을 낸다.
Codex runtime이 그 output을 event contract에 맞게 해석한다.
그 결과 일부 내용이 모델 가시 context로 들어간다.
```

이것이 “hook이 prompt보다 강하다”는 말의 정확한 의미다.

Hook이 모델보다 높은 마법 권한을 가진다는 뜻이 아니다. 모델 호출 전에 runtime이 별도 코드를 실행하고, 그 결과를 context assembly에 섞을 수 있다는 뜻이다.

## Skill 본문은 여전히 중요하다

지금까지 runtime 구조를 강조했지만, Ponytail의 실제 행동 방향은 결국 instruction text에 들어 있다.

`skills/ponytail/SKILL.md`에는 Ponytail의 핵심 철학이 있다.

예를 들어:

```text
Does this need to exist at all?
Already in this codebase?
Stdlib does it?
Native platform feature covers it?
Already-installed dependency solves it?
Can it be one line?
Only then: the minimum code that works.
```

이 “ladder”는 단순히 짧게 답하라는 지시가 아니다. 해결 전략의 우선순위를 바꾼다.

보통 모델은 사용자가 “기능 A 만들어줘”라고 하면 곧바로 구현을 시작하려는 경향이 있다. Ponytail은 그 앞에 판단 단계를 삽입한다.

```text
정말 필요한가?
이미 있는가?
표준 기능으로 되는가?
플랫폼 기능으로 되는가?
새 dependency 없이 되는가?
한 줄로 되는가?
```

이것은 output style이 아니라 decision checklist에 가깝다.

여기서 checklist라는 말은 모델 내부 알고리즘을 바꾼다는 뜻이 아니다. Ponytail instruction이 solution space를 평가할 때 우선적으로 고려할 기준과 선호도를 context로 주입한다는 뜻이다.

```text
기본 모델:
요청 이해 → 구현 계획 → 코드 작성

Ponytail mode:
요청 이해 → 안 만들어도 되는지 확인 → 기존 코드 확인
→ stdlib/native 확인 → 최소 구현 → 짧은 설명
```

그래서 Ponytail은 코드량을 줄이는 방향으로 모델을 강하게 유도한다. 다만 이것은 수학적으로 보장되는 결과라기보다, instruction과 lifecycle 주입이 만든 행동 경향이다. 그리고 “무조건 대충”도 아니다.

`SKILL.md`에는 Lazy가 금지되는 영역도 명시되어 있다.

```text
input validation at trust boundaries
error handling that prevents data loss
security measures
accessibility basics
anything explicitly requested
```

즉 Ponytail의 핵심은 “게으름”이라는 말장난이 아니라, ownership cost를 줄이는 engineering policy다.

```text
적게 만들되, 위험한 것을 생략하지 않는다.
```

이 균형이 없으면 Ponytail은 그냥 부실 구현 모드가 된다.

## 왜 단순 프롬프트보다 드라마틱한가

이제 질문에 답할 수 있다.

왜 Ponytail은 그냥 `AGENTS.md`에 “간단하게 해”라고 쓰는 것보다 더 강하게 느껴질까?

차이는 네 가지다.

### 1. 세션 시작 때 자동으로 들어간다

사용자가 매번 프롬프트에 붙이지 않아도 된다.

```text
SessionStart
→ hook 실행
→ instruction 주입
```

이것만 보면 AGENTS.md와 비슷해 보일 수 있다. 하지만 Ponytail은 여기서 끝나지 않는다.

### 2. 대화 중 mode 변경을 runtime이 감지한다

`/ponytail ultra` 같은 입력은 단순 자연어 요청이 아니다. `UserPromptSubmit` hook이 prompt를 보고 상태 파일을 바꾼다.

```text
사용자: /ponytail ultra
→ hook command가 prompt parse
→ .ponytail-active = ultra
→ mode changed context 출력
```

AGENTS.md는 이런 lifecycle command가 아니다. 파일에 적힌 규칙을 읽히게 할 수는 있지만, 사용자 입력마다 코드를 실행해 상태를 갱신하지는 않는다.

### 3. 상태가 모델 밖에 저장된다

Ponytail은 `.ponytail-active` 파일을 쓴다.

이것은 작지만 결정적인 차이다.

```text
모델 memory
→ 없음. context에 없으면 모른다.

Ponytail mode
→ runtime state file에 있음. hook이 다시 읽을 수 있다.
```

상태를 모델에게만 맡기지 않기 때문에, 대화가 길어져도 mode 복원이 쉬워진다.

### 4. subagent boundary를 따로 처리한다

부모 thread의 context가 subagent에게 자동 전파되지 않는 문제를 `SubagentStart` hook으로 해결한다.

이것은 단순 프롬프트로는 놓치기 쉬운 지점이다.

```text
parent session context
→ subagent에 자동 보장 안 됨
→ SubagentStart hook에서 다시 주입
```

이런 설계는 “프롬프트 잘 쓰기”라기보다 “runtime boundary 관리”에 가깝다.

## AGENTS.md와 비교하기

Ponytail을 이해할 때 가장 흔한 혼동은 이것이다.

```text
어차피 둘 다 모델에게 문구 넣는 거 아닌가?
```

결과만 보면 둘 다 모델 context에 instruction을 넣을 수 있다. 하지만 engineering 관점에서는 다르다.

| 구분 | AGENTS.md | Ponytail hook mode |
|---|---|---|
| 주 용도 | repo/folder 규칙 | runtime behavior mode |
| 실행성 | 파일 읽기, command 실행 아님 | lifecycle event마다 command 실행 |
| 상태 변경 | 사용자 prompt를 보고 자체 상태 변경하지 않음 | `/ponytail` 명령으로 state file 변경 |
| 적용 시점 | run/session instruction assembly | SessionStart, UserPromptSubmit, SubagentStart |
| 배포 방식 | repo 파일 | plugin manifest + skills + hooks |
| Settings > Hooks 노출 | 해당 없음 | manifest의 hooks 선언 때문에 노출 |

예를 들어 `AGENTS.md`에 이렇게 쓸 수는 있다.

```md
# Development style

- Prefer simple solutions.
- Do not add dependencies unless necessary.
- Use standard library first.
```

좋은 규칙이다. 실제로 repo 기본 원칙이라면 이렇게 두는 게 맞다.

하지만 이것만으로는 다음을 할 수 없다.

```text
사용자가 /ponytail ultra를 입력했는지 매 prompt마다 검사
mode 상태 파일 갱신
subagent 시작 때 부모 mode 재주입
statusline badge용 flag 유지
```

그래서 AGENTS.md와 Ponytail hook은 경쟁 관계가 아니다.

```text
AGENTS.md
→ repo의 기본 작업 규칙

Ponytail hook
→ 대화 runtime의 mode controller
```

목적이 다르면 표면도 달라진다.

## 작은 예시: minimal mode plugin을 직접 만든다면

Ponytail을 흉내 내서 아주 작은 mode plugin을 만든다고 해보자.

필요한 최소 구성은 대략 이렇다.

```text
.codex-plugin/plugin.json
hooks/hooks.json
hooks/activate.js
hooks/mode-tracker.js
skills/my-mode/SKILL.md
```

`plugin.json`에는 hook file을 선언한다.

```json
{
  "name": "my-mode",
  "version": "0.1.0",
  "skills": "./skills/",
  "hooks": "./hooks/hooks.json"
}
```

`hooks.json`에는 lifecycle event를 선언한다.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume",
        "hooks": [
          {
            "type": "command",
            "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/activate.js\"; exit 0",
            "timeout": 5
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/mode-tracker.js\"; exit 0",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

`activate.js`는 현재 mode instruction을 출력한다.

```js
process.stdout.write(JSON.stringify({
  hookSpecificOutput: {
    hookEventName: 'SessionStart',
    additionalContext: 'MY MODE ACTIVE. Prefer boring code and small diffs.'
  }
}));
```

`mode-tracker.js`는 사용자 prompt에서 명령을 감지한다.

```js
let input = '';
process.stdin.on('data', chunk => { input += chunk; });
process.stdin.on('end', () => {
  const data = JSON.parse(input);
  const prompt = (data.prompt || '').trim().toLowerCase();

  if (prompt === '/my-mode off') {
    // state file 제거
  }
});
```

이 예시는 완전한 제품이 아니다. 실제로 배포하려면 `PLUGIN_DATA` 기반 상태 경로, JSON parse 실패 처리, mode off 처리, `SubagentStart` 재주입, Windows command, hook trust/review까지 고려해야 한다. 하지만 Ponytail의 핵심 골격은 보인다.

```text
manifest가 hook을 노출한다.
hook이 lifecycle event에서 command를 실행한다.
command가 state를 읽고 쓴다.
command output이 context로 들어간다.
```

AX 개발자로 성장하려면 이 골격을 보는 눈이 중요하다.

## 흔한 오해

### 오해 1. “Ponytail은 그냥 강한 system prompt다”

절반만 맞다.

Ponytail의 행동 규칙은 instruction text다. 하지만 그 text가 들어가는 경로는 plugin hook이다.

```text
instruction text
 + SessionStart hook
 + UserPromptSubmit hook
 + state file
 + SubagentStart hook
```

이 조합이 Ponytail을 runtime mode처럼 만든다.

### 오해 2. “Hook이 있으니 모델이 반드시 지킨다”

아니다.

Hook은 context를 넣거나 tool 실행 전후에 개입할 수 있다. 하지만 최종 답변을 만드는 것은 여전히 모델이다. 모델은 instruction을 강하게 따르도록 조건화되지만, 수학적 보장처럼 100% 고정되는 것은 아니다.

그래서 중요한 규칙은 여러 층으로 둔다.

```text
style/policy reminder
→ instruction/hook

실행 제한
→ permission/sandbox/approval

품질 보증
→ tests/review/CI/evaluation
```

Ponytail은 주로 첫 번째 층, 즉 행동 정책과 작업 습관을 바꾸는 데 강하다.

### 오해 3. “상태 파일이 있으면 모델이 기억하는 것이다”

아니다.

상태 파일은 모델의 기억이 아니다. runtime storage다.

모델이 그 내용을 쓰려면 hook이나 tool이 읽어서 context에 넣어야 한다.

```text
.ponytail-active 파일 존재
→ hook이 읽음
→ instruction 생성
→ hook output
→ context assembly
→ 모델이 봄
```

이 흐름이 끊기면 모델은 상태 파일을 모른다.

### 오해 4. “Settings > Hooks에 보이면 plugin 전체 코드가 항상 실행된다”

아니다.

실행되는 것은 hook declaration에 연결된 command다.

Ponytail의 경우:

```text
SessionStart
→ ponytail-activate.js

UserPromptSubmit
→ ponytail-mode-tracker.js

SubagentStart
→ ponytail-subagent.js
```

다른 파일들은 이 command들이 `require()`하거나 읽을 때만 실행된다.

### 오해 5. “Ponytail은 무조건 적게만 하니 위험하다”

Ponytail을 잘못 이해하면 그렇게 보인다.

하지만 실제 rule은 “최소 구현”만 말하지 않는다. “언제 lazy하면 안 되는가”도 포함한다.

```text
보안
데이터 손실 방지 error handling
trust boundary input validation
accessibility
사용자가 명시적으로 요구한 것
```

좋은 minimalism은 삭제만 잘하는 것이 아니다. 남겨야 할 것을 아는 것이다. 야전 삽 같은 철학이다. 가볍지만, 땅은 파야 한다.

## 정리

Ponytail은 좋은 case study다.

왜냐하면 지금까지 배운 개념이 한 번에 모여 있기 때문이다.

```text
Plugin
→ Ponytail을 설치 가능한 단위로 묶는다.

Skill
→ Ponytail의 행동 규칙 본문을 제공한다.

Hook
→ SessionStart, UserPromptSubmit, SubagentStart에 command를 꽂는다.

Runtime state
→ .ponytail-active 파일로 현재 mode를 저장한다.

Context assembly
→ hook output이 additionalContext로 들어가 모델 행동을 조건화한다.
```

Ponytail이 단순 프롬프트보다 강하게 작동하는 이유는 문구가 더 세서가 아니다.

```text
문구가 runtime lifecycle에 연결되어 있기 때문이다.
```

이 차이를 이해하면, “나만의 에이전트”를 만들 때도 방향이 바뀐다.

처음부터 거대한 agent를 만들 필요는 없다.

작게 시작해도 된다.

```text
하나의 좋은 rule
하나의 lifecycle event
하나의 작은 state file
하나의 context injection
```

이 네 개만 있어도 단순 프롬프트보다 훨씬 agent harness다운 행동을 만들 수 있다.

## 생각해볼 질문

1. 내가 만들고 싶은 agent behavior는 단순 instruction인가, runtime mode인가?
2. 그 behavior는 세션 시작 때 한 번이면 충분한가, 매 prompt마다 확인해야 하는가?
3. 상태가 필요한가? 필요하다면 모델 memory가 아니라 어디에 저장할 것인가?
4. subagent나 background task처럼 context boundary가 생기면 같은 규칙을 어떻게 전달할 것인가?
5. Hook으로 넣을 것과 AGENTS.md에 둘 것을 어떤 기준으로 나눌 것인가?
