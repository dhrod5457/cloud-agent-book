# 2장 확장 설계 - Agent-friendly Execution Platform

## 목적

클라우드 에이전트를 단순한 원격 AI 개발자로 설명하지 않는다.

이 절에서는 클라우드 환경을 다음과 같이 정의한다.

> 클라우드 에이전트의 최종 형태는 Agent를 계속 실행하는 시스템이 아니라, Agent가 필요할 때 붙을 수 있도록 잘 준비된 실행 플랫폼이다.

기존 핵심 원칙은 유지한다.

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.

여기에 다음 운영 원칙을 추가한다.

- 환경은 미리 준비한다.
- 가능한 판정은 코드로 처리한다.
- Context는 필요할 때 필요한 만큼만 제공한다.
- 실패 결과 전체를 전달하지 않고 조회 가능한 형태로 저장한다.
- Agent의 자율성에는 시간, retry, token, 비용 한도를 둔다.
- Agent가 반복해서 실패하면 프롬프트보다 실행환경과 도구를 먼저 개선한다.

---

# 1. Cloud Agent에서 Cloud Execution Platform으로

초기에는 다음 구조를 떠올리기 쉽다.

```text
Developer
   ↓
Cloud Agent
   ↓
Build
   ↓
Test
   ↓
수정
```

이 구조에서는 Build, Test, Lint처럼 프로그램이 스스로 판정할 수 있는 작업까지 LLM이 통과한다.

권장 구조는 다음과 같다.

```text
Developer / PM
       ↓
Task Scheduler
       ↓
Cloud Runner
       ↓
Build / Test / Lint / Validation
       ↓
Result Gateway
       ↓
   +---+---+
   |       |
 PASS     FAIL
   |       |
  종료      ↓
       Agent Worker
           ↓
          수정
           ↓
       Cloud Runner
           ↓
         재검증
```

Cloud Agent가 모든 작업의 시작점이 아니다.

클라우드 실행 플랫폼은 다음 실행 단위를 제공한다.

- Build Runner
- Unit Test Runner
- Integration Test Runner
- E2E Runner
- Docker Build Runner
- Migration Validator
- Static Analysis Runner
- Security Scanner
- 필요 시 Agent Worker

이를 이 책에서는 `Agent-friendly Execution Platform`으로 부른다.

---

# 2. Prebuilt Environment

Worker가 생성될 때마다 다음 작업을 반복하면 실행시간뿐 아니라 Agent가 환경 문제를 해석하는 비용도 증가한다.

- Repository clone
- JDK 설치
- Node 설치
- Gradle dependency 다운로드
- npm install
- Playwright browser 설치
- Docker base image pull
- Code generation
- build cache 생성

권장 구조:

```text
Base Image
    ↓
Repository 준비
    ↓
Runtime 설치
    ↓
Dependencies 설치
    ↓
Code Generation
    ↓
Cache Warming
    ↓
Snapshot / Image 생성
```

이후 Runner와 Agent Worker는 준비된 환경에서 시작한다.

```text
Prepared Image
- Java 21
- Spring Boot toolchain
- Gradle dependencies
- Node
- node_modules 또는 package cache
- Playwright
- Docker CLI
- Static Analysis Tools
        |
        +-- Runner #1
        +-- Runner #2
        +-- Runner #3
        +-- Agent Worker
```

핵심 원칙:

> Agent에게 개발환경을 설치하게 하지 않는다. 이미 작업할 수 있는 환경을 준다.

환경 설치 실패가 반복된다면 프롬프트를 수정하기보다 base image 또는 bootstrap 과정을 먼저 개선한다.

---

# 3. Cache: 재사용 가능한 상태와 폐기해야 하는 상태를 분리한다

Ephemeral Worker라도 모든 것을 매번 처음부터 다운로드할 필요는 없다.

대표적인 재사용 대상:

- Gradle cache
- Maven repository
- npm cache
- node_modules snapshot
- Docker layer
- Playwright browser
- Code generation result
- Static analysis database
- Test fixture
- LLM과 무관한 Repository index

