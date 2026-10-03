# 부록 A. Task Contract / Evidence / Handoff 템플릿

이 부록은 본문의 개념을 다시 설명하지 않는다. 로컬과 클라우드 사이에서 작업을 넘기고 결과를 검증할 때 바로 복사해 사용할 수 있는 최소 템플릿만 제공한다.

프로젝트 상황에 맞게 필드를 줄이거나 추가할 수 있다. 다만 소스 코드 상태와 검증 결과가 연결되는 관계는 유지한다.

## A.1 Task Contract

```yaml
task_id: AUTH-142
base_sha: abc123

goal: >
  만료된 JWT로 Attendance API를 호출하면 HTTP 401을 반환한다.

scope:
  modules:
    - auth
    - attendance

relevant_files:
  - auth/src/main/java/.../AuthService.java
  - auth/src/main/java/.../JwtTokenProvider.java
  - auth/src/test/java/.../AuthServiceTest.java

do_not_change:
  - DB Schema
  - OAuth 전체 구조
  - 공통 Exception format

environment: backend-test

validation:
  - ./gradlew test --tests AuthServiceTest

expected_result:
  - expired token -> 401
  - normal token -> existing tests PASS

expected_output:
  - Result SHA
  - Changed Files
  - Validation Result
  - Artifact Reference
  - Limitations
```

핵심은 프롬프트 길이가 아니라 다음 질문에 답하는 것이다.

```text
어디서 시작하는가?
무엇을 바꾸는가?
무엇을 바꾸면 안 되는가?
무엇이 성공인가?
무엇을 돌려받아야 하는가?
```

## A.2 Cloud Runner 입력

```yaml
task_id: AUTH-142
sha: def456
environment: backend-test
command: ./gradlew test --tests AuthServiceTest
timeout_seconds: 1800
artifact_path: artifacts/AUTH-142/def456/
```

실행기 입력에는 자연어 설명보다 어떤 소스 코드 상태에서 어떤 명령을 실행하는지가 중요하다.

## A.3 Cloud Runner 출력

```yaml
task_id: AUTH-142
result_sha: def456
status: FAIL
exit_code: 1
duration_seconds: 73
validation_result: FAILED
artifact_path: artifacts/AUTH-142/def456/
```

실패가 발생하면 Result Gateway가 에이전트가 먼저 읽을 작은 실패 요약을 만든다.

```yaml
failure:
  type: TEST_FAILURE
  test: AuthServiceTest.expiredToken
  expected: 401
  actual: 200
  top_frame: AuthServiceTest.java:94
  fingerprint: TEST_FAILURE/401-200/AuthServiceTest:94
```

## A.4 Evidence 결과

```json
{
  "task_id": "AUTH-142",
  "base_sha": "abc123",
  "result_sha": "def456",
  "validation_result": "PASS",
  "changed_files": [
    "AuthService.java",
    "AuthServiceTest.java"
  ],
  "checks": {
    "target_test": "PASS",
    "module_test": "PASS",
    "integration": "PASS"
  },
  "artifacts": [
    "artifacts/AUTH-142/def456/result.json",
    "artifacts/AUTH-142/def456/junit.xml"
  ],
  "limitations": []
}
```

검증 근거는 원본 결과물을 대체하지 않는다. 처음 판단하는 데 필요한 정보만 작게 제공한다.

## A.5 Local → Cloud Handoff Package

```yaml
task_id: AUTH-142
repository: campus-platform
base_sha: abc123
branch: agent/auth-142
contract: task-contract.yaml
environment: backend-test
validation:
  - ./gradlew test --tests AuthServiceTest
expected_evidence:
  - Result SHA
  - Changed Files
  - Validation Result
  - Artifact Reference
```

여러 저장소에 걸친 작업이라면 저장소별 Base SHA를 따로 기록한다.

```yaml
repositories:
  campus-api: aaa111
  campus-common: bbb222
```

## A.6 Cloud → Local Return Package

```yaml
task_id: AUTH-142
base_sha: abc123
result_sha: def456
branch: agent/auth-142
validation_result: PASS
changed_files:
  - AuthService.java
  - AuthServiceTest.java
artifact_path: artifacts/AUTH-142/def456/
pr: 142
limitations:
  - Tibero 실제 환경 검증 필요
```

로컬에서는 최소한 다음 관계를 확인한다.

```text
Result SHA
=
Evidence가 검증한 SHA
=
PR에서 Review하는 SHA
```

코드가 변경됐다면 필요한 검증을 새 SHA에서 다시 실행한다.

## A.7 Local Fallback Package

클라우드 이점이 사라졌을 때 현재 작업을 버리지 않고 로컬에서 이어가기 위한 반환 형식이다.

```yaml
task_id: HSM-37
status: LOCAL_FALLBACK
base_sha: abc123
result_sha: def456

validation_result:
  unit: PASS
  mock_contract: PASS

failed_command: null
failure_fingerprint: null

artifact_path: artifacts/HSM-37/def456/

attempted_fixes:
  - HSM 호출 경계를 Mock으로 분리

fallback_reason: actual HSM required

remaining:
  - 실제 HSM session validation
```

클라우드에서 확인한 사실과 아직 확인하지 못한 경계를 분리해서 남긴다.

## A.8 Event-driven Task Candidate

이벤트는 바로 에이전트 호출이 아니라 작업 후보로 변환한다.

```yaml
source_event: ci_failure
repository: campus-platform
git_sha: abc123
task_type: test_failure_fix
failure: AuthServiceTest.expiredToken
failure_fingerprint: TEST_FAILURE/401-200/AuthServiceTest:94
environment: backend-test
validation: ./gradlew test --tests AuthServiceTest.expiredToken
status: candidate
```

이후 공통 경로를 사용한다.

```text
Event
→ Task Candidate
→ Dedup / Classification
→ Cloud Runner / Tool
→ 판단이 필요할 때 Cloud Agent
→ Cloud Runner 재검증
→ Evidence / PR
```

## A.9 최소 운영 상태

처음부터 별도 에이전트 플랫폼을 만들지 않아도 다음 상태만 추적하면 기본 작업 흐름을 운영할 수 있다.

```yaml
task_id: AUTH-142
base_sha: abc123
branch: agent/auth-142
result_sha: def456
execution: cloud-agent
status: verifying
validation_result: PASS
artifact_path: artifacts/AUTH-142/def456/
pr: 142
```

책 전체에서 유지하는 기본 관계는 다음과 같다.

```text
Task ID
→ Base SHA
→ Branch
→ Result SHA
→ Validation Result
→ Evidence / Artifact
→ PR
```

양식을 복잡하게 만드는 것보다 이 관계가 끊기지 않는 것이 중요하다.
