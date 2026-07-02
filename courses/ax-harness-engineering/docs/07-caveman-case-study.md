[← README](../README.md)

# 7장. Caveman으로 보는 surface와 adapter의 차이

## 이 장에서 배울 것

6장에서는 Ponytail을 보며 이런 구조를 배웠다.

```text
Ponytail = instructions + plugin manifest + lifecycle hooks + state file
```

이번 장에서는 반대쪽 사례를 본다. Caveman이다.

사용자는 Ponytail과 Caveman을 비슷하게 설치했다. 둘 다 GitHub repository를 Codex plugin 추가 input에 넣었다. 그런데 결과는 달랐다.

```text
Ponytail
→ Settings > Hooks에 보임
→ review/trust 대상 hook으로 나타남
→ SessionStart/UserPromptSubmit hook이 실행될 수 있음

Caveman
→ skill 목록에는 보임
→ 하지만 Settings > Hooks에는 Ponytail처럼 보이지 않음
```

처음 보면 이상하다.

왜냐하면 Caveman repository 안에도 hook 코드가 있기 때문이다.

이 장의 중심 문장은 이것이다.

```text
하네스 엔지니어링에서 중요한 것은 파일이 있는지가 아니라,
그 파일이 어떤 surface에 연결되어 언제 발견되고 언제 실행되는가다.
```

Caveman 사례의 핵심은 “왜 Hooks에 안 떴나?” 하나가 아니다. 더 중요한 질문은 이것이다.

```text
같은 Caveman behavior가
AGENTS.md로 들어갈 때,
Skill로 들어갈 때,
Hook으로 들어갈 때,
Plugin distribution으로 들어갈 때
각각 무엇이 달라지는가?
```

이 차이를 보면 하네스의 본질이 보인다.

```text
Instruction text
→ 모델에게 말하는 내용

Discovery path
→ runtime이 무엇을 발견하는가

Execution path
→ 실제 command가 언제 실행되는가

State path
→ 상태가 어디에 저장되고 누가 읽는가

Output contract
→ 실행 결과가 어떤 형식으로 context에 들어가는가
```

웹 개발자로 치면 익숙한 이야기다.

```text
controller 파일이 있음
≠ router에 등록됨

middleware 함수가 있음
≠ app.use(...)에 연결됨

migration 파일이 있음
≠ DB에 적용됨
```

Caveman 사례는 이 차이를 아주 잘 보여준다.

## 먼저 직관으로 이해하기

Express 서버를 떠올려보자.

프로젝트에 이런 파일이 있을 수 있다.

```text
src/middleware/auth.ts
```

파일 안에는 멋진 auth middleware가 들어 있다.

```ts
export function auth(req, res, next) {
  // token 검사
  next();
}
```

하지만 서버가 이 파일을 자동으로 실행하지는 않는다.

실제로 요청 흐름에 끼우려면 어딘가에서 연결해야 한다.

```ts
app.use(auth);
```

또는 router에 붙여야 한다.

```ts
router.get('/me', auth, getMe);
```

이 연결이 없으면 auth middleware는 그냥 파일이다. 좋은 코드지만 runtime path에 없다.

Codex plugin도 비슷하다.

```text
repository 안에 hook script가 있음
≠ Codex가 lifecycle hook으로 실행함

plugin manifest가 hooks를 선언함
→ Codex가 plugin-bundled hook으로 발견할 수 있음
```

Ponytail은 이 연결이 있었다. Caveman의 Codex plugin manifest에는 이 연결이 없었다.

## 먼저 adapter로 보기

Caveman 사례는 npm package와 framework adapter를 떠올리면 쉽다.

어떤 package가 core logic을 제공한다고 해보자.

```text
@acme/auth-core
```

이 core package 안에는 token 검증 함수가 있다.

```ts
verifyToken(token)
```

하지만 이것만으로 Express middleware가 되지는 않는다.

Express에서 쓰려면 Express adapter가 필요하다.

```ts
app.use(authExpressMiddleware())
```

Next.js middleware로 쓰려면 또 다른 adapter가 필요하다.

```ts
export function middleware(req) { ... }
```

같은 core logic이어도 runtime마다 연결 방식이 다르다.

Caveman도 그렇다.

```text
core behavior
→ caveman SKILL.md

Claude Code plugin adapter
→ .claude-plugin/plugin.json + src/hooks/*

Codex skill distribution
→ plugins/caveman/.codex-plugin/plugin.json + skills/*

Codex lifecycle hook adapter
→ 현재 manifest에는 없음
```

