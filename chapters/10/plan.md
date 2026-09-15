# 10장 설계 - Cloud Agent를 Test Runner처럼 사용하기

## 장의 목표

Cloud 환경에서 실행되는 모든 작업에 LLM을 붙이지 않고, 결정론적 실행은 Cloud Runner가 처리하고 실패 분석과 코드 수정이 필요한 경우에만 Cloud Agent를 호출하는 구조를 설계한다.

9장에서 Prepared Cloud Environment를 만들었다면, 10장에서는 그 환경을 `반복 가능한 실행 노드`로 사용한다.

핵심 질문:

> Cloud 환경에서 무엇은 Runner가 처리하고, 어떤 순간에만 Agent를 호출해야 하는가?

---

## 핵심 주장

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.

> 판단을 코드로 만들 수 있다면 LLM에게 판단시키지 않는다.

> Runner가 할 수 있으면 Runner에게 맡긴다.

기본 구조:

```text
Task
  ↓
Cloud Runner
  ↓
Build / Test / Validation
  ↓
+----+----+
|         |
PASS      FAIL
|         |
Done   Result Gateway
          ↓
     판단 필요?
      ├─ NO → deterministic 처리 / 종료
      └─ YES → Cloud Agent
                 ↓
               Fix
                 ↓
              Runner
                 ↓
            Verification
```

---

## 독자가 얻는 것

- Cloud Runner와 Cloud Agent의 책임을 구분할 수 있다.
- Build/Test/E2E/Docker 작업을 LLM 없이 반복 실행할 수 있다.
- PASS 경로에서 Agent 호출을 제거할 수 있다.
- FAIL을 바로 Agent에게 넘기지 않고 실패 종류를 먼저 분류할 수 있다.
- Runner 결과를 8장의 Result Gateway와 연결할 수 있다.
- CPU/RAM 작업과 Token 소비를 분리할 수 있다.
- Unit/Integration/E2E/Docker 검증을 여러 Runner로 병렬화할 수 있다.
- 수정 후 반드시 Runner가 다시 검증하는 폐루프를 만들 수 있다.

---

# 절 구성

## 10.1 Cloud Runner는 Agent가 아니다

Cloud Runner 특징:

- LLM 사용 없음
- 정해진 명령 실행
- CPU/RAM 중심
- exit code로 판정 가능
- 반복 실행에 적합
- Artifact 생성 가능

Cloud Agent 특징:

- 원인 분석
- 코드 탐색
- 수정 방향 결정
- 코드 수정
- 설계 판단
- Token/Context 사용

구분 예:

```text
./gradlew test
→ Runner

왜 테스트가 실패했는지 분석하고 수정
→ Agent
```

---

## 10.2 Deterministic First

좋지 않은 방식:

```text
Agent:
"코드를 검토해보니 문제 없어 보입니다."
```

권장 방식:

```text
./gradlew test
PASS

./gradlew check
PASS

architecture-check
PASS

migration-check
PASS
```

검증 후보:

- Compile
- Unit Test
- Integration Test
- Contract Test
- E2E
- Lint
- Static Analysis
- Architecture Rule
- Migration Validation
- Secret Scan
- Vulnerability Scan
- Docker Build

기계적으로 판정할 수 있는 결과는 자연어 판단보다 우선한다.

---

## 10.3 성공 경로에서 Agent를 없앤다

예:

```text
100 Task
95 PASS
5 FAIL
```

비효율적 구조:

```text
100 Task
→ 100 Agent Session
```

권장 구조:

```text
100 Runner Execution
95 PASS → 종료
5 FAIL → 실패 분석 필요 시 Agent
```

Cloud Agent 비용 최적화의 첫 단계는 Agent 호출 자체를 줄이는 것이다.

---

## 10.4 Runner 종류를 역할별로 분리한다

논리적 Runner 후보:

```text
runner-build
runner-unit
runner-integration
runner-e2e
runner-docker
runner-migration
runner-static
runner-security
```

물리적으로 항상 별도 VM일 필요는 없다.

목적은 다음을 분리하는 것이다.

- 필요한 Environment
- 실행 명령
- Timeout
- Artifact
- Compute 요구량

예:

```text
unit
→ backend-test environment

E2E
→ frontend-e2e environment
```

9장의 Task-specific Environment와 연결한다.

