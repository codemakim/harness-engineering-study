[← README](../README.md)

# 7. Case study: Caveman은 왜 Hooks에 안 떴나

Caveman은 hook 코드가 있다. 그런데 현재 Codex Settings > Hooks에는 plugin-owned hook으로 뜨지 않았다.

이 차이가 하네스 엔지니어링에서 아주 중요하다.

```text
코드가 repo 안에 존재한다
≠
runtime이 그 코드를 발견해서 실행한다
```

## 로컬에서 확인한 Codex manifest

파일:

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

## 하지만 hook 코드는 존재한다

확인한 파일:

```text
/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/.codex/hooks.json
/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/.claude-plugin/plugin.json
/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/src/hooks/caveman-activate.js
/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/src/hooks/caveman-mode-tracker.js
```

Caveman의 Claude plugin manifest에는 hook 설정이 있다. repo-local `.codex/hooks.json`도 있다. 하지만 현재 설치된 Codex plugin manifest가 그 hook 파일을 가리키지 않았다.

## 그래서 생긴 결과

```text
Ponytail
→ .codex-plugin/plugin.json에 "hooks" 있음
→ Codex가 plugin hook으로 discovery
→ Settings > Hooks에 표시

Caveman
→ .codex-plugin/plugin.json에 "hooks" 없음
→ hook 코드는 있지만 Codex plugin packaging에 연결 안 됨
→ Settings > Hooks에 표시되지 않음
```

## AGENTS.md로 넣은 Caveman 규칙은 무엇인가

이전 세션에서 `~/.codex/AGENTS.md`에 Caveman식 전역 지시를 넣었다.

이건 hook이 아니다.

```text
~/.codex/AGENTS.md
→ 새 세션 시작 때 로드될 durable instruction
→ command 실행 없음
→ Settings > Hooks에 안 뜸
→ trust review UI 없음
```

즉 “Caveman 스타일의 전역 지시”와 “Caveman hook plugin”은 다르다.

## 핵심 교훈

하네스는 정해진 위치만 본다.

```text
~/.codex/hooks.json
project/.codex/hooks.json
plugin manifest의 hooks 필드
plugin 기본 hooks/hooks.json
```

실행되길 원하면 discovery surface에 연결해야 한다.

