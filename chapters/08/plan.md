# 8장 설계 - Tool Output을 줄이고 Evidence를 남기기

## 장의 목표

7장에서 Cloud Agent에게 전달하는 `Task와 Context`를 작게 만드는 방법을 정의했다.

8장에서는 반대 방향인 `Cloud 실행 결과가 Agent와 사람에게 돌아오는 방식`을 설계한다.

Build, Test, E2E, Docker 실행은 수십 MB에서 수백 MB의 로그와 여러 Artifact를 만들 수 있다. 이 결과를 그대로 LLM Context에 넣으면 Token과 분석 비용이 증가하고, 중요한 실패 원인이 대량의 정상 로그에 묻힐 수 있다.

따라서 다음 구조를 기본으로 사용한다.

```text
Cloud Runner / Agent
        ↓
Raw Result / Artifact
        ↓
Result Filter / Result Gateway
        ↓
Small Evidence Summary
        ↓
Agent / Developer
        ↓
필요한 경우에만 상세 Artifact 조회
```

핵심 질문:

> 실행 결과를 잃지 않으면서 Cloud Agent가 실제로 읽어야 하는 정보량을 어떻게 줄일 것인가?

---

## 핵심 주장

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

> 큰 결과를 요약해서 버리는 것이 아니라, 큰 결과를 저장하고 필요한 부분만 조회한다.

> Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

7장과 8장의 관계는 다음과 같다.

```text
7장
작은 Task / 작은 Context
        ↓
Cloud Runner / Agent
        ↓
8장
작은 Result / 검증 가능한 Evidence
```

즉 Cloud Agent 효율화는 입력 Context만 줄이는 문제가 아니다.

Tool Output과 작업 결과도 같은 방식으로 관리해야 한다.

---

## 독자가 얻는 것

- 대형 Build/Test 로그를 LLM에 직접 전달하지 않는 구조를 설계할 수 있다.
- Result Filter와 Result Gateway의 역할 차이를 설명할 수 있다.
- 원본 로그와 Artifact를 폐기하지 않고 조회 가능한 형태로 보존할 수 있다.
- Agent가 처음 읽을 `result.json` 또는 동등한 Summary 형식을 정의할 수 있다.
- Failed Test, Exception, Stack Trace, Log를 단계적으로 조회하게 만들 수 있다.
- 동일 실패가 반복되는지 Failure Fingerprint로 판단할 수 있다.
- Retry와 Token/Cost Budget을 실패 결과와 연결할 수 있다.
- Commit, Diff, Test Result, Screenshot, Video, PR을 Evidence로 반환하게 만들 수 있다.
- UI 변경에서 `Demos over Diffs`를 빠른 1차 검증 방식으로 사용할 수 있다.

---

# 절 구성

## 8.1 Tool Output도 Context다

Cloud Agent의 Token 사용은 Source Code와 문서를 읽을 때만 발생하지 않는다.

다음 출력도 Agent가 읽으면 Context가 된다.

- Gradle build log
- JUnit output
- Spring Boot startup log
- Testcontainers log
- Docker build output
- Playwright trace
- Browser console log
- Static analysis report
- Git diff

예를 들어 전체 테스트가 다음 결과를 만들 수 있다.

```text
8,214 tests
100MB build/test log
JUnit XML
Coverage XML
Docker logs
Screenshots
Browser video
```

좋지 않은 방식:

```text
Cloud Runner
→ 100MB log
→ 전체를 Agent에게 전달
→ Agent가 실패 3건 탐색
```

권장 방식:

```text
Cloud Runner
→ 100MB log + reports
→ Artifact Store
→ Result Gateway
→ 실패 3건 Summary
→ Agent
```

핵심은 `실행 결과의 크기`와 `LLM에 전달하는 결과의 크기`를 분리하는 것이다.

---

## 8.2 Result Filter는 결정론적 프로그램으로 시작한다

Result Filter의 기본 역할은 대형 결과에서 다음과 같은 기계적으로 추출 가능한 정보를 뽑는 것이다.

- exit code
- build status
- total/passed/failed test count
- failed test names
- exception type
- assertion message
- top stack frame
- error/warning count
- artifact path

예:

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

Result Filter 자체에 LLM을 기본 사용하지 않는다.

구현 후보:

