[← README](../README.md)

# 5장. Hook은 왜 lifecycle에 꽂히는가?

## 이 장에서 배울 것

4장에서는 Hook을 이렇게 정리했다.

```text
Hook = lifecycle command
```

이 장에서는 그 말을 더 깊게 푼다.

Hook은 단순히 “더 강한 프롬프트”가 아니다. 대화와 tool 사용의 특정 순간에 실행되는 command다. 그래서 Hook을 이해하려면 문장보다 먼저 시점을 봐야 한다.

이 장의 중심 문장은 이것이다.

```text
Hook은 모델에게 무엇을 말할지보다,
언제 runtime에 개입할지를 설계하는 장치다.
```

## 먼저 직관으로 이해하기

웹 개발에서 middleware를 떠올려보자.

Controller에 도착하기 전에 이런 일이 일어날 수 있다.

```text
request 수신
→ auth 확인
→ rate limit 검사
→ body validation
→ logging
→ controller 실행
→ response logging
```

모든 로직을 controller 안에 넣을 수도 있다. 하지만 그러면 공통 검사와 반복 작업이 금방 지저분해진다. 그래서 특정 시점에 끼어드는 middleware나 event handler를 둔다.

Codex Hook도 비슷하다.

```text
session 시작
→ SessionStart hook

user prompt 제출
→ UserPromptSubmit hook

tool 실행 전
→ PreToolUse hook

tool 실행 후
→ PostToolUse hook

turn 종료
→ Stop hook
```

Hook은 “모델이 답하기 전에 참고할 문서”가 아니라, agent runtime의 특정 시점에 실행되는 작은 프로그램이다.

## 정확한 개념: event, matcher, handler

Codex Hook은 대략 세 층으로 생각할 수 있다.

```text
event
→ 언제 실행되는가?

matcher
→ 그 event 중 어떤 경우에 실행되는가?

handler
→ 무엇을 실행하는가?
```

예를 들어:

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

이 설정은 이렇게 읽는다.

```text
SessionStart event가 발생했을 때
startup 또는 resume에 match하면
python3 ~/.codex/hooks/session_start.py를 실행한다.
```

현재 공식 문서 기준으로 실제 실행되는 handler type은 `command`다. `prompt`, `agent` handler는 파싱될 수 있지만 실행되지 않는 것으로 설명되어 있다.

## 주요 lifecycle event

Hook은 event마다 쓰임이 다르다.

### SessionStart

세션 시작, 재개, clear, compact 이후처럼 thread/subagent 시작 범위에서 실행된다.

좋은 용도:

```text
세션 시작 안내 context 주입
프로젝트 상태 요약 로드
mode instruction 주입
작업 디렉터리 기반 설정 확인
```

Ponytail이 세션 시작 때 “PONYTAIL MODE ACTIVE” 같은 instruction을 주입하는 구조가 여기에 가깝다.

### UserPromptSubmit

사용자가 prompt를 제출할 때 실행된다.

좋은 용도:

```text
slash command 감지
mode 변경 감지
prompt logging
위험한 secret paste 검사
사용자 입력 기반 memory 후보 수집
```

단, 이 event의 matcher는 지원되지 않거나 무시될 수 있다. 즉 UserPromptSubmit은 “prompt 내용에 따라 matcher로 나눠 실행”하기보다, hook command 안에서 입력을 직접 읽고 판단하는 쪽으로 생각하는 게 안전하다.

### PreToolUse

Tool 실행 전에 실행된다.

좋은 용도:

```text
위험한 command guardrail
특정 tool 사용 전 정책 검사
쓰기 작업 전 승인 요구
파일 경로 제한 검사
```

여기서 중요한 점이 있다.

```text
PreToolUse hook은 guardrail이지 완전한 보안 경계가 아니다.
```

실제 보안은 sandbox, permission, managed config, 외부 API 권한, CI policy와 함께 설계해야 한다. Hook만으로 모든 실행 경로를 완전히 막는다고 생각하면 위험하다.

### PostToolUse

Tool 실행 후 실행된다.

좋은 용도:

```text
tool result logging
명령 실행 결과 검사
수정된 파일 검증
추가 lint/test 안내
audit trail 기록
```

PostToolUse는 이미 실행된 뒤다. 그래서 “막기”보다는 “검사, 기록, 후속 조치”에 가깝다.

### Stop

Turn 종료 시점에 실행된다.

좋은 용도:

```text
대화 요약 저장
작업 결과 검증
다음 action 추천
thread 종료 전 상태 기록
```

Stop hook은 편리하지만, turn 종료마다 실행될 수 있으므로 무겁거나 느린 작업을 넣으면 UX를 망칠 수 있다.