Ponytail은 Codex lifecycle adapter가 연결된 사례다. Caveman은 현재 확인한 Codex distribution에서는 skill surface까지만 연결된 사례다.

그래서 이 장은 “Caveman hook 미노출 원인 분석”이라기보다 “같은 행동 규칙도 surface와 adapter에 따라 전혀 다른 harness가 된다”는 사례 분석이다.

## 실제 Caveman 설치본에서 확인한 구조

이 장은 사용자의 현재 로컬 설치본을 기준으로 확인했다.

```text
/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/
```

중요한 파일은 두 묶음으로 나뉜다.

```text
Codex plugin distribution:
plugins/caveman/.codex-plugin/plugin.json
plugins/caveman/skills/...
plugins/caveman/agents/...

Repository root / Claude-oriented files:
.claude-plugin/plugin.json
.codex/hooks.json
.codex/config.toml
src/hooks/caveman-activate.js
src/hooks/caveman-mode-tracker.js
src/hooks/caveman-statusline.sh
src/hooks/README.md
```

여기서 가장 중요한 파일은 Codex용 manifest다.

```text
plugins/caveman/.codex-plugin/plugin.json
```

이 파일에는 `skills`가 있다.

```json
{
  "name": "caveman",
  "version": "0.1.0",
  "skills": "./skills/",
  "interface": {
    "displayName": "Caveman",
    "capabilities": ["Write"]
  }
}
```

하지만 `hooks` 필드는 없다.

```text
hooks: 없음
```

이 한 줄 차이가 Settings > Hooks에서 보이는 차이를 만든다.

```text
Ponytail Codex manifest
→ "hooks": "./hooks/claude-codex-hooks.json"

Caveman Codex manifest
→ "skills": "./skills/"
→ hooks 선언 없음
```

그래서 Codex는 Caveman을 skill 제공 plugin으로는 볼 수 있지만, plugin-bundled lifecycle hook 제공 plugin으로는 보지 않는다.

## 그런데 Caveman에도 hook 코드는 있다

여기서 헷갈림이 생긴다.

Caveman repository 안에는 실제로 hook 관련 파일이 있다.

```text
src/hooks/caveman-activate.js
src/hooks/caveman-mode-tracker.js
src/hooks/caveman-statusline.sh
src/hooks/README.md
.claude-plugin/plugin.json
.codex/hooks.json
```

`src/hooks/README.md`는 Caveman hook의 의도를 이렇게 설명한다.

```text
SessionStart hook
→ .caveman-active flag file 쓰기
→ caveman rules를 hidden context로 출력

UserPromptSubmit hook
→ /caveman command나 자연어 activation/deactivation 감지
→ flag file 갱신
→ per-turn reinforcement 출력

statusline script
→ .caveman-active 읽고 badge 표시
```

즉 설계 자체는 Ponytail과 닮았다.

```text
SessionStart
→ caveman-activate.js

UserPromptSubmit
→ caveman-mode-tracker.js

state file
→ .caveman-active
```

하지만 이 hook들은 현재 Codex plugin manifest의 `hooks`로 노출되어 있지 않다.

이 차이를 정확히 잡아야 한다.

```text
Caveman repo에는 hook implementation이 있다.
하지만 Codex plugin distribution manifest가 그 hook을 lifecycle hook으로 선언하지 않는다.
```

파일이 있다는 사실은 “가능성”이다.

Manifest에 연결되어 runtime이 발견하는 것은 “실행 경로”다.

## Ponytail과 Caveman을 나란히 보기

두 plugin을 비교하면 차이가 바로 보인다.

| 구분 | Ponytail | Caveman |
|---|---|---|
| Codex manifest 위치 | `.codex-plugin/plugin.json` | `plugins/caveman/.codex-plugin/plugin.json` |
| Codex manifest의 `skills` | 있음 | 있음 |
| Codex manifest의 `hooks` | 있음 | 없음 |
| Settings > Hooks 노출 | 됨 | 안 됨 |
| Skill 목록 노출 | 됨 | 됨 |
| hook implementation 코드 | 있음 | 있음 |
| 현재 Codex plugin-bundled hook 연결 | 있음 | 없음 |

이 표의 핵심은 이 줄이다.

```text
hook implementation 코드 있음
≠
Codex plugin-bundled hook 연결 있음
```

Caveman에는 `.claude-plugin/plugin.json`도 있다. 이 파일에는 `hooks`가 inline으로 들어 있다.

```json
{
  "name": "caveman",
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"${CLAUDE_PLUGIN_ROOT}/src/hooks/caveman-activate.js\""
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"${CLAUDE_PLUGIN_ROOT}/src/hooks/caveman-mode-tracker.js\""
          }
        ]
      }
    ]
  }
}
```

