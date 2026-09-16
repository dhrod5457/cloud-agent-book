# Part III. Cloud 실행환경과 검증을 설계한다

Cloud에 보낼 Task를 골랐다면 다음 문제는 실행 준비다.

Agent가 Session을 시작할 때마다 JDK, Browser, Dependency, DB Tool을 설치한다면 Task보다 환경 준비에 더 많은 시간이 들 수 있다. 반대로 준비된 환경이 있어도 모든 실패를 Agent에게 넘기면 결정론적으로 처리할 수 있는 Build와 Test까지 LLM 비용을 사용하게 된다.

이 Part에서는 Cloud Task가 빠르게 시작하고, 필요한 경우에만 Agent가 개입하도록 실행 경로를 만든다.

Prepared Environment와 Cache로 시작 비용을 줄이고, 정상 검증은 Cloud Runner가 처리한다. Source는 Branch나 Worktree로, Runtime은 Container나 VM으로, 결과는 Artifact 경로로 분리한다. 병렬화할 때는 Worker 수보다 독립 Task와 Fan-in 비용을 먼저 본다.

핵심 흐름은 다음과 같다.

```text
Prepared Environment
→ Runner-first
→ Source / Runtime / Evidence Isolation
→ Independent Task Fan-out
→ Fan-in
```

이 구조가 만들어지면 Cloud 작업은 독립적으로 실행될 수 있다. 다음 Part에서는 이 독립 Task를 기존 개발 Workflow와 연결한다.