Java 프로젝트에서는 특히 Gradle dependency와 Docker image의 반복 다운로드가 전체 실행시간에 큰 영향을 줄 수 있다.

다만 cache가 테스트 상태를 오염시키면 재현성이 떨어진다.

따라서 다음 두 종류를 분리한다.

## Reusable Cache

재사용 가능:

- dependency cache
- Docker layer
- compiler cache
- package cache
- browser binary
- immutable generated artifact

## Disposable Runtime State

매 실행 초기화:

- DB
- temporary files
- generated user data
- test output
- process state
- session state
- mutable fixture

권장 구조:

```text
Reusable Cache
      ↓
Fresh Runtime
      ↓
Test / Build
      ↓
Runtime 폐기
      ↓
Cache만 유지
```

---

# 4. Deterministic First

LLM이 판단하지 않아도 프로그램으로 판정할 수 있는 작업은 프로그램으로 처리한다.

좋지 않은 방식:

```text
Agent:
"코드를 살펴보니 문제없이 보입니다."
```

권장 방식:

```text
./gradlew test
PASS

./gradlew check
PASS

architecture-check
PASS

dependency-check
PASS

security-scan
PASS
```

핵심 원칙:

> 판단을 코드로 만들 수 있다면 LLM에게 판단시키지 않는다.

Validation은 가능한 한 executable validation으로 구현한다.

예:

- 테스트 성공 여부: exit code
- architecture dependency: ArchUnit
- API schema compatibility: contract test
- migration 검증: disposable DB
- secret 유출: scanner
- formatting: formatter/linter
- dependency vulnerability: scanner

---

# 5. Cloud Runner와 Agent Worker

## Cloud Runner

특징:

- LLM을 사용하지 않음
- CPU/RAM 중심
- 명령이 결정적임
- exit code로 성공 여부 판단
- 반복 실행에 적합

역할:

- build
- unit test
- integration test
- E2E
- Docker build
- migration validation
- lint
- static analysis
- vulnerability scan

## Agent Worker

특징:

- LLM 사용
- token 사용
- Context 크기에 민감
- 실패 원인 분석과 판단이 필요

역할:

- 원인 분석
- 코드 탐색
- 수정 방향 결정
- 코드 수정
- 설계 판단

기본 원칙:

> Runner가 할 수 있으면 Runner에게 맡긴다.

---

# 6. Failure-driven Agent

정상 경로에서는 LLM이 개입하지 않는다.

예:

```text
100개 작업 실행

95개 PASS
→ Runner에서 종료

5개 FAIL
→ Agent 호출
```

구조:

```text
Task
 |
 v
Runner
 |
 +-- PASS --> Done
 |
 +-- FAIL
 |
 v
Result Gateway
 |
 v
Agent Worker
 |
 v
Fix
 |
 v
Runner
 |
 v
Verification
```

이 구조의 목적은 Agent를 많이 사용하는 것이 아니라 호출 횟수를 줄이는 것이다.

---

# 7. Result Filter에서 Result Gateway로

단순한 로그 축약을 넘어 결과를 조회 가능한 형태로 저장한다.

가정:

- 테스트 로그 100MB
- JUnit XML
- Coverage report
- Browser screenshot/video
- Docker log
- Git diff

원본 결과를 버리지 않는다.

```text
Raw Artifact
├─ build.log
├─ junit.xml
├─ coverage.xml
├─ screenshots/
├─ browser-video/
├─ docker.log
└─ git.diff
       |
       v
Result Gateway
├─ Summary
├─ Failed Test Index
├─ Exception Index
├─ Stack Trace Lookup
├─ Log Search
└─ Artifact Lookup
       |
       v
Agent
```

Agent에게 처음 전달되는 정보는 작게 유지한다.

```text
BUILD: FAIL

Tests:
- total: 8,214
- passed: 8,211
- failed: 3

Failures:
1. AuthServiceTest.expiredToken
2. UserServiceTest.deleteUser
3. UserMapperTest.insert
```