하지만 이건 Claude Code plugin manifest다. Codex가 사용하는 Codex plugin manifest와 같다고 보면 안 된다.

```text
.claude-plugin/plugin.json
→ Claude Code용 plugin metadata

plugins/caveman/.codex-plugin/plugin.json
→ Codex용 plugin metadata
```

이 둘은 같은 repository 안에 있어도 서로 다른 distribution surface다.

## `.codex/hooks.json`은 왜 자동으로 안 먹었나?

Caveman repository root에는 이런 파일도 있다.

```text
.codex/hooks.json
.codex/config.toml
```

`.codex/hooks.json` 안에는 `SessionStart` hook이 있다.

개념적으로는 이런 내용이다.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'CAVEMAN MODE ACTIVE ...'"
          }
        ]
      }
    ]
  }
}
```

그럼 왜 이게 Settings > Hooks에 안 떴을까?

이유는 `.codex/hooks.json`이 “그 repository를 project로 열었을 때의 project-local Codex config”에 가깝기 때문이다.

Codex가 현재 작업 중인 project root에서 `.codex/hooks.json`을 발견하면 project-local hook으로 볼 수 있다. 단, project-local hook으로 동작하려면 그 repository가 현재 Codex project로 열려 있어야 하고, project `.codex/` config layer가 trusted 상태여야 한다. 어떤 GitHub repository를 plugin으로 설치했다는 사실만으로, 그 repository root의 `.codex/hooks.json`이 자동으로 “plugin-bundled hook”이 되는 것은 아니다.

Plugin-bundled hook으로 노출하려면 Codex plugin manifest가 hook file을 가리켜야 한다.

```text
project-local hook
→ 현재 project의 .codex/hooks.json

plugin-bundled hook
→ plugin manifest의 hooks 필드가 가리키는 hook definition
```

Caveman의 `.codex/hooks.json`은 repository 안에 존재하지만, 사용자가 작업하던 project의 `.codex/hooks.json`도 아니고, Codex plugin manifest의 `hooks` 필드로 연결된 것도 아니었다.

그래서 Settings > Hooks에 Ponytail처럼 보이지 않은 것이다.

## Skill은 왜 보였나?

Caveman의 Codex manifest에는 `skills`가 있다.

```json
{
  "skills": "./skills/"
}
```

그래서 Codex는 Caveman plugin에서 skill들을 발견할 수 있다.

실제 세션에서도 다음 같은 skills가 보였다.

```text
caveman:caveman
caveman:caveman-commit
caveman:caveman-review
caveman:caveman-compress
caveman:caveman-help
caveman:caveman-stats
caveman:cavecrew
```

이건 hook과 별개다.

```text
Skill discovery
→ manifest의 skills 경로 또는 plugin skill distribution을 봄

Hook discovery
→ manifest의 hooks 경로, active config layer의 hooks.json/config.toml을 봄
```

따라서 “Caveman skill은 있는데 hook은 없다”는 말은 모순이 아니다.

정확히는 이렇게 말해야 한다.

```text
Caveman의 Codex plugin distribution은 skills를 노출한다.
하지만 현재 Codex plugin manifest 기준으로 lifecycle hooks를 노출하지 않는다.
```

## 같은 Caveman, 다른 surface

이제 이 장의 큰 질문으로 돌아오자.

같은 “Caveman처럼 짧고 정확하게 말하기”라는 behavior도 어떤 surface에 연결되느냐에 따라 완전히 다른 harness가 된다.

| 형태 | 모델에게 들어가는 방식 | 실행 코드 | 상태 유지 | 자동 강화 | UI 노출 |
|---|---|---|---|---|---|
| AGENTS.md Caveman | session/run instruction | 없음 | 없음 | 약함 | Hooks 아님 |
| Skill Caveman | 선택 시 `SKILL.md` 로드 | 자동 lifecycle 실행은 없음, optional script 가능 | 없음 | 작업 단위 | Skills |
| Hook Caveman | lifecycle hook output | 있음 | 가능 | 강함 | Hooks |
| Plugin Caveman | surface 배포 단위 | manifest에 따라 다름 | manifest에 따라 다름 | manifest에 따라 다름 | capabilities/discovery에 따라 |

이 표가 중요한 이유는 결과만 보면 각 방식이 모두 비슷해 보일 수 있기 때문이다.

```text
AGENTS.md에 Caveman 규칙을 넣음
→ 모델이 짧게 말할 수 있음

Caveman skill을 사용함
→ 모델이 짧게 말할 수 있음

