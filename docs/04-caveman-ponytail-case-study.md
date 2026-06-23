[← README](../README.md)

# 9. Ponytail 로컬 설치 상태

확인한 Ponytail manifest:

```text
/Users/jhkim/.codex/plugins/cache/ponytail/ponytail/4.7.0/.codex-plugin/plugin.json
```

핵심 필드:

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

즉 Ponytail은 Codex plugin manifest에 `hooks`를 공식 등록했다.

Hook file:

```text
/Users/jhkim/.codex/plugins/cache/ponytail/ponytail/4.7.0/hooks/claude-codex-hooks.json
```

등록 이벤트:

```text
SessionStart
→ node "${CLAUDE_PLUGIN_ROOT}/hooks/ponytail-activate.js"

UserPromptSubmit
→ node "${CLAUDE_PLUGIN_ROOT}/hooks/ponytail-mode-tracker.js"
```

따라서 Codex Settings > Hooks에 Ponytail이 뜬다.

Ponytail 동작:

```text
SessionStart
→ ponytail-activate.js 실행
→ active mode 결정
→ .ponytail-active 상태 파일 기록
→ ponytail instruction 생성
→ stdout으로 Codex에 additionalContext 전달

UserPromptSubmit
→ ponytail-mode-tracker.js 실행
→ /ponytail, /ponytail ultra, normal mode 같은 명령 감지
→ 상태 파일 수정
→ 필요하면 mode changed context 출력
```

Ponytail이 모델에게 주입하려는 핵심 행동:

```text
- 가장 작은 구현
- YAGNI
- stdlib/native 우선
- 불필요한 추상화 금지
- 새 dependency 피하기
- 짧은 diff
- 하지만 문제 이해는 생략하지 않기
```

---

## 10. Caveman 로컬 설치 상태

확인한 Caveman Codex plugin manifest:

```text
/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/plugins/caveman/.codex-plugin/plugin.json
```

핵심:

```json
{
  "name": "caveman",
  "version": "0.1.0",
  "skills": "./skills/",
  "interface": {
    "capabilities": ["Write"]
  }
}
```

여기에는 `hooks` 필드가 없다.

하지만 Caveman repo 안에는 hook 관련 파일이 존재한다.

```text
/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/.codex/hooks.json
/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/.claude-plugin/plugin.json
/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/src/hooks/caveman-activate.js
/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/src/hooks/caveman-mode-tracker.js
```

즉:

```text
Caveman은 hook 코드가 있다.
하지만 현재 Codex plugin manifest가 그 hook을 가리키지 않는다.
그래서 Codex Settings > Hooks에 plugin-owned hook으로 뜨지 않는다.
```

이전 세션에서 `~/.codex/AGENTS.md`에 Caveman식 규칙을 전역으로 넣었지만, 그건 hook이 아니다.

```text
~/.codex/AGENTS.md
→ 새 Codex 세션 시작 때 로드될 전역 지시문
→ Settings > Hooks에는 안 뜸
→ command 실행 없음
→ trust review UI 없음
```

그래서 “Caveman이 완전히 hook 기반으로 적용됐다”와는 다르다.

---

## 11. 왜 Ponytail은 Hooks에 뜨고 Caveman은 안 뜨는가

정리:

```text
Ponytail
→ .codex-plugin/plugin.json에 "hooks": "./hooks/claude-codex-hooks.json" 있음
→ Codex가 plugin-bundled hook으로 발견
→ Settings > Hooks에 뜸
→ review/trust 필요

Caveman
→ Codex plugin manifest에 "hooks" 없음
→ hook 코드와 Claude plugin manifest는 있지만 Codex plugin packaging에 연결 안 됨
→ Settings > Hooks에 안 뜸
```

중요한 구분:

```text
코드가 repo 안에 존재한다
≠
Codex plugin으로 등록되어 실행된다
```

Codex가 실행하려면 discovery surface에 걸려야 한다.

```text
~/.codex/hooks.json에 있거나
project/.codex/hooks.json에 있거나
plugin manifest가 hooks 파일을 가리켜야 함
```

---

## 12. `AGENTS.md`와 Hook의 실전 차이

둘 다 최종적으로는 모델에게 instruction/context를 준다. 그래서 “그게 그거 아닌가?”라는 느낌은 맞다.

하지만 차이는 크다.

| 구분 | `AGENTS.md` | Hook |
|---|---|---|
| 로드 시점 | 세션 시작 instruction chain 구성 시 | 이벤트 발생 시 |
| 반복성 | 기본적으로 한 번 로드 | 이벤트마다 실행 가능 |
| 동적 상태 | 약함 | 강함 |
| 명령 실행 | 없음 | 있음 |
| user prompt 검사 | 못 함 | 가능 |
| tool 사용 전후 검사 | 못 함 | 가능 |
| 승인 UI | 없음 | command hook은 trust 필요 |
| 좋은 용도 | repo 규칙, 기본 작업 방식 | 모드 전환, 검사, 차단, 동적 주입 |

예:

```text
"이 repo는 pnpm을 쓴다"
→ AGENTS.md

"/ponytail ultra라고 말하면 이후 모드를 ultra로 바꿔라"
→ Hook + 상태 파일

"Bash 실행 전에 위험 명령인지 검사해라"
→ PreToolUse hook

"매 프롬프트마다 brevity mode reminder를 넣어라"
→ UserPromptSubmit hook
```

---

## 13. 왜 Caveman/Ponytail 같은 모드는 Hook이 더 드라마틱한가

응답 스타일/행동 모드는 대화 중 흔들리기 쉽다.

예:

```text
- 유저가 긴 설명을 요구함
- 다른 plugin이 더 최근 instruction을 주입함
- 세션이 길어짐
- context compaction 발생
- mode 변경 명령이 들어옴
```

`AGENTS.md`로도 어느 정도 가능하다. 하지만 hook을 쓰면 더 강하다.

```text
SessionStart에서 강한 instruction 주입
+ UserPromptSubmit에서 매 턴 상태 확인
+ 상태 파일로 현재 mode 유지
+ slash command로 mode 변경
+ 필요하면 매 턴 짧은 reminder 재주입
```

그래서 같은 “문구”라도 효과가 다르다.

```text
AGENTS.md
→ 세션 기본 instruction

Hook
→ lifecycle마다 새로 실행되는 자동 장치
```

---
