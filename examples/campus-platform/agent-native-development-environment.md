# campus-platform Agent-Native Development Environment 설계

이 문서는 책의 `Agent-Native Development Environment` 장에서 사용할 `campus-platform` 적용 예를 정의한다.

구현 코드는 아직 작성하지 않는다.

## 목표

하나의 Agent가 Repository 전체를 직접 처리하는 구조 대신 다음을 분리한다.

- Brain: 판단과 task state
- Hands: Backend, Browser, Android, Database 실행환경
- Runner: deterministic build/test/validation
- Result Gateway: artifact indexing과 필요한 결과 조회
- PM / Scheduler: task classification과 compute allocation

## 기본 구조

```text
                     PM / Scheduler
                          |
                  Task Classification
                          |
            +-------------+-------------+
            |                           |
     Deterministic                   Reasoning
            |                           |
         Runner                    Agent Brain
            |                           |
     +------+------+          +---------+---------+
     |      |      |          |         |         |
 Backend  Web   Android    Linux Hand Browser Hand Android Hand
     |      |      |          |         |         |
     +------+------+----------+---------+---------+
                          |
                    Result Gateway
                          |
                  Deterministic Verify
```

## Execution Hands

### Backend Hand

대상:

- Java 21
- Spring Boot
- Gradle
- PostgreSQL/Testcontainers
- Redis/Kafka가 필요한 integration test

역할:

- compile
- unit test
- integration test
- architecture test
- migration validation

### Web Hand

대상:

- 관리자 Web
- Node runtime
- browser
- Playwright

역할:

- frontend build
- component/unit test
- E2E
- accessibility
- browser console/network artifact 수집

### Android Hand

대상:

- Android SDK
- emulator/device execution environment

역할:

- build
- unit test
- instrumented test
- 로그인/모바일ID 같은 주요 flow 검증

### Database Hand

대상:

- disposable PostgreSQL
- 기업 환경 단계에서는 Local Tibero/Oracle validation hand로 교체 가능

역할:

- schema migration
- seed fixture
- query compatibility
- rollback/forward migration test

## Brain

Agent Brain의 기본 입력은 Repository 전체가 아니다.

입력 후보:

- Task Context Package
- Result Gateway summary
- relevant source references
- required documentation references
- current task budget

Brain은 필요한 hand를 tool처럼 호출한다.

```text
execute(backend-test, input)
execute(browser-e2e, input)
execute(android-test, input)
execute(db-migration, input)
```

실제 API 이름은 구현 단계에서 결정한다.

## 예제 Task: 토큰 만료 처리 회귀 검증

### Task

`AuthServiceTest.expiredToken` 실패 및 사용자 재로그인 flow 검증

### 1단계 - Backend Runner

```text
./gradlew test --tests AuthServiceTest.expiredToken
```

PASS이면 Backend bug 분석 Agent를 호출하지 않는다.

FAIL이면 Result Gateway가 다음만 초기 반환한다.

```text
status: FAIL
test: AuthServiceTest.expiredToken
expected: 401
actual: 200
artifact: task-142/backend/
```

### 2단계 - Agent Brain

필요 Context:

- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java
- auth 관련 문서

Agent가 수정한 후 Backend Runner 재검증.

### 3단계 - Web Hand

만료 세션에서 로그인 화면으로 전환되는지 Playwright로 검증.

Artifact:

- trace
- screenshot
- console errors
- network failures

### 4단계 - Android Hand

만료 access token 이후 재로그인 또는 refresh flow를 검증.

### 5단계 - Fan-in

Result Gateway가 결과를 통합한다.

```text
Backend: PASS
Web: PASS
Android: PASS
DB: NOT_REQUIRED
```

Full validation 정책을 만족하면 PR 상태를 갱신한다.

## Agent-friendly Log

Runner stdout은 가능한 한 짧게 유지한다.

```text
TASK=auth-expired-token
STATUS=FAIL
TEST_TOTAL=43
TEST_FAIL=1
ERROR=AuthServiceTest.expiredToken expected=401 actual=200
ARTIFACT=task-142/backend
```

