[← README](../README.md)

# 17. 이번 세션에서 정정된 중요한 표현

이전 설명 중 모호했던 표현:

```text
AGENTS.md는 초반에 들어간 지시가 긴 대화 속에서 희미해질 수 있다.
```

더 정확한 표현:

```text
AGENTS.md는 세션 시작 때 로드되어 지속 지시문이 된다.
다만 매 이벤트마다 동적으로 재주입되는 Hook reminder보다
실제 모델 행동에 대한 즉각적 영향은 약해질 수 있다.
```

또 다른 표현:

```text
AGENTS.md = 항상 지켜야 하는 규칙
```

더 정확한 표현:

```text
AGENTS.md = 항상 유효한 정적/지속 규칙을 담기에 적합한 위치.
하지만 항상 다시 실행되는 장치는 아니다.
```

---

## 18. 레퍼런스

공식 문서:

- Codex manual - Skills: https://developers.openai.com/codex/skills
- Codex manual - AGENTS.md guidance: https://developers.openai.com/codex/guides/agents-md
- Codex manual - Hooks: https://developers.openai.com/codex/hooks
- Codex manual - Build plugins: https://developers.openai.com/codex/plugins/build
- Codex manual - MCP: https://developers.openai.com/codex/mcp

이번 세션에서 확인한 로컬 파일:

- `/Users/jhkim/.codex/plugins/cache/ponytail/ponytail/4.7.0/.codex-plugin/plugin.json`
- `/Users/jhkim/.codex/plugins/cache/ponytail/ponytail/4.7.0/skills/ponytail/SKILL.md`
- `/Users/jhkim/.codex/plugins/cache/ponytail/ponytail/4.7.0/hooks/claude-codex-hooks.json`
- `/Users/jhkim/.codex/plugins/cache/ponytail/ponytail/4.7.0/hooks/ponytail-activate.js`
- `/Users/jhkim/.codex/plugins/cache/ponytail/ponytail/4.7.0/hooks/ponytail-mode-tracker.js`
- `/Users/jhkim/.codex/plugins/cache/ponytail/ponytail/4.7.0/hooks/ponytail-runtime.js`
- `/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/plugins/caveman/.codex-plugin/plugin.json`
- `/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/skills/caveman/SKILL.md`
- `/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/.codex/hooks.json`
- `/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/.claude-plugin/plugin.json`
- `/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/src/hooks/caveman-activate.js`
- `/Users/jhkim/.codex/plugins/cache/caveman/caveman/local/src/hooks/caveman-mode-tracker.js`

---

## 19. 앞으로 보강할 질문들

아직 더 파고들 만한 질문:

```text
1. hook stdout JSON schema를 정확히 어디까지 쓸 수 있는가?
2. decision: block 같은 hook output은 Codex에서 어떤 이벤트에 먹히는가?
3. UserPromptSubmit hook이 prompt를 어떻게 stdin으로 받는가?
4. plugin manifest의 전체 schema는 무엇인가?
5. plugin marketplace는 어떻게 구성하는가?
6. local plugin 개발 후 Codex app에서 어떻게 테스트/공유하는가?
7. Skill description은 implicit invocation에 얼마나 중요하고 어떻게 써야 하는가?
8. AGENTS.md, developer instruction, skill instruction, hook additionalContext의 실제 우선순위는 어떻게 체감되는가?
9. MCP 서버를 plugin에 포함시키면 어떤 구조가 되는가?
10. 안전한 hook 작성 패턴은 무엇인가?
```

이 문서는 계속 업데이트용이다. 이후 질문에서 “문서에 보강해줘”라고 하면, 이 파일에 새 섹션을 추가하는 식으로 이어가면 된다.
