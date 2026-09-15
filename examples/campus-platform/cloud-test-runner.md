# Cloud Test Runner 설계

이 문서는 `campus-platform` 예제에서 Cloud Agent의 실행 환경을 Build/Test/Validation 노드로 사용하는 구조를 정의한다.

구현 코드는 아직 작성하지 않는다.

## 목적

Cloud Agent를 모든 작업을 수행하는 범용 Agent로만 사용하지 않는다.

다음 두 역할을 구분한다.

### Agent Worker

판단이 필요한 작업을 수행한다.

- 코드 탐색
- 설계 판단
- 구현
- 복잡한 오류 분석
- Review

### Test Runner

결정론적으로 실행 가능한 검증 작업을 수행한다.

- Build
- Unit Test
- Integration Test
- Docker Build
- Static Analysis
- Migration Validation
- E2E Test

핵심 원칙:

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

## 기본 구조

```text
PM / Local Agent
      ↓
Task Scheduler
      ↓
Cloud Test Runner
      ↓
Build / Test / Validation
      ↓
Raw Result
      ↓
Result Filter
      ↓
Structured Summary
      ↓
PM / Agent Worker
```

## 실행 정책

가능한 작업은 먼저 Test Runner에서 처리한다.

```text
Task
  ↓
Test Runner
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
   Test Runner 재실행
```

성공한 작업은 별도의 LLM 분석을 요구하지 않는다.

## Runner 유형

초기 예제에서는 다음 논리적 Runner를 정의한다.

```text
cloud-runner-unit
cloud-runner-integration
cloud-runner-docker
cloud-runner-migration
cloud-runner-static-analysis
```

실제 인프라는 특정 Cloud Agent 제품에 종속시키지 않는다.

## 병렬 실행 예

```text
Local PM Agent
|
+-- Cloud Session #1
|    Backend Unit Test
|
+-- Cloud Session #2
|    Integration Test
|
+-- Cloud Session #3
|    Notification Module Test
|
+-- Cloud Session #4
|    Docker Build
|
+-- Cloud Session #5
     Migration Validation
```

각 Runner는 Repository 전체를 분석하지 않는다.

Scheduler가 최소한 다음 정보를 전달한다.

- task id
- 실행 명령
- 관련 module/path
- timeout
- 완료 조건
- artifact/log 저장 위치

## Result Filter

Raw stdout/stderr를 그대로 LLM에 전달하지 않는다.

Result Filter는 다음 정보를 추출한다.

- exit code
- status
- test total/passed/failed
- failed test names
- root cause 후보
- 핵심 stack trace
- 중요 warning
- artifact 경로
- raw log 경로

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

## 로그 처리 원칙

원본 로그는 artifact로 보존한다.

LLM에는 기본적으로 summary만 전달한다.

추가 분석이 필요하면 다음 순서로 Context를 확대한다.

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

처음부터 Full Raw Log를 전달하지 않는다.

## Task Context 최소화

예:

```text
Task: 회원 탈퇴 API 테스트

Relevant Files:
- UserController.java
- UserService.java
- UserRepository.java
- UserServiceTest.java

Command:
./gradlew test --tests UserServiceTest

Acceptance Criteria:
- 정상 탈퇴 PASS
- 존재하지 않는 사용자 404
- 이미 탈퇴한 사용자 409

Forbidden Changes:
- DB Schema
- auth module
- common exception model
```

Test Runner는 위 범위에서 명령 실행과 결과 수집에 집중한다.

## Fast / Full Verification과의 연결

기존 테스트 전략과 다음처럼 연결한다.

### Fast Verification

- compile
- unit test
- selected integration test
- static rule

변경 작업 중 자주 실행한다.

### Full Verification

- full unit test
- full integration test
- container-based test
- migration validation
- architecture test
- security/static checks

PR 또는 작업 완료 판단에 사용한다.

Full Verification은 여러 Cloud Runner로 분할할 수 있다.

## 실패 처리

### 실행 환경 실패

예:

- dependency download 실패
- disk 부족
- memory 부족
- container startup 실패

Agent Worker 코드 수정으로 바로 연결하지 않는다.

`INFRA_FAILURE`로 분류한다.

### 테스트 실패

`TEST_FAILURE`로 분류하고 Result Filter가 관련 테스트와 원인을 추출한다.

### Migration 실패

`MIGRATION_FAILURE`로 분류하고 migration id, DB error code, failed statement 위치를 반환한다.

### 정적 분석 실패

`STATIC_ANALYSIS_FAILURE`로 분류하고 rule id와 파일/라인을 반환한다.

## Observability

Cloud Runner의 다음 지표를 별도로 수집한다.

- queue time
- execution time
- exit code
- retry count
- CPU/RAM 사용량을 제공할 수 있는 환경에서는 해당 지표
- raw log size
- filtered result size
- Agent Worker 호출 여부
- LLM usage와 cost

실행 자원과 LLM usage를 하나의 비용 값으로 합치지 않는다.

## 제품 독립성

Claude Code Web은 이 패턴을 설명하는 사례 중 하나다.

예제의 인터페이스는 특정 제품에 종속되지 않으며 다음과 같은 실행 환경으로 교체할 수 있어야 한다.

- GitHub Actions Runner
- Jenkins Agent
- Kubernetes Job
- 자체 VM
- 다른 Cloud Coding Agent의 sandbox

## 이후 구현 후보

8장과 10장 구현 단계에서 다음 인터페이스를 구체화한다.

```text
scripts/
├─ verify-fast.sh
├─ verify-full.sh
├─ test-unit.sh
├─ test-integration.sh
├─ validate-migration.sh
└─ filter-result.sh
```

구체적인 파일명은 실제 구현 단계에서 프로젝트 구조와 함께 결정한다.