Caveman hook이 매 prompt마다 reinforcement를 넣음
→ 모델이 짧게 말할 수 있음
```

하지만 하네스 관점에서는 다르다.

```text
무엇이 발견되는가?
언제 로드되는가?
코드가 실행되는가?
상태를 저장하는가?
다음 turn에서 다시 강화되는가?
사용자가 UI에서 review/trust할 수 있는가?
```

이 질문에 대한 답이 surface마다 다르다.

## 그래서 AGENTS.md로 넣은 Caveman은 무엇이었나?

사용자는 이전에 전역 `AGENTS.md` 또는 현재 세션의 AGENTS instructions로 Caveman 규칙을 넣었다.

예를 들면 이런 내용이다.

```text
Respond terse like smart caveman.
Drop filler, pleasantries, and hedging.
Preserve exact technical terms, code, API names, commands.
Stay active across turns.
```

단, 여기서 “across turns”는 instruction이 context에 남아 있는 동안 계속 따르라는 의미일 뿐이다. Hook처럼 상태 파일을 읽거나 매 prompt마다 reinforcement를 주입한다는 뜻은 아니다.

이 방식은 instruction file 방식이다.

```text
AGENTS.md
→ 모델에게 지시문으로 들어감
→ command 실행 아님
→ 상태 파일 없음
→ Settings > Hooks에 뜨지 않음
```

그래서 “Caveman이 적용된 것처럼 말한다”와 “Caveman hook이 실행된다”는 다르다.

```text
AGENTS.md Caveman
→ 말투 규칙이 context에 들어감

Caveman Skill
→ 필요할 때 skill 본문이 context에 들어감

Caveman Hook
→ lifecycle event에서 JS command가 실행되어 state/context를 관리함
```

이 셋은 결과가 비슷해 보일 수 있다.

하지만 하네스 관점에서는 완전히 다른 표면이다.

## Hook으로 만들려면 무엇이 필요할까?

만약 Caveman을 Ponytail처럼 Codex Settings > Hooks에 뜨게 만들고 싶다면, 핵심은 hook code를 새로 쓰는 것이 아니다.

이미 Caveman에는 hook code가 있다.

필요한 것은 Codex plugin distribution에서 그 hook definition을 노출하는 일이다.

개념적으로는 이런 방향이다.

```text
plugins/caveman/.codex-plugin/plugin.json
→ hooks 필드 추가

plugins/caveman/hooks/...
→ Codex가 실행할 hook definition 배치

hook command
→ Codex 환경에서 올바른 path/env/output contract 사용
```

하지만 여기서 바로 복붙하면 위험하다.

Caveman의 기존 hook은 Claude Code 환경을 강하게 가정한다.

예를 들어:

```text
CLAUDE_PLUGIN_ROOT
CLAUDE_CONFIG_DIR
~/.claude/.caveman-active
Claude Code statusLine
Claude Code hook output behavior
```

Codex plugin hook으로 제대로 배포하려면 다음을 다시 확인해야 한다.

```text
Codex에서 plugin root를 어떤 env var로 제공하는가?
Codex에서 plugin data directory는 어디인가?
SessionStart/UserPromptSubmit output contract는 무엇인가?
plain text stdout과 JSON additionalContext 중 무엇을 써야 하는가?
상태 파일은 어디에 써야 안전한가?
review/trust UI에 어떤 command가 표시되는가?
timeout은 충분히 짧은가?
Windows command도 필요한가?
```

Ponytail은 이 부분을 Codex branch로 처리했다.

```text
PLUGIN_DATA 감지
→ Codex용 state path 사용
→ Codex용 hook output JSON 작성
```

Caveman도 같은 수준의 Codex adapter가 필요하다.

이것이 하네스 엔지니어링이다.

```text
좋은 프롬프트를 쓰는 것만으로는 부족하다.
runtime surface에 맞게 배포하고 연결해야 한다.
```

## 흔한 오해

### 오해 1. “GitHub repository를 plugin으로 넣으면 모든 파일이 활성화된다”

아니다.

Plugin install은 repository 전체를 무조건 실행하는 과정이 아니다.

Codex는 manifest와 정해진 discovery rule을 보고 노출할 surface를 결정한다.

```text
skills가 선언됨
→ skills 발견

hooks가 선언됨
→ plugin-bundled hooks 발견

