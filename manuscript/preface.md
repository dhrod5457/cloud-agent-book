# 서문

Coding Agent가 코드를 작성하고, 테스트를 실행하고, Pull Request를 만들 수 있게 되면서 개발자의 질문도 달라지고 있다.

처음에는 어떤 모델이 더 코드를 잘 쓰는지가 중요했다. 실제 프로젝트에 Agent를 넣기 시작하면 다른 문제가 더 크게 보인다.

```text
이 작업을 내 개발환경에서 계속할 것인가?
아니면 Cloud에 넘길 것인가?
```

Cloud Agent를 단순히 `클라우드에서 실행되는 AI`로 보면 이 질문에 답하기 어렵다.

이 책에서는 Cloud Agent를 다음과 같이 본다.

```text
Cloud Agent
= LLM
+ Repository
+ Independent Execution Environment
+ CPU / RAM / Disk
+ Development Tools
```

즉 Cloud Agent는 대화만 하는 모델이 아니라 Repository와 실행환경을 받아 실제 개발 작업을 수행할 수 있는 Remote Development Worker다.

이 관점으로 보면 Cloud Agent의 장점도 달라진다.

핵심 가치는 단순히 더 많은 Token이나 더 큰 모델에 있지 않다. Local 개발환경과 분리된 곳에서 장시간 작업을 실행하고, 여러 독립 Task를 병렬로 처리하며, 개발자가 그 시간 동안 다른 일을 할 수 있다는 점이 중요하다.

하지만 모든 작업을 Cloud로 보내는 것이 좋은 방법은 아니다.

아키텍처를 결정해야 하는 작업, 내부망과 실제 장비가 필요한 작업, 요구사항이 계속 바뀌는 작업, 개발자의 짧은 Feedback Loop가 필요한 작업은 Local이 더 적합할 수 있다.

반대로 실행 명령과 완료 조건이 분명한 Build, Test, E2E, Docker Build 같은 작업은 Cloud Runner에 맡기기 쉽다. 재현 가능한 실패를 분석하고 제한된 범위의 코드를 수정하는 작업은 Cloud Agent 후보가 될 수 있다.

그래서 이 책의 중심 질문은 `어떤 Agent를 사용할 것인가`가 아니다.

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

이 질문에 답하려면 모델 선택만으로는 부족하다.

Task의 범위, Context 크기, 실행환경, Git 상태, 테스트 방법, Evidence, 내부망 의존성, 병렬화 비용, Review 비용을 함께 봐야 한다.

이 책은 그 판단을 실제 개발 Workflow 안에서 다룬다.

Java/Spring Boot 기반 `campus-platform` 예제를 사용하지만 특정 제품 사용 설명서는 아니다. GitHub Copilot cloud agent, OpenAI Codex, Claude 계열의 hosted agent 사례는 공통 실행 패턴과 설계 원칙을 확인하기 위한 근거로만 사용한다.

제품 이름과 기능은 바뀔 수 있다. 하지만 다음 문제는 계속 남는다.

```text
작업을 어떻게 나눌 것인가?
어디서 실행할 것인가?
어떤 상태를 넘길 것인가?
무엇으로 검증할 것인가?
어떤 결과를 받아야 하는가?
```

이 책에서는 다음 방향을 일관되게 유지한다.

- Runner가 할 수 있으면 Runner에게 맡긴다.
- Agent는 판단과 수정이 필요한 구간에만 사용한다.
- 작은 Task와 작은 Context를 전달한다.
- 원본 결과는 Artifact로 보존하고 필요한 정보만 Agent에게 보여준다.
- Git을 Local과 Cloud 사이의 Handoff Boundary로 사용한다.
- 병렬화의 대상은 Agent가 아니라 독립 Task다.
- Cloud 이점이 사라지면 Local로 돌아온다.

Cloud Agent는 Local 개발을 없애지 않는다. CI와 Runner도 없애지 않는다.

오히려 각 역할을 더 분명하게 나누게 만든다.

이 책을 다 읽은 뒤에는 새로운 Cloud Agent 제품을 볼 때 기능 목록부터 확인하기보다 자신의 작업을 먼저 보고 다음을 판단할 수 있기를 바란다.

```text
이 설계는 Local에서 하자.
이 검증은 Cloud Runner로 보내자.
이 실패는 Cloud Agent에게 맡기자.
이 마지막 검증은 내부망에서 하자.
```

그 판단을 반복 가능한 개발 Workflow로 만드는 것이 이 책의 목적이다.
