[← README](../README.md)

# 4. Codex 지시문 표면: AGENTS.md, Skills, Hooks, Plugins

처음엔 전부 “프롬프트 넣는 방법”처럼 보인다. 하지만 각각 쓰임이 다르다.

```text
AGENTS.md = durable project guidance
Skill = on-demand workflow manual
Hook = lifecycle command
Plugin = distributable package
```

## AGENTS.md

`AGENTS.md`는 세션 시작 때 로드되는 지속 지시문이다.

공식 Codex manual 기준으로 Codex는 다음 순서로 guidance를 찾는다.

```text
1. ~/.codex/AGENTS.override.md
   또는 ~/.codex/AGENTS.md

2. project root부터 현재 작업 디렉토리까지
   AGENTS.override.md
   AGENTS.md
   configured fallback filenames

3. root 쪽 파일이 먼저 들어가고,
   현재 폴더에 가까운 파일이 나중에 들어간다.
```

좋은 용도:

```text
repo 규칙
테스트 명령
코딩 컨벤션
리뷰 기준
팀 작업 방식
```

중요한 정정:

```text
AGENTS.md는 항상 유효한 규칙을 담기에 좋다.
하지만 매 프롬프트마다 파일을 다시 읽는 장치는 아니다.
```

## Skill

Skill은 필요할 때 읽히는 작업 매뉴얼이다.

Codex는 처음부터 모든 `SKILL.md`를 context에 넣지 않는다. 먼저 skill의 name, description, path 같은 목록을 알고 있다가, 작업에 맞는 skill이 선택되면 그때 전체 `SKILL.md`를 읽는다. 공식 문서는 이 방식을 progressive disclosure라고 설명한다.

좋은 용도:

```text
PDF 생성 절차
PR 리뷰 방식
데이터 분석 리포트 작성법
특정 framework 작업법
```

## Hook

Hook은 특정 lifecycle event마다 실행되는 command다.

```text
SessionStart
UserPromptSubmit
PreToolUse
PostToolUse
Stop
```

좋은 용도:

```text
매 프롬프트마다 상태 확인
위험한 tool call 차단
세션 시작 때 동적 context 주입
slash command 감지
상태 파일 읽기/쓰기
```

Hook은 문서가 아니라 실행이다. 그래서 trust/review가 필요하다.

## Plugin

Plugin은 배포 단위다.

```text
skills/
hooks/
MCP config
assets
app mappings
manifest
```

내가 혼자 쓰는 작업법이면 skill이나 AGENTS.md로 충분할 수 있다. 남이 설치하게 만들고 싶거나 hooks/MCP/assets를 함께 묶고 싶다면 plugin이 맞다.

## 선택 기준

```text
항상 유효한 정적 규칙 → AGENTS.md
특정 작업의 절차 → Skill
이벤트마다 검사/주입/차단 → Hook
남에게 설치 가능한 제품 → Plugin
외부 데이터/액션 연결 → MCP/App
```

