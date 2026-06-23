[← README](../README.md)

# 1. 지금까지의 핵심 질문

지금까지 질문의 흐름은 대략 이랬다.

1. 현재 세션에 `caveman`, `ponytail`이 적용되어 있는가?
2. 왜 `ponytail`은 Codex Settings > Hooks에 뜨는데 `caveman`은 안 뜨는가?
3. Hook이 뭔가? 왜 사용자가 검토 후 승인해야 하는가?
4. `AGENTS.md`와 Hook은 둘 다 모델 컨텍스트에 문구를 넣는 것 같은데, 실제 차이는 뭔가?
5. `AGENTS.md`가 “항상 지켜야 하는 규칙”이라면 왜 hook처럼 매번 다시 넣지 않는가?
6. 나도 이런 에이전트/플러그인을 만들려면 무엇을 배워야 하는가?

이 질문들은 전부 “프롬프트를 잘 쓰는 법”보다 한 층 낮은 문제다. 즉, 모델이 어떤 지시문을 언제, 어디서, 어떤 우선순위와 반복성으로 받게 되는지 설계하는 문제다. 이걸 여기서는 편의상 하네스 엔지니어링이라고 부른다.

---

## 2. 제일 중요한 정신모델

Codex 확장 표면은 이렇게 나눠서 보면 된다.

```text
AGENTS.md
= 세션 시작 때 로드되는 지속 지시문
= repo/global 기본 헌법

Skill
= 필요할 때 꺼내 쓰는 작업 매뉴얼
= Codex가 description을 보고 선택한 뒤 SKILL.md 전체를 읽음

Hook
= 세션/대화/툴 라이프사이클 이벤트마다 실행되는 자동 장치
= stdout으로 context, decision, block reason 등을 Codex에 전달할 수 있음

Plugin
= skills/hooks/MCP/assets 등을 남이 설치할 수 있게 포장한 배포 단위

MCP/App/Connector
= 외부 데이터나 외부 액션을 모델에게 도구로 연결하는 계층
```

짧게:

```text
항상 유효한 정적 규칙 → AGENTS.md
필요할 때만 쓰는 작업법 → Skill
특정 순간마다 실행/검사/주입 → Hook
남에게 설치시키고 싶음 → Plugin
외부 시스템과 연결해야 함 → MCP/App
```

---

## 3. `AGENTS.md`는 언제, 어디서 로드되는가

Codex는 시작할 때 instruction chain을 만든다. 이때 `AGENTS.md` 계열 파일을 읽는다.

공식 manual 기준 discovery 순서:

```text
1. Global scope
   ~/.codex/AGENTS.override.md
   또는
   ~/.codex/AGENTS.md

2. Project scope
   project root부터 현재 작업 디렉토리까지 내려오며
   각 디렉토리에서:
   AGENTS.override.md
   AGENTS.md
   fallback filenames

3. Merge order
   root 쪽 파일이 먼저 들어가고,
   현재 작업 디렉토리에 가까운 파일이 나중에 들어감.
   나중 지시가 더 구체적인 override 역할을 함.
```

예:

```text
~/.codex/AGENTS.md
/my-repo/AGENTS.md
/my-repo/apps/web/AGENTS.md
/my-repo/apps/web/components/AGENTS.override.md
```

`/my-repo/apps/web/components`에서 Codex를 시작하면 대략 이렇게 합쳐진다.

```text
global AGENTS
→ repo root AGENTS
→ apps/web AGENTS
→ components AGENTS.override
```

중요한 정정:

```text
AGENTS.md는 “항상 지켜야 하는 규칙”을 담는 곳이 맞다.
하지만 “매 프롬프트마다 파일을 다시 읽는 장치”는 아니다.
```

즉:

```text
AGENTS.md = durable instruction
Hook = lifecycle automation
```

`AGENTS.md`는 세션 시작 때 로드되어 지속 지시문이 된다. 하지만 매 턴 동적으로 다시 실행되지는 않는다. 그래서 세션 중간에 `~/.codex/AGENTS.md`를 새로 만들거나 수정해도 현재 세션에 즉시 반영된다고 기대하면 안 된다. 새 세션/재시작이 필요할 수 있다.

---

## 4. “항상 지켜야 하는 규칙”인데 왜 희석될 수 있나

여기서 헷갈렸던 표현:

> AGENTS.md는 항상 지켜야 하는 규칙이라면서 왜 희석된다고 말하는가?

정확한 답:

```text
규칙상 계속 유효한 것과,
모델 행동에서 매 턴 강하게 눈앞에 놓이는 것은 다르다.
```

`AGENTS.md`는 세션 초기에 instruction chain에 들어간다. 규칙상 계속 유효하다. 하지만 긴 대화에서는 다음 이유로 실제 행동 영향이 약해질 수 있다.

```text
1. 새 사용자 지시와 새 맥락이 계속 쌓임
2. 대화가 길어져 compaction/요약이 발생할 수 있음
3. 다른 plugin/hook/developer instruction이 더 최근 context로 들어올 수 있음
4. LLM은 법률 엔진이 아니라 attention 기반 모델이라,
   오래된 일반 지시보다 최근의 구체적 지시가 더 강하게 작동할 수 있음
```

그래서 `AGENTS.md`는 프로젝트 규칙, 명령, 스타일, 검증 방식 같은 “기본 헌법”에 적합하다. 반면 `caveman`이나 `ponytail`처럼 응답 스타일/행동 모드를 강하게 유지하려면 Hook이 더 효과적일 수 있다.

---

## 5. Skill은 `AGENTS.md`와 어떻게 다른가

Skill은 작업별 매뉴얼이다. 항상 전체 내용이 context에 들어가는 게 아니다.

공식 manual 기준:

```text
Codex는 처음에 skill의 name, description, file path 정도만 context에 둔다.
작업이 skill description과 맞거나 사용자가 명시적으로 호출하면,
그때 SKILL.md 전체를 읽는다.
```

이 방식을 progressive disclosure라고 한다. 목적은 context 절약이다.

Skill 위치:

```text
repo/.agents/skills/<skill-name>/SKILL.md
~/.agents/skills/<skill-name>/SKILL.md
/etc/codex/skills/<skill-name>/SKILL.md
Codex system bundled skills
plugin 안의 skills/
```

예:

```text
ponytail plugin
→ skills/ponytail/SKILL.md

caveman plugin
→ skills/caveman/SKILL.md
```

Skill이 좋은 경우:

```text
- “PR 리뷰를 이렇게 해라”
- “PDF 만들 때 이 절차를 따라라”
- “데이터 분석 리포트는 이 구조로 만들어라”
- “이 특정 작업을 할 때만 이 workflow를 써라”
```

항상 켜져 있는 성격 모드보다는, 특정 작업에서 재사용되는 workflow에 강하다.

---
