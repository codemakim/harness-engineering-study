[← README](../README.md)

# 6. Case study: Ponytail은 왜 강하게 작동하는가

Ponytail은 “짧게 답해”라는 단순 프롬프트보다 강하게 느껴진다. 이유는 문구가 멋져서만이 아니다. Codex lifecycle에 붙어 있기 때문이다.

## 로컬에서 확인한 manifest

파일:

```text
/Users/jhkim/.codex/plugins/cache/ponytail/ponytail/4.7.0/.codex-plugin/plugin.json
```

핵심:

```json
{
  "name": "ponytail",
  "version": "4.7.0",
  "skills": "./skills/",
  "hooks": "./hooks/claude-codex-hooks.json",
  "interface": {
    "capabilities": ["Instructions", "Lifecycle hooks"]
  }
}
```

`hooks` 필드가 있다. 그래서 Codex가 plugin-bundled hook으로 발견하고 Settings > Hooks에 보여준다.

## Hook file

파일:

```text
/Users/jhkim/.codex/plugins/cache/ponytail/ponytail/4.7.0/hooks/claude-codex-hooks.json
```

핵심 이벤트:

```text
SessionStart
→ node "${CLAUDE_PLUGIN_ROOT}/hooks/ponytail-activate.js"

UserPromptSubmit
→ node "${CLAUDE_PLUGIN_ROOT}/hooks/ponytail-mode-tracker.js"
```

## 동작 흐름

```text
SessionStart
→ ponytail-activate.js 실행
→ default mode 확인
→ .ponytail-active 상태 파일 기록
→ Ponytail instruction 생성
→ stdout으로 additionalContext 출력

UserPromptSubmit
→ ponytail-mode-tracker.js 실행
→ /ponytail, /ponytail ultra, normal mode 감지
→ 상태 파일 수정
→ 필요하면 mode changed context 출력
```

## 왜 드라마틱한가

Ponytail은 그냥 “한 번 말한 스타일 지시”가 아니다.

```text
세션 시작 때 instruction 주입
+ 매 prompt 때 mode command 감지
+ 상태 파일로 mode 유지
+ hook output으로 모델 context 보강
```

그래서 대화가 길어지거나 사용자가 mode를 바꾸어도 runtime이 따라간다.

## 핵심 교훈

```text
좋은 에이전트 행동은 프롬프트 문장만으로 만들어지지 않는다.
언제, 어디서, 얼마나 반복해서 context에 들어가는지가 중요하다.
```

