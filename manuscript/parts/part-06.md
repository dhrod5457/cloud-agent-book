# Part VI. Cloud의 한계와 다음 단계를 정한다

Cloud Agent를 잘 사용하는 방법에는 Cloud를 쓰지 않을 때를 판단하는 것도 포함된다.

Task가 너무 작거나, Scope가 계속 커지거나, 내부망과 장비가 필요하거나, 같은 실패가 반복된다면 Cloud 이점이 사라질 수 있다. 이때 Cloud Task를 끝까지 유지하는 것이 목표가 아니다. 현재 Evidence를 남기고 Local이나 Hybrid로 재Routing하는 것이 더 나은 선택일 수 있다.

17장에서는 실행 중 발견되는 Stop Signal과 Local Fallback을 다룬다.

18장에서는 Workflow가 안정된 뒤의 다음 단계를 본다. 반복되는 Environment 선택, Runner/Agent 분류, Validation, Retry 판단을 Harness와 일반 코드로 옮길 수 있다. 그 다음에야 Orchestration이 의미를 가진다.

순서는 다음과 같다.

```text
안정된 Task Routing
→ 반복 가능한 Validation
→ Prepared Environment
→ Evidence
→ Harness
→ 필요한 범위의 Orchestration
```

이 책은 더 많은 Agent를 만드는 방법으로 끝나지 않는다.

마지막에 남는 질문은 처음과 같다.

```text
이 Task는 Local에서 할 것인가?
Cloud Runner로 보낼 것인가?
Cloud Agent에게 맡길 것인가?
Hybrid로 나눌 것인가?
Cloud 이점이 사라지면 언제 Local로 돌아올 것인가?
```

> 더 많은 Agent보다 더 나은 Task Routing, Harness, Validation이 먼저다.
