# Cloud Runner / Agent Worker 설계

이 문서는 `campus-platform` 예제에서 Cloud 실행 환경을 Build/Test/Validation 노드로 사용하고, 판단이 필요한 실패에만 Agent Worker를 호출하는 구조를 정의한다.

구현 코드는 아직 작성하지 않는다.

## 핵심 원칙

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.

목표는 Agent를 많이 호출하는 것이 아니다.

정해진 명령으로 처리 가능한 작업은 Runner에서 끝내고, 실패나 모호한 판단이 필요한 경우에만 Agent Worker를 호출한다.

## 역할 구분

### Cloud Runner

결정론적으로 실행 가능한 작업을 담당한다.

- build
- unit test
- integration test
- E2E test
- Docker build
- lint
- static analysis
- migration validation
- security scan

특징:

- CPU/RAM 중심
- LLM 추론 불필요
- 정해진 command 실행
- exit code와 artifact 생성
- 병렬화 가능

### Agent Worker

판단이 필요한 작업을 담당한다.

- 코드 분석
- 실패 원인 판단
- 설계 결정
- 코드 수정
- 복잡한 오류 분석
- 변경 범위 판단

특징:

- LLM token/usage 발생
- Context 크기가 중요
- 호출 빈도를 줄일수록 성공 경로 비용을 줄일 수 있음

## 기본 구조

```text
Local Developer / PM Agent
            ↓
       Task Scheduler
            ↓
        Cloud Runner
            ↓
 Build / Test / Lint / Validation
            ↓
       +----+----+
       |         |
      PASS      FAIL
       |         |
      종료   Result Filter
                 ↓
           Agent Worker
                 ↓
          원인 분석 / 수정
                 ↓
            Cloud Runner
                 ↓
              재검증
                 ↓
                PR
```

## 실행 정책

기본 정책은 `Runner-first`다.

```text
Task
  ↓
Runner
  ├─ PASS → 완료
  │
  └─ FAIL
       ↓
   Result Filter
       ↓
   Agent Worker
       ↓
      수정
       ↓
   Runner 재실행
```

다음 조건에서는 Agent를 호출하지 않는다.

- build 성공
- test 성공
- lint 성공
- migration validation 성공
- security/static rule 성공

다음 조건에서는 Agent 호출 후보가 된다.

- test failure
- build failure 중 코드 원인이 의심되는 경우
- 서로 다른 실패 원인이 섞인 경우
- 규칙만으로 수정 방향을 결정하기 어려운 경우
- CI/review event에서 코드 수정 판단이 필요한 경우

Infrastructure failure는 코드 Agent로 바로 넘기지 않는다.

## Result Filter

Raw stdout/stderr를 그대로 LLM에 전달하지 않는다.

```text
Cloud Runner
      ↓
  Raw Result
      ↓
 Result Filter
      ├─ exit code
      ├─ failed tests
      ├─ root cause 후보
      ├─ 핵심 stack trace
      ├─ error count
      ├─ warning count
      └─ artifact path
      ↓
 Agent Worker
```

Result Filter는 기본적으로 일반 프로그램이나 스크립트로 구현한다.

예상 결과:

```json
{
  "taskId": "verify-user-delete",
  "status": "FAIL",
  "exitCode": 1,
  "tests": {
    "total": 8214,
    "passed": 8211,
    "failed": 3
  },
  "failures": [
    {
      "test": "UserServiceTest.deleteUser",
      "location": "UserService.java:142",
      "cause": "NullPointerException"
    },
    {
      "test": "AuthServiceTest.expiredToken",
      "cause": "expected 401, actual 200"
    },
    {
      "test": "UserMapperTest.insert",
      "cause": "duplicate key"
    }
  ],
  "rawLog": "artifacts/test/verify-user-delete.log"
}
```

원본 로그는 artifact로 보존한다.

Context는 다음 순서로 확대한다.

```text
Summary
  ↓
Failure Detail
  ↓
Specific Test Log
  ↓
Relevant Source Files
  ↓
Full Raw Log
```

## 성공 경로에서 LLM 제거

예를 들어 하루 동안 검증 작업 100건을 실행한다고 가정한다.

```text
100 Runner jobs
├─ 95 PASS → 종료
└─ 5 FAIL  → Agent Worker 호출
```

95개의 정상 경로에서는 LLM 분석을 수행하지 않는다.

실패 5건에서도 먼저 Result Filter가 결과를 축약한 후 Agent Worker를 호출한다.

## Runner 유형

초기 논리적 Runner:

```text
runner-unit
runner-integration
runner-e2e
runner-docker
runner-migration
runner-static-analysis
runner-security
```

실제 인프라는 특정 Cloud Agent 제품에 종속시키지 않는다.

대체 가능한 실행 환경:

- GitHub Actions Runner
- Jenkins Agent
- Kubernetes Job
- 자체 VM
- Cloud coding environment

