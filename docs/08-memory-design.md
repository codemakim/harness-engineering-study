[← README](../README.md)

# 8. Memory design: 모델이 기억하는 게 아니라 하네스가 가져온다

Memory는 모델 내부에 박힌 마법이 아니다. 대부분은 외부 저장소에서 관련 정보를 찾아 context에 넣는 설계다.

## 기억의 실제 위치

```text
현재 thread history
압축된 conversation summary
memory 파일
DB row
vector store 검색 결과
Notion/Gmail/Calendar/Slack/GitHub 같은 외부 app data
사용자가 만든 문서
방금 tool이 읽은 출력
```

모델이 “아는 것”처럼 보이는 정보는 보통 이 중 하나가 context에 들어왔기 때문이다.

## Memory retrieval은 ranking 문제다

많이 넣는다고 좋은 게 아니다.

```text
너무 적게 넣음
→ 모델이 필요한 배경을 모름

너무 많이 넣음
→ 중요한 정보가 묻힘

오래된 정보를 넣음
→ 모델이 틀린 확신을 가짐

출처 없이 넣음
→ 나중에 왜 그렇게 답했는지 추적 불가
```

좋은 memory system은 다음을 고민한다.

```text
recency: 최근 정보인가?
relevance: 지금 질문과 관련 있는가?
authority: 사용자 명시 선호인가, 추론인가?
staleness: 바뀌었을 가능성이 있는가?
source: 어디서 온 정보인가?
```

## 웹 개발자 비유

Memory는 session + cache + DB search + ranking을 합친 것에 가깝다.

```text
session
→ 지금 대화에서 이어지는 상태

cache
→ 자주 쓰는 사용자 선호

DB search
→ 과거 기록 검색

ranking
→ 지금 request에 넣을 것 선택
```

## 이 repo 자체도 memory다

```text
/Users/jhkim/Documents/Codex/harness-engineering-study/README.md
```

이 파일은 모델 내부 기억이 아니다. 하지만 다음 세션에서 “이 README 읽고 이어가자”라고 하면 Codex가 파일을 읽고 context로 가져올 수 있다.

즉 문서화는 memory design의 가장 단순하고 강력한 형태다.