필요할 때만 추가 조회한다.

```text
get_failure("AuthServiceTest.expiredToken")

get_stacktrace("JWTExpiredException")

get_log(
  test="AuthServiceTest.expiredToken",
  lines=100
)
```

핵심 원칙:

> 큰 결과를 요약해서 버리는 것이 아니라, 큰 결과를 저장하고 필요한 부분만 조회한다.

---

# 8. Progressive Context

Agent 시작 시 Repository의 모든 문서를 전달하지 않는다.

좋지 않은 구조:

```text
architecture.md 500줄
security.md 600줄
database.md 400줄
api.md 800줄
testing.md 500줄
→ 모두 전달
```

권장 구조:

```text
AGENTS.md
   |
   +-- Backend → docs/backend.md
   +-- Database → docs/database.md
   +-- Auth → docs/security/auth.md
   +-- Testing → docs/testing.md
   +-- Deployment → docs/deployment.md
```

JWT 오류라면:

```text
AGENTS.md
   ↓
security/auth.md
   ↓
JwtTokenProvider.java
   ↓
관련 테스트
```

핵심 원칙:

> Context를 줄이는 것뿐 아니라 Context를 필요할 때 가져오게 만든다.

---

# 9. Task Context Package

Agent를 호출할 때 가능한 한 작은 작업 패키지를 제공한다.

```text
Task:
AuthServiceTest.expiredToken 실패 수정

Failure:
expected: 401
actual: 200

관련 파일:
- AuthService.java
- AuthServiceTest.java
- JwtTokenProvider.java

검증 명령:
./gradlew test --tests AuthServiceTest.expiredToken

변경 금지:
- DB schema
- 전체 JWT 구조
- 공통 Exception API

완료 조건:
- 해당 테스트 PASS
- 기존 AuthService 테스트 PASS
```

이 구조는 Agent가 Repository 전체를 다시 탐색하는 비용을 줄인다.

정식 Task Contract 형식은 6장에서 설명한다.

---

# 10. Agent별 독립 실행환경

병렬 Agent가 동일한 Working Directory를 공유하지 않도록 한다.

```text
Repository
|
+-- Agent A
|    branch/task-a
|    worktree/task-a
|
+-- Agent B
|    branch/task-b
|    worktree/task-b
|
+-- Agent C
     branch/task-c
     worktree/task-c
```

Cloud 환경에서는 Agent마다 별도 VM/Container와 branch를 할당할 수 있다.

이 구조는 다음 충돌을 줄인다.

- 동일 파일 동시 수정
- dependency 상태 충돌
- test artifact 충돌
- git index 충돌
- 임시 파일 충돌

단, 서로 같은 파일이나 schema를 변경하는 작업은 처음부터 병렬화하지 않는다.

---

# 11. 실패 컨테이너 보존

모든 ephemeral container를 작업 종료 직후 삭제하는 것이 항상 좋은 것은 아니다.

기존 방식:

```text
Test FAIL
→ Container 삭제
→ Agent 생성
→ 환경 다시 구성
→ 오류 재현
```

개선 방식:

```text
Test FAIL
→ Container Freeze
→ Agent Attach
→ 실패 당시 상태 조사
→ 수정
→ 재실행
```

유용한 사례:

- Testcontainers
- DB transaction 문제
- race condition
- browser 상태
- file system 문제
- temporary artifact
- process 상태
- network timeout

권장 정책 예:

```text
PASS
→ 즉시 삭제

FAIL
→ 30분 보존
→ 사람/Agent attach 가능
→ TTL 만료 시 자동 삭제
```

TTL 값은 환경과 비용 정책에 맞게 결정한다.

---

# 12. Artifact First

Cloud Runner는 PASS/FAIL만 반환하지 않는다.

각 작업은 독립된 artifact 디렉터리를 남긴다.

```text
task-142/
├─ result.json
├─ junit.xml
├─ coverage.xml
├─ build.log
├─ git.diff
├─ container-info.json
├─ screenshots/
└─ video/
```

