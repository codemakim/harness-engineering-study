[← README](../README.md)

# 6. Hook은 무엇인가

Hook은 Codex의 agentic loop 중 특정 이벤트에 실행되는 스크립트다.

공식 manual 기준 주요 이벤트:

```text
SessionStart
UserPromptSubmit
PreToolUse
PostToolUse
PermissionRequest
PreCompact
PostCompact
Stop
SubagentStart
SubagentStop
```

감각적으로는 React lifecycle hook과 비슷하게 생각해도 된다.

```text
세션 시작됨 → SessionStart hook
유저가 프롬프트 보냄 → UserPromptSubmit hook
툴 쓰기 전 → PreToolUse hook
툴 쓴 후 → PostToolUse hook
턴 끝남 → Stop hook
```

Hook 위치:

```text
~/.codex/hooks.json
~/.codex/config.toml 안의 [hooks]

project/.codex/hooks.json
project/.codex/config.toml 안의 [hooks]

plugin manifest가 가리키는 hooks 파일
plugin/hooks/hooks.json 같은 기본 위치
```

Hook은 명령을 실행한다.

예:

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

현재 공식 manual 기준으로 실제 실행되는 handler type은 `command`다. `prompt`, `agent` handler는 파싱되지만 아직 실행되지 않는다고 되어 있다.

---

## 7. Hook trust/승인 모델

Hook은 임의 명령을 실행할 수 있다. 그래서 Codex는 non-managed command hook을 실행하기 전에 사용자의 review/trust를 요구한다.

핵심:

```text
1. Codex가 configured hooks를 발견함
2. 실행 전에 hook definition을 보여줌
3. 사용자가 review/trust 해야 실행됨
4. trust는 hook의 현재 hash 기준으로 저장됨
5. hook 내용이 바뀌면 새 hash가 되므로 다시 review 필요
```

그래서 Ponytail hook이 Settings > Hooks에 뜨고, “검토 후 승인” UI가 나온다.

이건 귀찮은 절차가 아니라 보안 모델이다. Hook은 `node`, `python`, `bash` 같은 명령을 실행할 수 있으므로 신뢰 확인이 필요하다.

---

## 8. Hook이 LLM과 연결되는 원리

중요:

```text
Hook JS가 LLM API를 직접 호출하는 것이 아니다.
Hook JS는 stdout으로 Codex가 읽을 출력을 만든다.
Codex가 그 출력을 모델 context나 hook decision으로 해석한다.
```

Ponytail의 핵심 runtime 코드 개념:

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

즉 흐름은 이렇다.

```text
hook event 발생
→ Codex가 command 실행
→ JS가 stdout으로 JSON/text 출력
→ Codex가 stdout을 해석
→ additionalContext가 모델 입력에 추가됨
→ 모델 행동 변화
```

이게 “하네스” 느낌이 나는 지점이다. 모델 자체를 바꾸는 게 아니라, 모델 주변 실행환경이 모델에게 들어가는 context를 제어한다.

---
