# 10장. Cloud Agent를 Test Runner처럼 사용하기

Cloud 환경에서 실행되는 모든 작업에 LLM이 필요한 것은 아니다.

9장에서 Prepared Cloud Environment를 만들었다면 그 환경은 Agent 전용 공간이 아니라 반복 가능한 실행 노드가 된다.

Build, Test, E2E, Docker Build처럼 명령과 판정 기준이 정해진 작업은 먼저 Runner가 처리할 수 있다.

```text
Cloud Runner
→ 정해진 명령 실행
→ PASS / FAIL 판정

Cloud Agent
→ 실패 원인 분석
→ 코드 수정 판단
→ 변경 수행
```

이 장의 원칙은 간단하다.

> 정상 경로는 Runner가 처리하고, 판단이 필요한 예외 경로에서만 Agent를 호출한다.

> 검증할 수 있는 것은 Agent에게 묻지 말고 실행한다.

---

## 1. Cloud Runner와 Cloud Agent를 분리한다

다음 명령은 LLM이 없어도 실행할 수 있다.

```bash
./gradlew test
```

필요한 것은 Repository와 Runtime, CPU/RAM, Test Tool이다.

반대로 다음 질문은 판단이 필요하다.

```text
AuthServiceTest.expiredToken이 왜 실패했는가?
어느 코드를 바꿔야 하는가?
```

따라서 한 Task 안에서도 실행 주체가 바뀐다.

```text
Runner
→ Test
→ FAIL

Agent
→ Analyze / Fix

Runner
→ Verification
```

Cloud Agent를 많이 호출하는 것이 목적이 아니다. Agent가 필요한 구간을 좁히는 것이 목적이다.

---

## 2. Deterministic First

개발 과정에는 프로그램으로 판정할 수 있는 검증이 많다.

```text
Compile
Unit Test
Integration Test
Lint
Architecture Rule
Migration Validation
Docker Build
E2E
```

이런 결과를 자연어 판단으로 대체하지 않는다.

```text
좋지 않은 방식
→ 코드를 보고 문제없어 보이는지 판단해줘

권장 방식
→ ./gradlew test
→ exit code / report 확인
```

가능하면 완료 조건을 실행 가능한 명령으로 만든다.

```text
Formatting
→ Formatter / Linter

API Compatibility
→ Contract Test

Migration
→ Disposable DB
```

판단을 코드로 만들 수 있다면 그 판단을 반복해서 LLM에 맡기지 않는다.

---

## 3. 성공 경로에서는 Agent를 호출하지 않는다

설명용 예시로 100개의 검증 Task가 있다고 하자.

```text
100 Runner Execution
→ 95 PASS
→ 5 FAIL
```

95개 PASS 경로는 그대로 종료할 수 있다.

실패 5개도 모두 Agent 작업은 아니다.

```text
5 FAIL
├─ Infra Failure
├─ Auto-fix 가능한 규칙 위반
└─ Code Reasoning이 필요한 Failure
```

Agent는 마지막 경우에만 필요하다.

```text
Runner
→ PASS → Done
→ FAIL
    ↓
Failure Classification
    ↓
Reasoning 필요?
    ├─ NO → Retry / Tool / Environment 처리
    └─ YES → Cloud Agent
```

Cloud Agent 비용을 줄이는 가장 단순한 방법 중 하나는 Agent 한 번의 Token을 조금 줄이는 것이 아니라 불필요한 Agent 호출 자체를 없애는 것이다.

---

## 4. Runner 입력과 출력도 고정한다

Runner는 단순 Shell 실행이지만 어떤 Source를 검증했는지 추적할 수 있어야 한다.

입력 예:

```text
Task ID
Git SHA
Environment
Command
Timeout
Artifact Rule
```

출력 예:

```text
Status
Exit Code
Duration
Result Summary
Artifact Reference
```

예:

```text
TASK=AUTH-142
SHA=def456
COMMAND=./gradlew :auth:test
STATUS=FAIL
EXIT=1
RESULT=artifacts/AUTH-142/result.json
```

8장에서 만든 작은 구조화 결과를 Runner의 출력 경계로 사용할 수 있다.

핵심은 Runner의 결과가 특정 Git SHA와 연결되어 있다는 점이다.

---

## 5. FAIL이라고 모두 코드 문제는 아니다

Runner가 실패하면 먼저 실패 종류를 구분한다.

```text
FAIL
 ↓
Classification
 ├─ INFRA_FAILURE
 ├─ BUILD_FAILURE
 ├─ TEST_FAILURE
 ├─ E2E_FAILURE
 └─ MIGRATION_FAILURE
```

예를 들어 다음은 인프라 문제다.

```text
Registry timeout
Worker disk full
Dependency mirror unavailable
Container start failure
```

이런 실패를 Agent에게 보내 코드 수정을 시키면 잘못된 변경이 생길 수 있다.

```text
INFRA_FAILURE
→ Retry / Environment 처리
```

반대로 다음은 코드 분석 후보가 된다.

```text
Compilation Error
Assertion Failure
NullPointerException
UI selector mismatch
Migration syntax error
```

그중 자동 수정 도구가 해결할 수 있는 것은 Tool을 먼저 사용한다.

---

## 6. Agent-on-failure

코드 판단이 필요한 실패가 남으면 Agent를 호출한다.