Agent는 기본적으로 `result.json`만 읽는다.

필요할 때 다른 artifact를 조회한다.

사람도 Repository를 checkout하지 않고 결과를 확인할 수 있다.

---

# 13. Budgeted Autonomy

Agent에게 무제한 수정 권한을 주지 않는다.

작업마다 예산을 설정한다.

예:

```text
Simple Test Fix
- max_turns: 8
- max_retry: 2
- max_tokens: 30000

Simple Refactoring
- max_turns: 15
- max_retry: 3
- max_tokens: 50000

Architecture Change
- max_turns: 40
- max_retry: 3
- max_tokens: 200000
```

수치는 정책 예시이며 제품별 사용량 구조와 프로젝트 특성에 맞게 조정한다.

구조:

```text
Agent
  ↓
Analyze
  ↓
Fix
  ↓
Runner
  ↓
+-- PASS --> Done
|
+-- FAIL
     ↓
Budget Check
     ↓
+----+----+
|         |
Remaining Exhausted
|         |
Retry     Human Escalation
```

핵심 원칙:

> Agent의 자율성은 무제한 실행 권한이 아니라 예산 안에서 스스로 해결할 수 있는 권한이다.

Budget 후보:

- wall-clock time
- max turns
- max retries
- max tokens
- max cost
- max changed files
- max diff size

---

# 14. Retry 전략과 Failure Fingerprint

`성공할 때까지 계속 수정해` 같은 지시는 피한다.

Retry마다 실패가 실제로 달라졌는지 확인한다.

예:

```text
Retry #1
Compilation Error

Retry #2
Unit Test Failure

Retry #3
Same Failure
```

동일 failure fingerprint가 반복되면 중단한다.

```text
fingerprint:
AuthServiceTest.expiredToken
expected 401
actual 200
```

정책 예:

```text
동일 fingerprint 2회 반복
→ Agent 중단
→ Human Escalation
```

Fingerprint 후보:

- failing test id
- exception type
- key assertion message
- error code
- top stack frame

Retry는 무조건 반복하는 루프가 아니라 상태 변화가 있는지 확인하는 제어 흐름이어야 한다.

---

# 15. Event-driven Agent

Agent는 항상 살아 있을 필요가 없다.

```text
Git Push
  ↓
CI
  ├─ PASS → 종료
  └─ FAIL → Agent

PR Review Comment
  → Agent

Nightly Test Failure
  → Agent

Security Scan Failure
  → Agent

Dependency Update
  → Runner
  └─ FAIL → Agent
```

이를 `Event-driven Agent Pattern`으로 설명한다.

Agent 활성화 이벤트 후보:

- CI failure
- review comment
- scheduled regression failure
- security alert
- dependency update failure
- human escalation

---

# 16. Continuous Small Task

거대한 작업 하나보다 작은 작업을 지속적으로 반복하는 방식을 우선한다.

좋지 않은 작업:

```text
프로젝트 전체 테스트 커버리지를 100%로 만들어.
```

권장 방식:

```text
오늘:
UserService 미검증 branch 3개

다음 실행:
AuthService 미검증 branch 4개

다음 실행:
PaymentService 미검증 branch 2개
```

각 작업:

```text
분석
→ 작은 변경
→ 검증
→ 작은 PR
→ 다음 Task
```

장점:

- Context가 작음
- 실패 범위가 작음
- rollback 쉬움
- 검증 쉬움
- PR review 쉬움
- token 사용량 예측이 쉬움

GitHub Continuous AI 사례는 이 패턴의 실제 사례로 사용한다. 공개된 숫자는 `research/github/continuous-ai-runner-first.md`에서 관리한다.

---

# 17. Fan-out과 Fan-in

독립 작업은 여러 Worker로 fan-out할 수 있다.

예: Java 17 → Java 21 migration

