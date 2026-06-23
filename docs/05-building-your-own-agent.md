[← README](../README.md)

# 14. Plugin은 무엇인가

Plugin은 기능 자체라기보다 배포 단위다.

공식 manual 기준:

```text
Skill은 workflow authoring format.
Plugin은 reusable skills/apps/hooks/MCP config 등을 설치 가능하게 묶는 distribution unit.
```

최소 plugin 구조:

```text
my-plugin/
  .codex-plugin/
    plugin.json
  skills/
    hello/
      SKILL.md
```

Hook까지 포함하려면:

```text
my-plugin/
  .codex-plugin/
    plugin.json
  skills/
    my-mode/
      SKILL.md
  hooks/
    hooks.json
    activate.js
    mode-tracker.js
```

`plugin.json`:

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "My Codex behavior plugin",
  "skills": "./skills/",
  "hooks": "./hooks/hooks.json"
}
```

이렇게 해야 Codex가 plugin-bundled hook으로 발견할 수 있다.

---

## 15. 직접 만들 때의 첫 번째 미니 프로젝트

처음부터 거대한 에이전트를 만들 필요 없다. 가장 작은 실험은 이거다.

```text
mini-mode-plugin/
  .codex-plugin/
    plugin.json
  skills/
    mini-mode/
      SKILL.md
  hooks/
    hooks.json
    activate.js
    mode-tracker.js
```

목표:

```text
1. SessionStart 때 "MINI MODE ACTIVE" context 주입
2. UserPromptSubmit 때 "/mini off", "/mini full" 감지
3. .mini-active 상태 파일에 mode 저장
4. active면 매 유저 prompt마다 짧은 reminder 출력
5. Codex Settings > Hooks에 뜨고 review/trust 가능하게 만들기
```

여기까지 만들면 Ponytail/Caveman류의 핵심 원리를 거의 다 만진다.

---

## 16. 하네스 엔지니어링 체크리스트

무언가를 만들 때 항상 이 질문을 한다.

```text
1. 이 instruction은 어디서 발견되는가?
2. 언제 로드/실행되는가?
3. 매 턴 반복되는가, 세션 시작 한 번인가?
4. 모델 context에 들어가는가, UI에만 보이는가?
5. 상태는 어디 저장되는가?
6. 사용자가 mode를 바꿀 수 있는가?
7. hook trust/review가 필요한가?
8. 실패하면 조용히 무시되는가, 턴을 막는가?
9. 보안상 임의 명령 실행 위험은 없는가?
10. 남에게 배포하려면 plugin manifest가 올바른가?
```

이 질문들이 “좋은 프롬프트”와 “좋은 에이전트 하네스”를 가른다.

---
