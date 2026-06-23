[← README](../README.md)

# 1. LLM은 기억하는 것이 아니라 context를 읽는다

처음 AI를 쓰면 모델이 “기억한다”고 느껴진다. 방금 말한 내용을 알고, 이전 대화 흐름을 이어가고, 어떤 때는 오래전 취향까지 아는 것처럼 답한다.

하지만 하네스 엔지니어링 관점에서는 이렇게 보는 편이 안전하다.

```text
model(context) -> next_message_or_tool_call
```

모델은 지금 들어온 context를 보고 다음 출력을 만든다. 그 context 안에 최근 대화가 있으면 기억하는 것처럼 보이고, 장기 기억 검색 결과가 있으면 오래전 일을 아는 것처럼 보인다.

## 웹 개발자 비유

LLM 호출은 stateless API request에 가깝다.

```text
HTTP request
→ middleware가 user/session/feature flag를 붙임
→ controller 실행

LLM request
→ harness가 history/memory/tool schema/instructions를 붙임
→ model inference 실행
```

서버가 매 요청마다 DB에서 session을 읽어 붙이듯, 에이전트 하네스도 매 요청마다 필요한 기억과 도구 설명을 context에 붙인다.

## 사용자의 Telegram LLM 서버 경험

사용자는 예전에 Telegram 메시지를 받는 Node 서버를 만들었다.

```text
Telegram 채팅
→ Node 서버가 요청 수신
→ 최근 대화 몇 건 조회
→ 장기 기억과 적합도 비교
→ 관련 기억 선택
→ tool 설명과 input/output schema 추가
→ 현재 질문과 함께 LLM 호출
→ 필요하면 tool 실행
→ 답변 또는 일정 등록
```

이 구조는 Codex를 이해하는 데 거의 정답에 가깝다. Codex는 같은 원리를 더 큰 제품 레벨로 구현한다.

## 기억처럼 보이는 것의 정체

```text
최근 대화가 context에 들어감
→ 모델이 방금 대화를 기억하는 것처럼 답함

요약된 history가 context에 들어감
→ 모델이 긴 대화를 따라오는 것처럼 답함

memory DB/file 검색 결과가 context에 들어감
→ 모델이 장기 기억을 가진 것처럼 답함

tool output이 context에 들어감
→ 모델이 파일이나 DB를 직접 본 것처럼 답함
```

중요한 점은 “모델 안에 모든 기억이 있다”가 아니라 “하네스가 적절한 정보를 넣어준다”는 것이다.

## 왜 중요한가

이걸 모르면 질문이 이렇게 된다.

```text
왜 모델이 까먹지?
왜 이걸 기억 못 하지?
왜 방금 한 말을 다르게 이해하지?
```

이걸 알면 질문이 바뀐다.

```text
무슨 context가 빠졌지?
어떤 기억을 검색해서 넣어야 하지?
너무 오래된 정보가 들어간 건 아닐까?
tool 결과를 더 명확히 넣어야 하나?
```

하네스 엔지니어링은 여기서 시작한다. 모델을 탓하기 전에, 모델이 실제로 무엇을 보고 있었는지 확인한다.