```text
Migration Controller
|
+----+----+----+----+
|    |    |    |    |
A    B    C    D    E
|    |    |    |    |
Worker / Runner 각각 실행
|    |    |    |    |
PR   PR   PR   PR   PR
 \    |    |    |   /
       Fan-in
          ↓
       통합 검증
```

병렬화 조건:

- 서로 다른 repository
- 서로 다른 module
- 서로 다른 file scope
- 독립적으로 검증 가능

공통 schema나 공통 파일을 수정한다면 dependency를 먼저 분석하고 순차 작업으로 바꾼다.

---

# 18. PM / Orchestrator의 책임

PM Agent는 직접 구현하기보다 작업을 분류하고 실행 자원을 선택하는 역할로 발전한다.

책 후반 19장에서 상세히 다루며 이 절에서는 인터페이스만 정의한다.

책임:

- Task 분해
- Runner 또는 Agent 선택
- Context 범위 결정
- 병렬 실행 가능 여부 판단
- Task Budget 설정
- Retry 여부 결정
- 실패 escalation
- 결과 통합

예:

```text
Task A
- type: unit-test
- context: small
- deterministic: true
- execution: cloud-runner

Task B
- type: bug-fix
- context: medium
- reasoning: required
- execution: cloud-agent

Task C
- type: database-debug
- internal-network: true
- execution: local-agent
```

---

# 19. Harness Engineering

Agent 성능을 모델 성능이나 프롬프트 품질만으로 평가하지 않는다.

Repository와 실행 플랫폼에 다음 요소를 준비한다.

- 명확한 build command
- 명확한 test command
- deterministic validation
- architecture documentation
- AGENTS.md
- coding convention
- error parser
- Result Gateway
- task template
- PR template
- dependency setup
- cache
- reproducible environment

Agent가 반복해서 실패하는 경우 좋지 않은 대응은 프롬프트를 계속 길게 만드는 것이다.

대신 실패 원인을 Harness 부족 관점에서 본다.

```text
테스트 명령을 찾지 못함
→ AGENTS.md에 명시

로그가 너무 큼
→ Result Gateway 추가

환경 설치 실패
→ Prebuilt Image 추가

Architecture 위반 반복
→ architecture-check 추가

같은 오류 반복
→ failure fingerprint 추가
```

핵심 원칙:

> Agent가 반복해서 실패하면 프롬프트보다 Harness를 먼저 개선한다.

---

# 20. 최종 권장 아키텍처

```text
                PM / Orchestrator
                       |
                Task Scheduler
                       |
            Task Classification
                       |
      +----------------+----------------+
      |                                 |
Deterministic                     Reasoning Required
      |                                 |
      v                                 v
Cloud Runner                       Agent Worker
      |                                 |
Build/Test/E2E                    Analyze / Fix
      |                                 |
      v                                 v
Result Gateway <------------------ Runner
      |
  +---+---+
  |       |
 PASS    FAIL
  |       |
 Done  Budget Check
          |
      +---+---+
      |       |
    Retry   Human Escalation
```

기반 계층:

```text
Prebuilt Environment
Cache
Repository Harness
Progressive Documentation
Artifact Store
Isolated Worktree / Container
```

---

# 21. 비용과 토큰을 세 종류로 분리한다

클라우드 Agent 시스템의 비용은 하나가 아니다.

## Compute Cost

- CPU
- RAM
- Storage
- Container runtime

## LLM Cost

- input token
- output token
- reasoning
- tool result processing

## Human Cost

- waiting
- review
- reproduction
- context switching

최적화 목표는 토큰 최소화 하나가 아니다.

예를 들어 Cloud CPU를 더 사용하더라도 개발자가 오류 재현에 30분을 쓰지 않아도 된다면 전체 비용은 낮아질 수 있다.

핵심 기준:

> 싼 deterministic compute로 해결할 수 있는 문제에 비싼 probabilistic reasoning을 사용하지 않는다.

---

# 22. 좋은 사례와 나쁜 사례

## 환경 준비

나쁜 방식:

