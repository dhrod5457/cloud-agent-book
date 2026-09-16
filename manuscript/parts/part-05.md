# Part V. 실제 프로젝트에 적용한다

앞의 Part에서는 Cloud Agent Workflow를 구성하는 요소를 각각 나눠 설명했다.

이제 그 요소를 하나의 프로젝트 안에서 연결한다.

예제는 Java/Spring Boot 기반 `campus-platform`이다. Local에서는 요구사항과 Architecture, 내부망 검증을 담당하고, Cloud Runner는 Build/Test/E2E/Docker 같은 결정론적 실행을 맡는다. Cloud Agent는 재현 가능한 Failure 분석과 제한된 코드 수정에 사용한다.

15장에서는 이 구조를 **정적인 운영 모델**로 본다.

```text
Task Routing
→ Task Contract
→ Environment
→ Runner / Agent
→ Evidence
→ Handoff
```

16장에서는 같은 구조를 `학생 출결 API 인증 변경` 기능 하나에 적용해 Requirement부터 Merge까지 시간 순서대로 따라간다.

이 Part의 목적은 새로운 개념을 추가하는 것이 아니다.

앞에서 만든 원칙이 실제 기능 개발에서 어떻게 연결되는지 확인하는 것이다.

다음 Part에서는 이 Workflow를 언제 멈추거나 Local로 되돌려야 하는지, 그리고 반복되는 판단을 어느 단계부터 자동화할 수 있는지 정리한다.
