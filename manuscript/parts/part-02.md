# Part II. 어떤 Task를 Cloud로 보낼 것인가

Cloud에서 실행할 수 있다는 이유만으로 모든 작업을 Cloud에 보내는 것은 좋은 Routing이 아니다.

작업마다 필요한 Context, Human Steering, 내부망 접근, 실행시간, 검증 방식이 다르다. 같은 Bug Fix라도 재현 가능하고 Scope가 작다면 Cloud Agent에 적합할 수 있지만, Architecture 판단과 내부 시스템 접근이 필요하다면 Local이 더 적합할 수 있다.

이 Part에서는 먼저 Task의 실행 위치를 결정하는 기준을 만든다. 다음으로 Build, Test, E2E, Docker, Migration, Bug Fix, Refactoring 같은 실제 개발 작업을 `Local / Cloud Runner / Cloud Agent / Hybrid` 관점에서 분류한다.

Cloud Agent에게 작업을 넘기기로 했다면 입력도 줄여야 한다. Task Contract로 Scope와 Validation을 고정하고, Repository 전체가 아니라 필요한 Context부터 제공한다. 결과 역시 대형 로그가 아니라 검증 가능한 Evidence를 중심으로 받는다.

이 Part의 흐름은 다음과 같다.

```text
Task 선택
→ 실행 주체 결정
→ 입력 범위 축소
→ 결과 / Evidence 경계 정의
```

다음 Part에서는 이렇게 선택한 Task가 Cloud에서 바로 실행될 수 있도록 **환경과 검증 경로**를 설계한다.