- shell script
- Python/Java utility
- JUnit XML parser
- static analysis report parser
- CI post-processing script

핵심 원칙:

> 코드로 추출할 수 있는 결과를 다시 LLM에게 읽혀서 찾게 하지 않는다.

---

## 8.3 Result Gateway는 원본을 보존하고 조회하게 한다

Result Filter가 단순히 로그를 축약해서 새 텍스트를 만드는 수준이라면, Result Gateway는 `원본 결과에 대한 조회 인터페이스`까지 제공한다.

권장 구조:

```text
Raw Artifact
├─ build.log
├─ junit.xml
├─ coverage.xml
├─ docker.log
├─ git.diff
├─ screenshots/
├─ browser-video/
└─ browser-trace/
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
Agent / Developer
```

핵심 원칙:

> 큰 결과를 요약해서 버리는 것이 아니라, 큰 결과를 저장하고 필요한 부분만 조회한다.

이 구조를 사용하면 처음에는 몇 KB의 Summary만 전달하고 추가 정보가 필요한 경우에만 Context를 확장할 수 있다.

---

## 8.4 Agent가 처음 읽는 결과는 작아야 한다

Agent에게 처음 전달하는 정보는 문제를 다음 단계로 좁힐 수 있는 수준이면 충분하다.

예:

```text
Task: auth-expired-token
Git SHA: abc123

BUILD: PASS

Tests:
total: 8214
passed: 8211
failed: 3

Failures:
- AuthServiceTest.expiredToken
- UserServiceTest.deleteUser
- UserMapperTest.insert

Artifacts:
result.json
junit.xml
build.log
```

Agent가 `AuthServiceTest.expiredToken`만 담당한다면 다른 두 실패의 전체 로그까지 처음부터 읽을 필요가 없다.

7장의 Task Scope와 연결하면 다음처럼 좁힐 수 있다.

```text
Task Scope
AuthService expired token
       +
Result Summary
AuthServiceTest.expiredToken FAIL
       ↓
Agent
```

---

## 8.5 필요한 정보만 단계적으로 조회한다

결과 조회도 Progressive Context와 같은 방식으로 설계한다.

```text
Summary
  ↓
Failure Detail
  ↓
Stack Trace
  ↓
Specific Test Log
  ↓
Related Artifact
  ↓
Full Raw Log
```

개념적 조회 인터페이스 예:

```text
get_failure("AuthServiceTest.expiredToken")

get_stacktrace("JWTExpiredException")

get_log(
  test="AuthServiceTest.expiredToken",
  lines=100
)

get_artifact("junit.xml")
```

이 인터페이스를 실제 HTTP API로 반드시 구현해야 한다는 의미는 아니다.

다음 형태도 가능하다.

- CLI
- CI artifact link
- 파일 index
- JSON report
- object storage path

중요한 것은 `전체 결과를 전달할 것인가 / 아무것도 전달하지 않을 것인가`의 이분법에서 벗어나는 것이다.

---

## 8.6 Artifact First

Cloud Runner는 PASS/FAIL 한 줄만 반환하지 않는다.

각 Task마다 최소한의 실행 Evidence를 남긴다.

예:

```text
task-142/
├─ result.json
├─ junit.xml
├─ coverage.xml
├─ build.log
├─ git.diff
├─ container-info.json
├─ screenshots/
├─ video/
└─ traces/
```

Agent가 기본적으로 읽는 파일은 `result.json`이다.

나머지는 필요할 때 조회한다.

사람도 Repository를 다시 checkout하거나 로컬에서 오류를 재현하기 전에 Artifact를 먼저 확인할 수 있다.

`Artifact First`의 목적은 Artifact를 많이 만드는 것이 아니다.

다음 질문에 답할 수 있는 결과를 남기는 것이다.

- 무엇을 실행했는가?
- 어느 Git 상태에서 실행했는가?
- 성공했는가?
- 무엇이 실패했는가?
- 실패를 다시 분석할 원본이 남아 있는가?

---

## 8.7 result.json을 Task의 첫 번째 결과 인터페이스로 사용한다

책의 예제에서는 다음과 같은 개념 스키마를 사용할 수 있다.

```json
{
  "taskId": "task-142",
  "gitSha": "abc123",
  "status": "FAIL",
  "build": "PASS",
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
  "artifacts": {
    "junit": "junit.xml",
    "log": "build.log"
  }
}
```

