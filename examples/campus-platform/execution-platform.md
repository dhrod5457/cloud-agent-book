# campus-platform Agent-friendly Execution Platform

이 문서는 `campus-platform` 예제에서 Cloud Runner, Agent Worker, Result Gateway, Cache, Artifact Store, Budget을 하나의 실행 플랫폼으로 구성하는 설계안이다.

Phase 5에서는 구현하지 않는다.

## 목표

```text
정상 작업
→ deterministic Runner
→ PASS면 종료

판단이 필요한 실패
→ Result Gateway
→ Agent Worker
→ 수정
→ Runner 재검증
```

핵심 원칙:

- LLM은 판단하고 컨테이너는 실행한다.
- 정상 경로는 Runner가 처리한다.
- 환경은 미리 준비한다.
- 결과는 버리지 않고 Artifact로 저장한다.
- Agent는 필요한 Context만 조회한다.
- Agent의 retry와 token에는 Budget을 둔다.

## 실행 플랫폼 구성

```text
                PM / Orchestrator
                       |
                Task Scheduler
                       |
               Task Classification
                       |
        +--------------+--------------+
        |                             |
 Deterministic                   Reasoning Required
        |                             |
        v                             v
  Cloud Runner                   Agent Worker
        |                             |
 Build/Test/E2E                  Analyze/Fix
        |                             |
        +------------+----------------+
                     |
                Result Gateway
                     |
              +------+------+
              |             |
             PASS          FAIL
              |             |
             Done       Budget Check
                            |
                       +----+----+
                       |         |
                     Retry   Human Escalation
```

기반 계층:

```text
Prepared Image
Reusable Cache
Artifact Store
Repository Harness
Progressive Documentation
Isolated Worktree / Container
```

## Prepared Image

후속 구현에서 다음 도구가 준비된 이미지를 사용한다.

```text
Java 21
Gradle
Node
Playwright browser
Docker CLI
Static analysis tools
Security scanning tools
```

가능한 경우 dependency와 browser binary를 warming한다.

Agent Worker에게 JDK나 Node를 직접 설치하도록 하지 않는다.

## Cache 정책

### Reusable Cache

- Gradle dependency cache
- Maven repository가 필요한 보조 도구
- npm cache
- Docker layers
- Playwright browser
- compiler cache
- immutable code generation output

### Disposable Runtime State

- PostgreSQL/Testcontainers DB data
- Redis/Kafka runtime state
- temporary file
- user-generated test data
- test output
- browser session
- process state

Cache hit가 재현성을 깨뜨리지 않도록 runtime state는 매 작업 초기화한다.

## Runner 종류

```text
runner-build
runner-unit
runner-integration
runner-e2e
runner-docker
runner-migration
runner-architecture
runner-static-analysis
runner-security
```

모든 Runner는 exit code와 machine-readable report를 남긴다.

## Deterministic Validation

예상 검증 인터페이스:

```text
./scripts/build.sh
./scripts/test-unit.sh
./scripts/test-integration.sh
./scripts/test-e2e.sh
./scripts/check-architecture.sh
./scripts/validate-migration.sh
./scripts/security-scan.sh
./scripts/verify-full.sh
```

Agent가 다음을 자연어로 판단하지 않도록 한다.

- compile 성공
- test 성공
- architecture rule 준수
- migration 가능 여부
- secret/security rule 위반

## Result Gateway

각 작업은 다음 artifact를 남길 수 있다.

```text
artifacts/task-142/
├─ result.json
├─ junit.xml
├─ coverage.xml
├─ build.log
├─ git.diff
├─ container-info.json
├─ screenshots/
└─ video/
```

`result.json` 예:

```json
{
  "taskId": "task-142",
  "status": "FAIL",
  "exitCode": 1,
  "failureType": "TEST_FAILURE",
  "tests": {
    "total": 8214,
    "passed": 8211,
    "failed": 3
  },
  "failures": [
    "AuthServiceTest.expiredToken",
    "UserServiceTest.deleteUser",
    "UserMapperTest.insert"
  ],
  "artifactRoot": "artifacts/task-142"
}
```

Agent Worker는 기본적으로 `result.json`만 받는다.

추가 lookup 후보:

```text
failure detail
specific stack trace
specific test log
screenshot
browser trace
git diff
container information
```

## Progressive Context

JWT 관련 오류 예:

```text
Task Context Package
    ↓
AGENTS.md
    ↓
docs/security/auth.md
    ↓
AuthService.java
JwtTokenProvider.java
AuthServiceTest.java
    ↓
필요한 failure artifact
```

Repository 전체 문서를 처음부터 읽지 않는다.

## Failure Container Retention

기본 정책 후보:

```text
PASS
→ container 즉시 삭제

FAIL
→ container 30분 freeze
→ Agent/Human attach 가능
→ TTL 만료 시 삭제
```

특히 다음 오류에 사용한다.

- Testcontainers DB state
- race condition
- browser state
- filesystem issue
- network timeout
- process state

실제 TTL은 운영 비용과 보안 정책에 따라 정한다.

## Budgeted Autonomy

Task Contract에 다음 budget을 추가할 수 있다.

```text
max_turns
max_retry
max_tokens
max_wall_clock
max_changed_files
max_diff_size
```

예:

```text
Task: AuthServiceTest.expiredToken fix
max_retry: 2
max_turns: 8
max_tokens: 30000
```

실제 숫자는 모델과 서비스에 종속되므로 설계 예로만 사용한다.

## Failure Fingerprint

Result Gateway가 failure fingerprint를 생성한다.

예:

```text
TEST_FAILURE
AuthServiceTest.expiredToken
expected=401
actual=200
```

동일 fingerprint가 일정 횟수 반복되면 retry를 중단한다.

```text
same fingerprint x2
→ STOP
→ Human Escalation
```

## Fan-out / Fan-in

서로 독립적인 모듈은 병렬 검증할 수 있다.

```text
Scheduler
|
+-- student       → runner → result
+-- attendance    → runner → result
+-- notification  → runner → result
+-- integration   → runner → result
                      |
                      v
                    Fan-in
                      |
                Full Verification
```

공통 migration/schema/common source를 동시에 수정하는 작업은 병렬화 대상에서 제외한다.

## PM 분류 입력

PM/Orchestrator가 사용할 task metadata 후보:

```yaml
task_id: auth-expired-token
kind: bug-fix
deterministic: false
reasoning_required: true
network: public
module: auth
verify: ./gradlew test --tests AuthServiceTest.expiredToken
budget:
  max_retry: 2
```

Runner 전용 작업 예:

```yaml
task_id: full-unit-test
kind: unit-test
deterministic: true
reasoning_required: false
execution: cloud-runner
```

내부망 작업 예:

```yaml
task_id: tibero-integration-check
kind: database-debug
internal_network: true
execution: local-agent
```

## Observability

Runner 지표:

- queue time
- execution time
- cache hit ratio
- CPU/RAM
- artifact size
- exit code

Agent 지표:

- invocation count
- turns
- token usage
- Context lookup count
- retry count
- budget exhaustion

운영 지표:

- human escalation count
- repeated failure fingerprint
- average time to resolution
- PASS path Agent avoidance rate

## 구현 단계 연결

- 4장: Progressive Documentation과 Repository entry point
- 5장: AGENTS.md와 실행 계약
- 6장: Task Context Package와 Budget
- 8장: Runner command 표준화
- 10장: executable validation과 result schema
- 13장: worktree/container isolation
- 17장: metrics와 artifact observability
- 19장: task classification과 scheduling