## Hook output은 event마다 의미가 다르다

Hook은 stdout을 통해 Codex와 통신할 수 있다. 하지만 stdout의 의미는 event마다 다르다.

입문 단계에서 가장 위험한 오해는 이것이다.

```text
hook stdout = 항상 모델 context에 들어간다
```

그렇지 않다.

예를 들어 개념적으로는 이렇게 나뉜다.

```text
SessionStart
→ plain text stdout이나 additionalContext가 extra developer context로 들어갈 수 있음

UserPromptSubmit
→ JSON output으로 additionalContext나 decision을 줄 수 있음

PreToolUse
→ plain text stdout은 context로 들어가지 않음
→ JSON decision으로 tool 실행을 제어할 수 있음

PostToolUse
→ 결과 검사나 logging에 적합
→ event contract에 따라 output 의미가 달라짐
```

즉 Hook을 만들 때는 반드시 “이 event에서 stdout이 context인가, decision인가, 단순 output인가?”를 확인해야 한다.

## 작은 예시: SessionStart hook

가장 단순한 hook은 세션 시작 때 짧은 context를 넣는 것이다.

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
            "command": "echo 'PROJECT MODE: prefer concise implementation notes.'",
            "timeout": 5,
            "statusMessage": "Loading project mode"
          }
        ]
      }
    ]
  }
}
```

개념 흐름:

```text
Codex session 시작
→ SessionStart event 발생
→ command 실행
→ stdout 출력
→ 출력이 extra context로 사용될 수 있음
→ 모델이 그 context를 보고 답변
```

이 정도는 이해하기 쉽다. 하지만 실제로는 echo보다 script를 쓰는 경우가 많다.

```text
현재 repo 확인
mode 상태 파일 읽기
필요한 instruction 조립
JSON output 출력
```

Ponytail이나 Caveman류 mode hook은 이 패턴을 더 발전시킨 것이다.

## 작은 예시: UserPromptSubmit mode tracker

UserPromptSubmit hook은 사용자의 prompt를 보고 상태를 바꿀 수 있다.

예를 들어 사용자가:

```text
/mini ultra
```

라고 입력하면 hook script가 stdin으로 prompt 정보를 받고, 상태 파일을 바꾼다.

개념 흐름:

```text
UserPromptSubmit
→ hook script가 prompt 읽음
→ /mini ultra 감지
→ .mini-active 파일에 ultra 저장
→ additionalContext로 "MINI MODE ACTIVE: ultra" 출력
```

여기서 중요한 점:

```text
상태를 기억하는 것은 모델이 아니다.
상태 파일을 읽고 쓰는 것은 hook script다.
그 결과가 다음 모델 호출 context에 들어갈 수 있다.
```

이 패턴이 Caveman/Ponytail 같은 mode plugin의 핵심이다.

## 작은 예시: PreToolUse guardrail

PreToolUse는 tool 실행 전에 끼어든다.

예를 들어 shell command에 위험한 패턴이 있는지 검사할 수 있다.

개념 흐름:

```text
모델이 shell command tool call 생성
→ PreToolUse hook 실행
→ command 내용 검사
→ 허용 또는 deny decision 반환
→ 허용되면 tool 실행
→ 거부되면 실행하지 않음
```

하지만 다시 강조한다.

```text
PreToolUse hook은 안전장치 중 하나다.
최종 보안 경계가 아니다.
```

보안이 중요하다면 hook만 믿지 말고 sandbox, permission, 외부 시스템 권한, 조직 정책과 함께 설계해야 한다.

## Hook은 어디에서 발견되는가

Codex는 active config layer 주변에서 hook을 찾는다.

대표 위치:

```text
~/.codex/hooks.json
~/.codex/config.toml
project/.codex/hooks.json
project/.codex/config.toml
plugin manifest가 가리키는 hooks 파일
plugin의 기본 hooks/hooks.json
```

여러 hook source가 있으면 matching hook들이 함께 실행될 수 있다. 같은 event에 여러 command hook이 있으면 병렬로 시작될 수 있으므로, 한 hook이 다른 hook 실행 자체를 막는다고 가정하면 안 된다.

Plugin-bundled hook도 같은 trust/review 흐름을 따른다.

## Trust/review는 왜 필요한가

Hook은 command를 실행한다.

이 말은 곧 hook이 이런 일을 할 수 있다는 뜻이다.

```text
파일 읽기
파일 쓰기
외부 command 실행
네트워크 호출
로그 저장
상태 파일 수정
```

그래서 Codex는 non-managed command hook을 실행하기 전에 review/trust를 요구한다.

개념 흐름:

```text
hook 발견
→ hook definition 표시
→ 사용자가 review/trust
→ 현재 hook hash 기준으로 신뢰 저장
→ hook 내용이 바뀌면 다시 review 필요
```

이 모델은 귀찮은 절차가 아니라 보안 모델이다.

```text
AGENTS.md
→ instruction file

