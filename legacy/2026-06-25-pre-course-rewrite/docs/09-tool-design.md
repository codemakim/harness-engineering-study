[← README](../README.md)

# 9. Tool design: 모델에게 함수를 맡기는 법

Tool은 그냥 함수 목록이 아니다. 모델을 위한 API다.

모델은 tool의 이름, 설명, input schema, 사용 지침을 보고 “언제 어떤 tool을 어떤 인자로 호출할지” 판단한다.

## 좋은 tool 설명에 들어갈 것

```text
name
→ 짧고 구체적이어야 함

description
→ 언제 쓰는 tool인지 명확해야 함

input schema
→ 필수/선택 필드, 타입, 제약이 분명해야 함

output contract
→ 결과가 어떤 형태로 돌아오는지 예측 가능해야 함

when to call
→ 호출 조건

when not to call
→ 호출하면 안 되는 상황

permission/safety
→ 외부 쓰기, 삭제, 결제, 메시지 전송 같은 위험 동작의 승인 규칙
```

## 웹 개발자 비유

Tool schema는 모델을 위한 API 문서다.

나쁜 API 문서가 나쁜 client 코드를 만들듯, 나쁜 tool 설명은 나쁜 tool call을 만든다.

```text
애매한 tool 이름
→ 모델이 언제 써야 할지 헷갈림

너무 넓은 tool
→ 위험한 동작까지 한 tool에 섞임

불명확한 output
→ 모델이 결과를 잘못 해석함

권한 설명 없음
→ 모델이 해도 되는 일과 안 되는 일을 구분 못 함
```

## Tool은 하네스가 실행한다

모델이 직접 실행하지 않는다.

```text
모델
→ tool call 제안

하네스
→ schema validation
→ permission 확인
→ 실제 실행
→ 결과를 모델 context에 반환
```

이 구조 때문에 tool design에는 보안과 UX가 같이 들어간다.

## 핵심 질문

Tool을 만들 때는 항상 물어본다.

```text
이 tool은 한 가지 일을 하는가?
실패하면 어떤 output을 내는가?
같은 요청을 두 번 실행해도 안전한가?
외부 상태를 바꾸는가?
사용자 승인이 필요한가?
모델이 이 tool을 과하게 호출할 위험은 없는가?
```