---

## 10.5 Test Runner의 기본 계약

Runner 입력:

```text
Task ID
Git SHA
Branch
Command
Environment
Timeout
Artifact Rules
```

Runner 출력:

```text
Status
Exit Code
Duration
Result Summary
Artifact References
```

예:

```text
TASK=task-142
SHA=abc123
COMMAND=./gradlew test --tests AuthServiceTest
STATUS=FAIL
EXIT=1
DURATION=42s
RESULT=result.json
```

Agent 없이도 자동화 시스템이 해석 가능한 형태로 만든다.

---

## 10.6 FAIL도 종류가 다르다

FAIL이 발생했다고 모두 코드 수정 Agent를 호출하지 않는다.

예:

```text
INFRA_FAILURE
- registry timeout
- worker disk full
- network unavailable

TEST_FAILURE
- assertion failure
- exception

BUILD_FAILURE
- compile error

E2E_FAILURE
- selector mismatch
- timeout

MIGRATION_FAILURE
- syntax
- constraint
```

분류:

```text
FAIL
 ↓
Failure Classification
 ├─ Infra → Retry/Environment 처리
 ├─ Deterministic Auto-fix → Tool
 └─ Code Reasoning 필요 → Agent
```

Infrastructure Failure를 코드 Agent에게 바로 보내면 잘못된 수정과 Token 낭비가 발생할 수 있다.

---

## 10.7 Result Gateway와 연결한다

Runner는 전체 로그를 Agent에게 직접 보내지 않는다.

```text
Runner
→ Raw Artifact
→ Result Gateway
→ Failure Summary
→ Agent
```

Agent 입력 예:

```text
STATUS: TEST_FAILURE
Test: AuthServiceTest.expiredToken
expected: 401
actual: 200
Stack: AuthServiceTest.java:94
```

필요한 경우에만 상세 로그를 조회한다.

---

## 10.8 Agent-on-failure

코드 판단이 필요한 실패만 Agent를 호출한다.

```text
Runner FAIL
    ↓
Result Gateway
    ↓
Agent Worker
    ↓
Analyze
    ↓
Fix
    ↓
Commit
    ↓
Runner 재실행
```

Agent가 `수정 완료`라고 말하는 것으로 끝내지 않는다.

최종 상태는 Runner의 Verification으로 결정한다.

---

## 10.9 수정 후에는 같은 Runner 계약으로 재검증한다

좋지 않은 흐름:

```text
Agent Fix
→ Agent가 스스로 성공 선언
```

권장 흐름:

```text
Agent Fix
→ Runner
→ PASS/FAIL
```

처음 실패를 만든 명령과 가능한 한 같은 검증 명령을 재사용한다.

예:

```text
Before
./gradlew test --tests AuthServiceTest.expiredToken
FAIL

After Fix
./gradlew test --tests AuthServiceTest.expiredToken
PASS
```

그 다음 회귀 범위를 넓힌다.

```text
Target Test
→ Module Test
→ Full Verification
```

---

## 10.10 Parallel Test

독립적인 검증은 여러 Runner로 fan-out할 수 있다.

```text
Git SHA
  ↓
+---------+---------+---------+---------+
|         |         |         |         |
Unit    Integration E2E      Docker
|         |         |         |
+---------+---------+---------+---------+
          ↓
       Fan-in
          ↓
    Verification Result
```

모든 Runner는 같은 Git SHA를 검증해야 한다.

서로 다른 commit을 검증한 결과를 한 작업의 Evidence로 합치지 않는다.

---

## 10.11 Test Sharding

대형 Test Suite는 실행 단위를 나눌 수 있다.

예:

```text
10,000 tests
→ shard A
→ shard B
→ shard C
→ shard D
```

조건:

- shard 간 상태 공유 최소화
- 결과 합산 가능
- 실패 테스트 식별 가능
- 동일 Git SHA

단, 무조건 shard 수를 늘리지 않는다.

환경 준비와 scheduling overhead가 테스트 실행시간보다 커질 수 있다.

---

## 10.12 Runner Timeout과 Resource 설정

Runner는 Agent와 별도 Budget을 가진다.

예:

```text
unit-test
cpu: small
timeout: 10m

integration
cpu: medium
memory: high
timeout: 30m

e2e
browser: required
timeout: 40m
```

