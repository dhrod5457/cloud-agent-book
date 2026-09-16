# Part IV. Local과 Cloud를 연결한다

독립 실행환경에서 Task를 잘 처리하는 것만으로는 실제 개발 Workflow가 완성되지 않는다.

개발자는 Local에서 요구사항을 좁히고 Architecture와 내부 제약을 판단한다. Cloud에서는 독립 작업과 검증을 수행한다. 결과는 다시 Local로 돌아와 Review, Internal Validation, Merge를 거친다.

이 Part에서는 이 이동 경계를 다룬다.

사람이 직접 Task를 정리해 Cloud로 보내는 경우에는 `Git + Task Contract`가 입력 경계가 된다. Cloud에서 돌아오는 결과는 `Result SHA + Evidence + PR`로 연결한다.

같은 구조는 CI Failure, Review Comment, Schedule 같은 Event에도 적용할 수 있다. Event 자체가 Agent 호출 명령은 아니다. 먼저 Task Candidate로 만들고, 중복을 제거하고, Runner나 Tool로 끝낼 수 있는지 확인한 뒤 판단이 필요한 경우에만 Agent를 호출한다.

```text
Human-driven
Developer → Task Contract → Cloud

Event-driven
CI / Review / Schedule → Task Candidate → Cloud
```

두 경로의 공통점은 같다.

> Git으로 작업 상태를 넘기고 Evidence로 결과를 돌려받는다.

다음 Part에서는 이 구조를 하나의 실제 프로젝트 Workflow로 합친다.