```text
Agent 생성
→ JDK 설치
→ Node 설치
→ dependency 다운로드
→ 오류 분석
→ build 시작
```

좋은 방식:

```text
Prepared Image
→ Runner/Agent 즉시 작업 시작
```

## 테스트 결과

나쁜 방식:

```text
100MB 로그
→ LLM 전체 전달
```

좋은 방식:

```text
100MB Artifact Store
→ result.json
→ 필요한 failure만 lookup
```

## Retry

나쁜 방식:

```text
같은 실패
→ 계속 수정
→ 계속 재실행
```

좋은 방식:

```text
failure fingerprint 동일 2회
→ 중단
→ Human Escalation
```

## Architecture Validation

나쁜 방식:

```text
Agent가 dependency 방향이 맞는지 판단
```

좋은 방식:

```text
architecture-check
→ PASS / FAIL
```

---

# 23. 다른 장과의 연결

## 4장 Repository as Interface

Progressive Documentation, Context discovery, canonical source를 구체화한다.

## 5장 Agent Contract

Prepared Environment, 실행 명령, Harness entry point를 문서 계약으로 연결한다.

## 6장 Task Contract

Task Context Package, Budget, Forbidden Changes를 정식 계약 형태로 정의한다.

## 8장 실행 인터페이스

Runner가 사용하는 build/test/verify 명령을 표준화한다.

## 10장 자동 검증

Deterministic First와 executable validation을 구체화한다.

## 13장 Git/Worktree/Cloud Sandbox

Agent별 독립 실행환경과 file-scope 충돌 제어를 구체화한다.

## 17장 Observability

다음 지표를 분리해 관찰한다.

- runner execution time
- cache hit ratio
- artifact size
- Agent invocation count
- token usage
- retry count
- repeated fingerprint count
- human escalation count

## 19장 PM Agent

Task Classification, Budget, Retry, Runner/Agent 선택 정책을 상세히 설명한다.

---

# 24. 이 절의 결론 메시지

초기 Agent 활용은 다음과 같았다.

> AI에게 일을 시킨다.

다음 단계는 다음과 같다.

> AI 여러 개에게 일을 병렬로 시킨다.

더 성숙한 구조는 다음과 같다.

> 소프트웨어가 처리할 수 있는 작업은 소프트웨어가 처리하고, 판단이 필요한 순간에만 AI를 호출한다.

최종 원칙:

1. LLM은 판단하고, 컨테이너는 실행한다.
2. 정상 경로는 deterministic Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.
3. CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.
4. 환경은 Agent가 만들게 하지 말고 미리 준비한다.
5. Context를 한꺼번에 전달하지 말고 필요할 때 조회하게 한다.
6. 로그를 버리면서 요약하지 말고 저장한 뒤 필요한 부분만 조회한다.
7. 검증할 수 있는 것은 AI에게 묻지 말고 코드로 검증한다.
8. Agent의 자율성에는 Budget을 둔다.
9. Agent가 반복해서 실패하면 프롬프트보다 Harness를 개선한다.
10. 클라우드 에이전트의 목표는 AI 사용량을 늘리는 것이 아니라 사람의 개입이 필요한 순간을 줄이는 것이다.

---

# 본문 작성 시 주의사항

- 특정 AI 제품 홍보처럼 작성하지 않는다.
- 제품 사례는 설계 패턴을 설명하기 위한 근거로만 사용한다.
- CPU/RAM/가격/세션 제한처럼 변경 가능한 제품 정보는 research 문서로 분리한다.
- 실제 숫자는 공식 출처가 있는 경우에만 사용한다.
- 19장의 PM Agent 상세 설명을 여기서 중복하지 않는다.
- 6장의 Task Contract 상세 형식을 여기서 반복하지 않는다.
- Java/Spring Boot 예제에서는 Gradle cache, Testcontainers, Docker layer, ArchUnit, JUnit XML을 중심으로 실제 적용 흐름을 보여준다.
- Phase 5에서는 설계만 확정하며 본문 초고는 Phase 6에서 작성한다.