## 병렬 Fan-out

독립적인 모듈 검증은 fan-out한다.

```text
             Scheduler
                 |
+----------------+----------------+
|                |                |
student       attendance      notification
Runner          Runner           Runner
|                |                |
test             test             test
|                |                |
result           result           result
```

Java 17 → 21 migration처럼 독립 서비스가 여러 개라면 Repository 또는 module 단위로 병렬 실행할 수 있다.

각 Runner는 Repository 전체 의미를 이해할 필요가 없다.

Scheduler가 최소한 다음 정보를 전달한다.

- task id
- module/path
- 실행 명령
- timeout
- 완료 조건
- artifact/log 저장 위치

## UI/E2E 반복 검증

Cloud compute를 적극적으로 사용하는 사례로 Playwright 기반 UI/E2E 검증을 둔다.

```text
Runner
→ Playwright
→ 회원가입
→ 로그인
→ 잘못된 입력
→ timeout/retry
→ accessibility
→ multi-step form
```

1,200개 시나리오 실행 예:

```text
scenarios: 1200
passed: 1187
failed: 13

failure groups:
- timeout: 8
- validation mismatch: 3
- accessibility: 2
```

Agent Worker에게 모든 browser log를 전달하지 않는다.

필요한 실패의 screenshot, trace, console log만 추가로 조회한다.

## Event-driven Agent

Agent는 상시 프로세스가 아니다.

다음 이벤트를 Agent 활성화 조건으로 사용할 수 있다.

- Runner failure
- PR CI failure
- review comment
- security alert
- scheduled regression failure
- 사람이 escalation 요청

Nightly 예:

```text
Nightly Scheduler
      ↓
 Full Runner
      ↓
 +----+----+
 |         |
PASS      FAIL
 |         |
종료   Result Filter
           ↓
      Agent Worker
           ↓
      Draft PR 후보
```

## Context 최소화

Agent Worker 호출 시 Repository 전체를 전달하지 않는다.

```text
Task:
AuthServiceTest.expiredToken 실패 수정

Relevant Files:
- AuthService.java
- AuthServiceTest.java
- JwtTokenProvider.java

Failure:
expected: 401
actual: 200

Location:
AuthServiceTest.java:94

Verify:
./gradlew test --tests AuthServiceTest.expiredToken

Forbidden Changes:
- DB schema
- 전체 JWT 구조
- 공통 Exception API
```

Agent가 추가 Context를 요구하면 단계적으로 범위를 넓힌다.

## Agent Harness

Repository에 다음 실행 도구를 준비한다.

```text
scripts/
├─ verify-fast.sh
├─ verify-full.sh
├─ test-unit.sh
├─ test-integration.sh
├─ test-e2e.sh
├─ validate-migration.sh
└─ filter-result.sh
```

추가 구성:

- architecture 문서
- coding convention
- Agent Contract
- 작은 Task Contract
- machine-readable result
- artifact 규칙
- PR template

목표는 Agent가 실패할 때마다 프롬프트를 길게 만드는 것이 아니라 프로젝트 자체가 반복 가능한 실행·검증 인터페이스를 제공하는 것이다.

## Fast / Full Verification

### Fast Verification

- compile
- unit test
- selected integration test
- static rule

작업 중 자주 실행한다.

### Full Verification

- full unit test
- full integration test
- container-based test
- E2E
- migration validation
- architecture test
- security/static checks

Full Verification은 여러 Runner로 분할할 수 있다.

## 실패 분류

### `INFRA_FAILURE`

- dependency download 실패
- disk 부족
- memory 부족
- container startup 실패

코드 Agent에 바로 전달하지 않는다.

### `TEST_FAILURE`

실패 테스트와 원인을 추출한 뒤 Agent Worker 후보로 전달한다.

### `MIGRATION_FAILURE`

migration id, DB error code, failed statement 위치를 반환한다.

### `STATIC_ANALYSIS_FAILURE`

rule id와 파일/라인을 반환한다.

### `E2E_FAILURE`

scenario, screenshot/trace path, 핵심 browser error를 반환한다.

## Observability

다음 지표를 서로 분리해 수집한다.

Runner:

- queue time
- execution time
- exit code
- retry count
- CPU/RAM 사용량
- raw log size
- artifact size

Result Filter:

- filtered result size
- extracted failure count

Agent Worker:

- invocation count
- input/output usage
- retry/turn count
- Context 확대 횟수

Compute resource와 LLM usage를 하나의 비용 값으로 합쳐 원인을 숨기지 않는다.

## 이후 구현

8장과 10장에서 실행 인터페이스와 Result Filter를 구체화한다.

17장에서는 Runner와 Agent Worker 지표를 함께 관찰하는 구조를 설계한다.

19장에서는 PM Agent가 Runner와 Agent Worker를 서로 다른 실행 자원으로 스케줄링하도록 확장한다.