상세 Gradle/JUnit log는 artifact로 저장한다.

## Artifact Store

예:

```text
artifacts/
└─ task-142/
   ├─ result.json
   ├─ backend/
   │  ├─ junit.xml
   │  ├─ build.log
   │  └─ coverage.xml
   ├─ web/
   │  ├─ playwright-trace.zip
   │  ├─ screenshots/
   │  └─ console.json
   ├─ android/
   │  ├─ test-result.xml
   │  └─ logcat.txt
   └─ db/
      └─ migration-result.json
```

Agent는 기본적으로 `result.json`과 failure index만 읽는다.

## Progressive Context

```text
AGENTS.md
  ↓
Task Type: auth
  ↓
docs/security/auth.md
  ↓
관련 Source/Test
  ↓
Result Gateway failure detail
  ↓
필요한 raw artifact
```

## Prebuilt Environment

각 hand는 가능한 범위에서 prepared image/snapshot을 사용한다.

### Backend Image

- JDK
- Gradle wrapper/toolchain
- dependency cache reference
- Docker/Testcontainers 지원 도구

### Web Image

- Node
- package cache
- Playwright browser

### Android Image

- JDK
- Android SDK
- Gradle/Android dependency cache
- emulator image reference

Mutable runtime state는 task마다 초기화한다.

## Deterministic Sampling

대형 regression suite가 생겼을 때 다음 계층을 고려한다.

```text
Feature Branch
→ deterministic fast sample

PR
→ wider sample

Merge Queue
→ full test
```

같은 retry는 같은 sample seed를 사용한다.

## Agent Hibernate

PR이 CI나 reviewer 응답을 기다리는 동안 Brain state는 durable session으로 유지하고 expensive hands는 종료할 수 있다.

```text
WORKING
→ WAITING_CI
→ SNAPSHOT
→ OFF
→ REVIEW_EVENT
→ WAKE
→ WORKING
```

## PR Babysitter

PR 생성 후 다음 이벤트를 관찰한다.

- CI status
- reviewer comment
- conflict
- flaky test result
- dependency/security check

PASS 경로는 deterministic check만 수행한다.

코드 판단이 필요한 failure만 Agent Brain을 깨운다.

## Compute-aware Scheduling

예시:

```text
backend-unit
- model: none
- hand: backend
- cpu: small
- memory: small

integration-test
- model: none
- hand: backend
- cpu: medium
- memory: high

web-e2e
- model: none unless failure
- hand: browser

android-debug
- model: required on failure
- hand: android
- internal-device: false

Tibero-debug
- execution: local-agent
- internal-network: true
```

구체적인 CPU/RAM 값은 특정 cloud 제품에 종속시키지 않는다.

## Agent-native Observability

Agent가 dashboard screenshot을 해석하는 방식보다 구조화 query interface를 우선한다.

예:

```text
get_errors(service="cms-backend", since="10m")
get_slow_spans(service="auth", threshold_ms=2000)
get_http_failures(status=500)
get_browser_console_errors(task="task-142")
get_test_failure(task="task-142", test="AuthServiceTest.expiredToken")
```

## Agent Replay / Benchmark

Agent platform 변경을 검증하기 위해 대표 Task를 replay 가능하게 저장한다.

예제 benchmark 후보:

- JWT expiration bug
- student lookup NPE
- migration failure
- admin UI validation bug
- forbidden module dependency

저장 메타데이터:

- Git SHA
- task input
- context reference
- model/harness version
- environment version
- tool calls
- result
- execution time
- token usage
- retry count

## 적용 단계

### 현재 적용 가능

1. build/test/verify 표준화
2. Agent-friendly result.json
3. Artifact directory
4. Progressive Context 문서 구조
5. task branch/worktree 격리
6. failure fingerprint
7. retry/token/time budget
8. PR event 기반 Agent 호출

### 이후 플랫폼 확장

1. Brain/Hands/Session 분리
2. Backend/Web/Android multi-hand
3. Warm Pool
4. Agent Hibernate
5. Best-of-N
6. Shadow/Canary Agent
7. Agent Replay CI
8. Agent-native Observability query plane