이 장에서는 JSON Schema 자체를 표준으로 확정하지 않는다.

핵심은 사람이 읽는 긴 로그와 별개로 Agent와 자동화가 먼저 읽을 수 있는 구조화된 결과가 존재해야 한다는 점이다.

---

## 8.8 자연어 완료 선언 대신 Evidence를 요구한다

좋지 않은 결과:

```text
구현을 완료했습니다.
테스트도 문제없습니다.
```

권장 결과:

```text
Implementation: DONE

Commit:
abc123

Build:
PASS

Unit Tests:
314 / 314 PASS

Integration Tests:
42 / 42 PASS

E2E:
PASS

Artifacts:
- junit.xml
- screenshot-after.png
- e2e-video.webm

PR:
#142
```

핵심 원칙:

> Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

Evidence 후보:

- Commit SHA
- Changed Files
- Diff
- Build Result
- Unit Test Result
- Integration Test Result
- E2E Result
- Coverage
- Screenshot
- Browser Video
- Trace
- Log Reference
- Docker Image Digest
- PR

Task마다 모든 Evidence가 필요한 것은 아니다.

7장의 `Output / Evidence` 필드에서 필요한 항목만 미리 정의한다.

---

## 8.9 Evidence는 Agent의 설명과 실행 사실을 분리한다

Cloud Agent의 자연어 설명은 유용하지만 검증 결과와 같은 것으로 취급하지 않는다.

예:

```text
Agent Summary:
"expired token 처리 로직을 수정했습니다."

Evidence:
AuthServiceTest.expiredToken PASS
AuthServiceTest 전체 PASS
Commit abc123
```

사람은 Summary를 빠르게 읽고 Evidence를 통해 사실을 확인할 수 있다.

이 구조는 Review 시간을 줄이면서도 Agent 설명을 그대로 신뢰해야 하는 상황을 줄인다.

---

## 8.10 Failure Fingerprint로 같은 실패 반복을 감지한다

Retry할 때 매번 전체 로그를 비교하지 않는다.

실패에서 안정적으로 추출할 수 있는 값을 조합해 Failure Fingerprint를 만든다.

후보:

- failing test id
- exception type
- assertion message
- error code
- top stack frame

예:

```text
failure fingerprint

AuthServiceTest.expiredToken
JWTExpiredException
expected=401
actual=200
AuthServiceTest.java:94
```

Retry #1과 Retry #2의 fingerprint가 같다면 수정이 실패를 바꾸지 못한 것이다.

```text
Retry #1
Fingerprint A

Retry #2
Fingerprint A

→ 동일 실패 반복
→ 계속 Retry하지 않음
```

모든 문자열을 byte-for-byte 비교할 필요는 없다.

Timestamp, random port, container id처럼 매 실행 달라지는 값은 fingerprint에서 제외해야 한다.

---

## 8.11 Retry는 횟수뿐 아니라 실패 변화 여부를 본다

좋지 않은 지시:

```text
성공할 때까지 계속 수정해.
```

권장 흐름:

```text
Agent Fix
   ↓
Runner
   ↓
FAIL
   ↓
Failure Fingerprint
   ↓
+-----------------------+
|                       |
새로운 실패          동일 실패 반복
|                       |
Retry 후보              중단
                        ↓
                 Human Escalation
```

예:

```text
Retry #1
Compilation Error

Retry #2
Unit Test Failure

Retry #3
Same Unit Test Failure
```

실패가 `Compilation → Unit Test`로 바뀌었다면 작업이 진전되었다고 볼 수 있다.

같은 fingerprint가 반복된다면 Token을 더 쓰기 전에 중단하는 것이 낫다.

Retry 정책 자체의 전체 설계는 이 책의 Agent Platform 일반론으로 확장하지 않고 Cloud Task 비용 제어 범위에서만 다룬다.

---

## 8.12 Token / Retry / Cost Budget을 결과와 연결한다

Cloud Agent의 자율성에는 한도를 둔다.

Task Budget 후보:

```text
max_turns
max_retry
max_tokens
max_wall_clock
max_cost
```

예:

```text
Simple Bug Fix
max_retry: 2
max_turns: 8
max_tokens: 30000
```