MCP/app 등이 선언됨
→ 해당 capability 발견
```

선언되지 않은 파일은 그냥 포함된 파일이다.

### 오해 2. “Caveman에 hook 코드가 있으니 Codex hook도 켜진 것이다”

아니다.

앞에서 본 것처럼 hook implementation은 가능성이고, hook registration은 manifest/config discovery path다. 코드 파일이 있어도 그 경로에 연결되지 않으면 lifecycle hook으로 실행되지 않는다.

### 오해 3. “Skill로 적용되는 것과 Hook으로 적용되는 것은 같다”

결과 말투만 보면 비슷할 수 있다.

하지만 mechanism은 다르다.

```text
Skill
→ 필요할 때 instruction 본문을 context에 넣음

Hook
→ lifecycle event마다 command를 실행함
→ 상태 파일을 읽고 쓸 수 있음
→ prompt 제출이나 tool 사용 전후에 개입할 수 있음
```

Skill은 문서 중심이고, Hook은 실행 중심이다.

### 오해 4. “AGENTS.md에 Caveman을 쓰면 항상 같은 효과다”

부분적으로만 맞다.

AGENTS.md에 Caveman 규칙을 넣으면 말투 지시문은 들어간다. 그래서 모델 응답이 짧아질 수 있다.

하지만 다음은 자동으로 생기지 않는다.

```text
/caveman lite command parsing
.caveman-active state file
statusline badge
UserPromptSubmit per-turn reinforcement
Settings > Hooks review/trust
```

AGENTS.md는 좋은 폴백이다. 하지만 hook runtime은 아니다.

### 오해 5. “Hooks에 안 뜨면 Caveman plugin이 설치 실패한 것이다”

아니다.

현재 확인한 상태에서는 Caveman skill들은 설치되어 있다.

문제는 “설치 실패”가 아니라 “Codex용 plugin manifest가 hooks를 노출하지 않는 배포 형태”에 가깝다.

즉:

```text
skill plugin으로는 정상
lifecycle hook plugin으로는 미노출
```

이렇게 보는 것이 정확하다.

## 왜 중요한가

이 장은 Ponytail보다 더 중요할 수도 있다.

Ponytail은 “잘 연결된 사례”다.

Caveman은 “코드는 있지만 현재 surface에 연결되지 않은 사례”다.

하네스 엔지니어링에서는 둘 다 봐야 한다.

```text
무엇이 있는가?
무엇이 발견되는가?
무엇이 실행되는가?
무엇이 context에 들어가는가?
무엇이 상태를 저장하는가?
무엇이 사용자 UI에 노출되는가?
```

이 질문을 못 하면 계속 헷갈린다.

```text
설치했는데 왜 안 뜨지?
파일 있는데 왜 실행 안 되지?
AGENTS.md랑 Hook이 뭐가 다르지?
Skill 켰는데 왜 mode가 유지 안 되지?
```

이 질문들의 답은 보통 모델 안에 있지 않다.

답은 harness의 discovery path에 있다.

```text
manifest
config layer
hook declaration
skill directory
state file
runtime env
output contract
trust/review
```

AX 개발자는 이 경로를 읽을 수 있어야 한다.

## 요약

```text
1. 같은 Caveman behavior도 AGENTS.md, Skill, Hook, Plugin surface에 따라 다르게 작동한다.
2. 하네스 엔지니어링의 핵심은 파일 존재가 아니라 discovery path, execution path, state path, output contract다.
3. 현재 Codex용 Caveman manifest에는 skills는 있지만 hooks 필드가 없다.
4. 그래서 Caveman은 skill surface로는 발견되지만, plugin-bundled lifecycle hook으로는 발견되지 않는다.
5. Caveman repository에는 Claude용 hook manifest와 src/hooks 구현이 있다.
6. 하지만 Claude adapter, Codex skill adapter, Codex hook adapter는 같은 것이 아니다.
7. .codex/hooks.json은 project-local config 성격이지, 자동 plugin-bundled hook이 아니다.
8. Ponytail은 lifecycle에 연결된 behavior이고, Caveman은 현재 Codex distribution에서는 skill surface까지만 연결된 behavior다.
```

## 생각해볼 질문

1. 어떤 파일이 “존재한다”는 것과 runtime이 “발견한다”는 것을 어떻게 구분할 수 있을까?
2. 내가 만든 plugin에서 Codex가 발견해야 하는 surface는 skills인가, hooks인가, MCP인가?
3. Claude Code용 plugin manifest와 Codex용 plugin manifest를 같은 것으로 착각하면 어떤 문제가 생길까?
4. AGENTS.md로 충분한 규칙과 Hook이 필요한 mode는 어떻게 나눌 수 있을까?
5. 내 agent가 상태를 유지해야 한다면 그 상태는 어디에 저장되고, 언제 다시 context로 들어와야 할까?
