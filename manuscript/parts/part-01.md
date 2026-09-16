# Part I. Cloud Agent를 이해한다

Coding Agent가 Local에서 파일을 읽고 수정하는 것만으로는 Cloud Agent를 설명하기 어렵다.

Cloud Agent는 Repository와 독립 실행환경을 함께 사용한다. Build와 Test를 실행하고, 장시간 작업을 비동기적으로 처리하며, 여러 Task를 서로 다른 실행환경에 나눠 맡길 수 있다.

이 Part에서는 먼저 Cloud Agent를 **Remote Development Worker**로 정의한다. 이어서 Local과 Cloud의 차이를 실행 위치와 접근 범위로 나누고, LLM이 판단하는 자원과 CPU / RAM / Disk가 실행하는 자원을 구분한다.

마지막으로 독립 실행환경이 왜 장시간 작업과 병렬 실행에 의미가 있는지 살펴본다.

이 Part를 읽은 뒤에는 다음 질문을 구분할 수 있어야 한다.

```text
Cloud Agent는 Local Agent와 무엇이 다른가?
Compute와 LLM Token은 어떻게 다른가?
Cloud가 개발자의 대기시간을 어떻게 줄일 수 있는가?
어떤 병렬성이 실제 개발에 도움이 되는가?
```

다음 Part부터는 이 실행환경에 **어떤 Task를 보낼 것인지** 결정한다.