흐름:

```text
Agent
  ↓
Fix
  ↓
Runner
  ↓
PASS → Done
  |
 FAIL
  ↓
Fingerprint + Budget Check
  |
  +-- remaining → Retry
  |
  +-- exhausted → Human Escalation
```

핵심 원칙:

> Agent의 자율성은 무제한 실행 권한이 아니라 예산 안에서 스스로 해결할 수 있는 권한이다.

예시 숫자는 정책 기준이 아니라 설명용이다. 실제 한도는 프로젝트와 모델/서비스 비용 구조를 측정한 뒤 정한다.

---

## 8.13 Demos over Diffs

UI 변경에서는 Diff부터 읽는 것보다 실행 결과를 먼저 확인하는 편이 빠른 경우가 있다.

기존 Review:

```text
14개 파일 변경
→ Diff 전체 확인
→ Local checkout
→ Build
→ Browser 실행
→ UI 확인
```

Cloud 결과 활용:

```text
Build PASS
Test PASS
Before Screenshot
After Screenshot
E2E Video
       ↓
1차 동작 확인
       ↓
필요한 Diff 검토
```

핵심 문장:

> Cloud Agent의 결과는 Diff보다 Demo가 먼저일 수 있다.

이는 코드 Review를 생략한다는 의미가 아니다.

Screenshot/Video/E2E는 `기능이 실제로 실행되는가`를 빠르게 확인하는 Evidence이고, 코드의 품질과 안전성은 별도 Diff Review가 필요하다.

---

## 8.14 Java/Spring Boot에서 남길 Evidence

`campus-platform` Backend Task 예:

```text
Task
AuthService expired token 처리 수정
```

권장 Evidence:

```text
Commit
abc123

Changed Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Build
./gradlew compileJava
PASS

Target Test
./gradlew test --tests AuthServiceTest.expiredToken
PASS

Regression
./gradlew test --tests AuthServiceTest
PASS

Artifacts
- junit.xml
- build.log
```

Spring Integration Task라면 추가한다.

- Testcontainers startup result
- Integration JUnit report
- application log reference

Docker Task라면 추가한다.

- build status
- image tag/digest
- build log reference

Web E2E라면 추가한다.

- scenario result
- screenshot
- video/trace

---

## 8.15 campus-platform Result Gateway 예제

예제 구조:

```text
Cloud Runner
→ ./gradlew test
      ↓
Raw Artifact
├─ build.log
├─ junit.xml
└─ test-results/
      ↓
Result Gateway
├─ result.json
├─ failed-tests.index
└─ exception.index
      ↓
Cloud Agent
```

Agent가 받는 초기 결과:

```text
TASK: auth-expired-token
STATUS: FAIL

Test:
AuthServiceTest.expiredToken

Assertion:
expected 401
actual 200

Stack:
AuthServiceTest.java:94

Lookup:
- junit.xml
- build.log
```

Agent가 추가 정보가 필요하면 Auth 관련 로그만 조회한다.

수정 후 Runner가 다시 실행되고 최종 Evidence를 생성한다.

---

# 좋은 사례와 나쁜 사례

## 사례 A - 100MB 테스트 로그

좋지 않은 방식:

```text
100MB log
→ LLM Context
→ 실패 위치 탐색
```

권장 방식:

```text
100MB log
→ Artifact Store
→ 실패 index 생성
→ 실패 3건만 Agent 전달
```

## 사례 B - 요약 후 원본 폐기

좋지 않은 방식:

```text
Raw Log
→ Summary
→ Raw Log 삭제
```

Summary가 부족하면 다시 실행해야 한다.

권장 방식:

```text
Raw Log 보존
+
Summary / Index 제공
+
필요 시 Lookup
```

## 사례 C - Agent 완료 선언

좋지 않은 방식:

```text
Agent:
"수정했고 테스트했습니다."
```

권장 방식:

```text
Commit abc123
Target Test PASS
Regression PASS
Artifacts available
```

## 사례 D - 무한 Retry

좋지 않은 방식:

```text
FAIL
→ Agent
→ FAIL
→ Agent
→ FAIL
→ Agent
→ ...
```

권장 방식:

```text
FAIL
→ Fingerprint
→ Agent Fix
→ FAIL
→ Fingerprint 비교
→ Budget Check
→ Retry 또는 Escalation
```