정확한 값은 프로젝트 측정치로 결정한다.

Task가 Timeout된 경우 코드 실패와 Compute 부족을 구분해야 한다.

---

## 10.13 Runner-first가 Token을 줄이는 이유

Token이 필요한 시점:

- 실패 분석
- 코드 탐색
- 수정 판단
- 코드 작성

Token이 필요하지 않은 시점:

- Gradle compile 진행
- JUnit 8,000건 실행
- Docker layer build
- Browser scenario 반복

따라서:

```text
LLM
→ 무엇을 할지 판단

Runner
→ 많이 실행

LLM
→ 필요한 실패만 판단
```

3장의 Compute/Token 분리를 실제 실행 구조로 구현한다.

---

## 10.14 Event-driven 호출의 기반

이 장에서는 이벤트 자체보다 Runner 결과가 Agent 호출 조건이 된다는 점까지만 설명한다.

```text
CI / Nightly / Dependency Update
        ↓
      Runner
        ↓
     PASS / FAIL
        ↓
     FAIL only
        ↓
       Agent
```

14장에서 Issue/CI/Review/Schedule을 Task Queue와 연결한다.

---

## 10.15 campus-platform Runner 구성

```text
runner-unit
→ ./gradlew test

runner-integration
→ Spring Boot + PostgreSQL/Testcontainers

runner-e2e
→ Playwright

runner-docker
→ docker build

runner-migration
→ PostgreSQL migration validation
```

Tibero 실제 검증은 Cloud Runner 범위 밖으로 남기고 13장의 Local Handoff에서 처리한다.

예제 흐름:

```text
PR SHA
  ↓
runner-unit ───────── PASS
runner-integration ─ PASS
runner-docker ─────── PASS
runner-e2e ────────── FAIL
                         ↓
                   Result Gateway
                         ↓
                    Cloud Agent
                         ↓
                      Fix
                         ↓
                    runner-e2e
                         ↓
                        PASS
```

---

# 좋은 사례와 나쁜 사례

## Agent가 모든 테스트를 관리

좋지 않은 방식:

```text
Agent
→ test 실행
→ 계속 상태 확인
→ 전체 로그 읽기
→ PASS 여부 판단
```

권장 방식:

```text
Runner
→ test
→ result.json
→ PASS면 종료
```

## Infra 실패를 코드 수정

좋지 않은 방식:

```text
Registry timeout
→ Agent가 Dockerfile 수정
```

권장 방식:

```text
Infra Failure 분류
→ retry/environment 처리
```

## 수정 후 검증 생략

좋지 않은 방식:

```text
Agent Fix
→ PR
```

권장 방식:

```text
Agent Fix
→ Runner Verification
→ Evidence
→ PR
```

---

# 필요한 그림

1. Runner-first / Agent-on-failure
2. Failure Classification
3. Multi Runner Fan-out/Fan-in
4. Agent Fix → Runner Reverification Loop
5. Compute vs Token 실행 흐름

---

# Phase 6 구현 후보

```text
scripts/
├─ verify-unit.sh
├─ verify-integration.sh
├─ verify-e2e.sh
├─ verify-docker.sh
└─ classify-failure.py
```

공통 결과:

```text
artifacts/<task-id>/result.json
```

---

# 필요한 조사

- Gradle test filtering/sharding 적용 방식
- JUnit XML
- Playwright parallel/sharding
- GitHub Actions/Jenkins의 matrix/fan-out 사례
- 현재 Cloud coding agent의 CI failure activation 사례

제품 기능은 원칙과 분리한다.

---

# 앞뒤 장 연결

9장:

```text
Runner가 빠르게 시작할 Environment를 준비
```

10장:

```text
Prepared Environment에서 결정론적 작업 반복
```

11장:

```text
여러 Task/Worker가 서로 간섭하지 않도록 Git/Branch/Container 격리
```

---

# 의도적으로 다루지 않을 내용

- 범용 CI/CD 입문
- Kubernetes Scheduler 설계
- Agent Platform control plane
- 범용 Workflow DSL

---

# 장의 결론 메시지

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.

> 판단을 코드로 만들 수 있다면 LLM에게 판단시키지 않는다.

> Agent가 수정한 결과도 Runner가 다시 검증한다.