Hook
→ command execution
```

이 둘은 위험도가 다르다.

## 웹 개발자 비유: middleware, webhook, CI guard

Hook은 하나의 비유로는 딱 떨어지지 않는다. 여러 개념이 섞여 있다.

```text
Express middleware
→ 특정 요청 흐름에 끼어들어 검사/수정

Webhook
→ 특정 event 발생 시 외부 script 실행

CI guard
→ 실행 전후 정책 검사

Shell script
→ 실제 local command 실행
```

그래서 Hook을 “프롬프트”로 생각하면 위험하다.

더 정확한 감각은 이것이다.

```text
Agent runtime에 붙는 event-driven command extension
```

## 흔한 오해

### 오해 1. “Hook은 더 강한 AGENTS.md다”

아니다.

`AGENTS.md`는 instruction file이다. Hook은 command다.

```text
AGENTS.md
→ 읽힌다

Hook
→ 실행된다
```

이 차이 하나로 쓰임과 위험도가 완전히 달라진다.

### 오해 2. “Hook stdout은 항상 모델이 본다”

아니다.

Event마다 stdout의 의미가 다르다. 어떤 event에서는 context가 될 수 있고, 어떤 event에서는 decision만 의미가 있으며, 어떤 plain text output은 무시될 수 있다.

Hook을 만들 때는 event contract를 확인해야 한다.

### 오해 3. “Hook으로 보안을 완전히 강제할 수 있다”

아니다.

Hook은 좋은 guardrail이지만 완전한 보안 경계가 아니다. 실제 보안은 여러 층이 함께 필요하다.

```text
permission model
sandbox
managed config
external API permission
CI policy
human approval
```

### 오해 4. “Hook은 많이 넣을수록 좋다”

아니다.

Hook은 command 실행이므로 비용이 있다.

```text
느려질 수 있음
실패할 수 있음
trust/review 관리가 필요함
디버깅이 어려워질 수 있음
hook끼리 순서 의존이 생길 수 있음
```

정적 규칙은 `AGENTS.md`가 낫고, 특정 작업 절차는 Skill이 낫다. Hook은 실행이 필요한 경우에 쓴다.

## 왜 중요한가

Hook을 이해하면 Ponytail/Caveman이 왜 단순 프롬프트가 아닌지 보인다.

단순 prompt:

```text
짧게 답해줘.
```

Hook 기반 mode:

```text
SessionStart에서 mode instruction 주입
UserPromptSubmit에서 mode command 감지
상태 파일에 현재 mode 저장
다음 prompt에서 상태를 다시 읽음
필요하면 추가 context 주입
```

이건 단순 문장이 아니라 runtime 동작이다.

그래서 다음 장의 Ponytail case study를 볼 때 핵심 질문은 이것이다.

```text
어떤 event에 hook이 등록되어 있는가?
그 event에서 어떤 command가 실행되는가?
그 command는 어떤 상태를 읽고 쓰는가?
stdout은 context인가, decision인가?
사용자가 trust해야 하는가?
```

이 질문을 할 수 있으면 Hook을 이해한 것이다.

## 요약

```text
1. Hook은 instruction file이 아니라 lifecycle event에 실행되는 command다.
2. Hook은 event, matcher, handler 구조로 이해할 수 있다.
3. SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, Stop은 서로 다른 시점에 실행된다.
4. Hook output의 의미는 event마다 다르다. stdout이 항상 모델 context가 되는 것은 아니다.
5. Hook은 command 실행이므로 trust/review가 필요하다.
6. Hook은 guardrail이지 완전한 보안 경계가 아니다.
7. 정적 규칙은 AGENTS.md, 작업 절차는 Skill, lifecycle 실행은 Hook이 적합하다.
```

## 생각해볼 질문

1. 내가 만들고 싶은 규칙은 문서로 충분한가, 아니면 event마다 실행되어야 하는가?
2. SessionStart에 넣을 정보와 UserPromptSubmit에 넣을 정보는 어떻게 다를까?
3. PreToolUse hook으로 막고 싶은 작업이 있다면, hook 외에 어떤 permission/sandbox 정책이 같이 필요할까?
4. Hook stdout이 context가 되는 event와 decision이 되는 event를 구분할 수 있는가?
5. Ponytail이나 Caveman 같은 mode plugin은 어떤 event를 사용해야 자연스러울까?