---

# 필요한 그림

## 그림 1 - 작은 Input / 작은 Output

```text
Task Contract
Small Context
      ↓
Cloud Worker
      ↓
Raw Artifact
      ↓
Result Gateway
      ↓
Small Evidence
```

## 그림 2 - Result Gateway

```text
Raw Artifact
├─ log
├─ junit
├─ screenshot
├─ video
└─ diff
      ↓
Result Gateway
├─ summary
├─ failure index
├─ stack lookup
├─ log search
└─ artifact lookup
      ↓
Agent / Developer
```

## 그림 3 - Progressive Result Lookup

```text
Summary
→ Failure
→ Stack
→ Specific Log
→ Artifact
→ Full Raw Log
```

## 그림 4 - Retry / Budget

```text
Agent Fix
   ↓
Runner
   ↓
PASS ─────────────→ Done
   |
 FAIL
   ↓
Fingerprint
   ↓
Budget Check
   ├─ Retry
   └─ Human Escalation
```

## 그림 5 - Demos over Diffs

```text
Build/Test PASS
+ Screenshot/Video
       ↓
동작 빠른 확인
       ↓
관련 Diff Review
```

---

# 필요한 구현 예제

Phase 6에서 구현 후보:

```text
scripts/
├─ test.sh
├─ collect-test-result.py
└─ failure-fingerprint.py

artifacts/task-142/
├─ result.json
├─ junit.xml
├─ build.log
├─ git.diff
└─ screenshots/
```

구현 예제의 목적:

- JUnit 결과에서 실패 테스트 추출
- 긴 Gradle log를 원본으로 보존
- 구조화된 `result.json` 생성
- failure fingerprint 생성
- Task Contract의 Evidence 요구사항과 연결

실제 Result Gateway를 별도 대형 서비스로 구현하지 않는다.

책에서는 file/CLI 수준부터 시작해 개념을 검증한다.

---

# 필요한 공식 자료 조사

본문 작성 전 필요한 조사:

- GitHub Actions artifact와 test report 보존 방식
- GitHub Agentic Workflows의 deterministic pre-processing/token 효율 사례
- Claude Code Web 또는 유사 Cloud Agent의 artifact/PR 결과 처리 사례
- Playwright screenshot/video/trace 기능
- Gradle/JUnit XML test result 구조

제품별 현재 UI, artifact retention 기간, 요금은 변경 가능한 정보이므로 research 문서에서 기준일과 출처를 관리한다.

---

# 앞 장과 뒤 장 연결

## 7장과 연결

7장:

```text
Cloud Agent가 읽는 Input을 줄인다.
```

8장:

```text
Cloud Agent와 사람이 읽는 Output을 줄인다.
단 원본 Evidence는 보존한다.
```

## 9장과 연결

8장까지는 Agent가 읽는 정보와 결과 비용을 줄였다.

9장에서는 Agent가 작업을 시작하기 전의 환경 준비 비용을 줄인다.

```text
Context Cost
→ 7장

Result/Output Cost
→ 8장

Environment Startup Cost
→ 9장
```

## 10장과 연결

8장의 Result Gateway는 10장의 Runner-first 구조에서 `FAIL → Agent` 전환 시 Agent에게 어떤 결과를 전달할지 정의한다.

---

# 이 장에서 의도적으로 다루지 않을 내용

- 범용 Observability Platform 설계
- Agent-native Log Platform
- 장기 Agent Memory
- Agent Replay Platform
- Security Telemetry Platform
- 제품별 CI Artifact UI 사용법
- 범용 Workflow Engine 구현

이 장의 Result Gateway는 오직 다음 목적에 집중한다.

> Cloud Agent가 읽어야 하는 Tool Output을 줄이고, 사람이 검증할 Evidence를 남긴다.

---

# 장의 결론 메시지

본문에서는 다음 메시지로 수렴한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

> 큰 결과를 요약해서 버리는 것이 아니라, 큰 결과를 저장하고 필요한 부분만 조회한다.

> Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

> Retry는 실패 횟수만 세지 말고 실패가 실제로 변하고 있는지 확인한다.

> Agent의 자율성은 무제한 실행 권한이 아니라 예산 안에서 스스로 해결할 수 있는 권한이다.