```text
Runner
  ↓
FAIL
  ↓
Failure Summary
  ↓
Cloud Agent
  ↓
Analyze / Fix
  ↓
Commit
  ↓
Runner
  ↓
Verification
```

AUTH-142를 예로 들면 Agent가 처음 받아야 할 정보는 대형 로그가 아니다.

```text
Test
AuthServiceTest.expiredToken

Expected
401

Actual
200

Relevant Files
AuthService.java
JwtTokenProvider.java
AuthServiceTest.java
```

필요한 경우에만 8장의 Result Gateway에서 상세 Artifact를 추가 조회한다.

10장의 핵심은 Failure Summary 형식을 다시 정의하는 것이 아니라 **Runner와 Agent 사이의 호출 경계**를 만드는 것이다.

---

## 7. Agent가 수정한 뒤에는 Runner가 다시 판정한다

다음 흐름으로 끝내지 않는다.

```text
Agent
→ Code Fix
→ "수정 완료"
→ PR
```

수정 후 같은 검증을 다시 실행한다.

```text
Agent Fix
→ Runner
→ PASS / FAIL
```

가능하면 처음 실패한 명령부터 실행한다.

```bash
./gradlew test --tests AuthServiceTest.expiredToken
```

Target Test가 PASS하면 필요한 범위까지 회귀 검증을 넓힌다.

```text
Target Test
→ Related Test
→ Module Test
→ 필요한 Full Verification
```

Agent의 설명과 실행 사실을 분리하는 과정이다.

---

## 8. Retry 판단도 Runner 결과를 기준으로 한다

수정 후 다시 실패하면 8장의 Failure Fingerprint를 사용한다.

```text
Agent Fix
→ Runner
→ FAIL
→ Fingerprint
```

실패가 바뀌었다면 진행 중일 수 있다.

```text
1차
Compilation Error

2차
Assertion Failure
```

반대로 동일 Fingerprint가 반복된다면 같은 시도를 계속할 이유가 줄어든다.

```text
Retry #1 → Fingerprint A
Retry #2 → Fingerprint A
```

구체적인 중단과 Local Fallback 기준은 17장에서 다룬다.

여기서는 Runner 결과가 다음 행동을 결정하는 Evidence가 된다는 점만 유지한다.

---

## 9. 독립 검증은 병렬 Runner로 실행할 수 있다

하나의 Git SHA에 여러 검증이 필요할 수 있다.

```text
Git SHA: def456
        ↓
+---------+---------+---------+---------+
|         |         |         |         |
Unit   Integration E2E      Docker
|         |         |         |
+---------+---------+---------+---------+
          ↓
        Fan-in
```

이때 모든 결과는 같은 Source 상태를 검증해야 한다.

```text
Unit        → def456
Integration → def456
E2E         → def456
Docker      → def456
```

서로 다른 SHA의 결과를 하나의 PASS 묶음으로 취급하면 안 된다.

병렬화의 중복 Context, Review, Merge 비용은 12장에서 다룬다. 10장에서는 먼저 **결정론적 검증을 여러 Runner로 분리할 수 있다**는 점만 사용한다.

---

## 10. Sharding은 실행시간과 시작비용을 함께 본다

큰 Test Suite는 여러 Runner로 나눌 수 있다.

```text
Test Suite
→ Shard A
→ Shard B
→ Shard C
→ Shard D
```

적합한 조건:

```text
Shard 간 State 공유가 적음
각 결과를 합산 가능
실패 Test 식별 가능
같은 Git SHA 사용
```

하지만 Worker Provisioning과 Environment Restore에도 비용이 있다.

짧은 테스트를 너무 잘게 나누면 시작비용이 실행시간보다 커질 수 있다.

따라서 Shard 수는 고정 규칙이 아니라 프로젝트의 실제 실행시간과 준비시간을 보고 정한다.

---

## 11. campus-platform에 적용하면

출결 인증 변경 Commit이 있다고 하자.

```text
SHA: def456
```

Cloud Runner가 먼저 검증한다.

```text
Unit        PASS
Integration FAIL
Docker      PASS
E2E         PASS
```

Integration Failure가 인프라 문제가 아니라 재현 가능한 코드 Failure라면 Agent가 해당 실패만 분석한다.

```text
Integration Runner
→ FAIL
→ Failure Summary
→ Cloud Agent
→ Fix Commit ghi789
→ Integration Runner
→ PASS
```

수정 SHA가 바뀌었으므로 필요한 검증은 `ghi789` 기준으로 다시 맞춘다.

이 구조에서 Agent는 전체 검증 파이프라인을 대신하는 존재가 아니다.

판단과 변경이 필요한 지점에 들어가는 Worker다.

---

## 12. 10장에서 기억할 실행 규칙

```text
Prepared Environment
        ↓
Runner
  ├─ PASS → Evidence → Done
  └─ FAIL
       ↓
   Classification
       ↓
   Tool / Retry / Agent
       ↓
   수정이 있으면 Runner 재검증
```

정리하면 다음 세 문장으로 충분하다.

> Runner가 할 수 있으면 Runner에게 맡긴다.

> FAIL이라고 모두 Agent를 호출하지 않는다.

> Agent가 수정했으면 다시 Runner가 검증한다.

다음 장에서는 여러 Runner와 Agent가 동시에 작업할 때 Source와 Runtime이 서로 섞이지 않도록 Git, Branch, Worktree, Container를 이용해 작업을 격리한다.
