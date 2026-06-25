[← README](../README.md)

# 5. Hooks lifecycle: 대화 주기에 코드를 꽂는다는 것

Hook은 “프롬프트 파일”이 아니다. 대화와 도구 사용의 특정 순간에 실행되는 command다.

React lifecycle hook처럼 생각해도 시작점으로는 괜찮다. 하지만 Codex hook은 UI component가 아니라 agent runtime에 붙는다.

## 주요 이벤트

```text
SessionStart
→ 세션 시작, 재개, clear, compact 이후

UserPromptSubmit
→ 사용자가 프롬프트를 보낼 때

PreToolUse
→ tool 실행 전

PostToolUse
→ tool 실행 후

PermissionRequest
→ 권한 요청 시점

Stop
→ 턴 종료 시점
```

## Hook config 위치

Codex는 active config layer 주변에서 hook을 찾는다.

```text
~/.codex/hooks.json
~/.codex/config.toml
project/.codex/hooks.json
project/.codex/config.toml
plugin manifest가 가리키는 hooks 파일
plugin의 기본 hooks/hooks.json
```

## 기본 shape

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.codex/hooks/session_start.py",
            "statusMessage": "Loading session notes"
          }
        ]
      }
    ]
  }
}
```

현재 공식 manual 기준 실제 실행되는 handler type은 `command`다. `prompt`, `agent` handler는 파싱되지만 실행되지 않는다고 되어 있다.

## stdout이 연결 통로다

Hook command는 stdout으로 Codex에 결과를 전달한다.

Ponytail runtime의 핵심 개념:

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

흐름:

```text
hook event 발생
→ command 실행
→ JS/Python/Bash가 stdout 출력
→ Codex가 stdout 해석
→ additionalContext가 모델 입력에 추가될 수 있음
```

## Trust/review

Hook은 임의 command를 실행할 수 있다. 그래서 Codex는 non-managed command hook을 실행하기 전에 review/trust를 요구한다.

```text
hook 발견
→ 사용자가 정의 검토
→ trust 승인
→ 현재 hash 기준으로 신뢰 저장
→ hook 내용이 바뀌면 다시 review 필요
```

이 보안 모델 때문에 Ponytail hook은 Settings > Hooks에 뜨고, 사용자의 승인을 기다린다.

