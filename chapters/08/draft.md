# 8장. Tool Output을 줄이고 Evidence를 남기기

7장에서는 클라우드 에이전트에게 전달하는 입력을 줄였다.

```text
Task Contract
→ Small Input / Context
```

하지만 입력만 작게 만든다고 맥락 정보 비용이 줄어드는 것은 아니다.

빌드, 테스트, E2E, Docker 작업은 많은 로그와 결과물을 만든다. 이 결과를 그대로 LLM에 전달하면 몇 건의 실패를 찾기 위해 대량의 정상 로그까지 읽게 된다.

8장의 기본 구조는 다음과 같다.

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
필요할 때만 상세 Artifact 조회
```

핵심은 결과를 버리는 것이 아니다.

> 큰 결과는 보존하고, 에이전트에게는 판단에 필요한 부분만 보여준다.

그리고 작업 완료는 자연어 선언이 아니라 검증 근거로 확인한다.

> 클라우드 에이전트에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

---

## 1. Tool Output도 Context다

에이전트가 읽는 다음 결과는 모두 맥락 정보가 된다.

```text
Gradle Build Log
JUnit Output
Spring Boot Startup Log
Testcontainers Log
Docker Build Output
Playwright Trace
Browser Console Log
Static Analysis Report
Git Diff
```

설명용 예로 전체 테스트가 다음 결과를 만들었다고 하자.

```text
Tests: 8,214
Passed: 8,211
Failed: 3
Raw Log: 100MB
```

좋지 않은 흐름:

```text
Cloud Runner
→ 100MB Log
→ Agent에게 전체 전달
→ Agent가 실패 3건 탐색
```

권장 흐름:

```text
Cloud Runner
→ Raw Log + Report 저장
→ 실패 3건 추출
→ Agent는 Failure Summary부터 확인
```

실행 결과의 크기와 LLM에 전달하는 결과의 크기를 분리한다.

---

## 2. Result Filter는 결정론적 프로그램으로 시작한다

대형 로그에서 다음 값은 LLM이 없어도 추출할 수 있다.

```text
Exit Code
Build Status
Total / Passed / Failed Count
Failed Test Name
Exception Type
Assertion Message
Top Stack Frame
Error / Warning Count
Artifact Path
```

예:

```text
BUILD: FAIL

Tests
- total: 8,214
- passed: 8,211
- failed: 3

Failures
1. AuthServiceTest.expiredToken
2. UserServiceTest.deleteUser
3. UserMapperTest.insert
```

이 정도 결과는 셸 스크립트, JUnit XML 분석 도구, CI 후처리 스크립트로 만들 수 있다.

> 코드로 추출할 수 있는 결과를 다시 LLM에게 읽혀서 찾게 하지 않는다.

이 원칙은 10장의 Runner-first 구조와 연결된다.

---

## 3. Result Gateway는 원본을 버리지 않는다

Result Filter는 결과에서 필요한 값을 추출하는 단계다. Result Gateway는 여기서 더 나아가 원본 결과물을 보존하고, 필요할 때 다시 찾아볼 수 있게 하는 창구다.

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
       ↓
Result Gateway
├─ Summary
├─ Failed Test Index
├─ Exception Index
├─ Log Search
└─ Artifact Lookup
       ↓
Agent / Developer
```

처음부터 `build.log` 전체를 읽지 않는다.

```text
Summary
  ↓
Failure Detail
  ↓
Specific Log
  ↓
Related Artifact
  ↓
Full Raw Log
```

7장의 단계적 정보 제공을 실행 결과에도 적용한 구조다.

이 인터페이스가 반드시 HTTP API일 필요는 없다.

```text
CLI
JSON Report
CI Artifact Link
File Index
Object Storage Path
```

핵심은 원본을 유지하면서 에이전트가 처음 읽는 결과를 작게 만드는 것이다.

---

## 4. result.json은 첫 번째 결과 인터페이스가 될 수 있다

작업 결과를 사람이 읽는 긴 로그와 별개로 구조화할 수 있다.

예:

```json
{
  "taskId": "AUTH-142",
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

이 JSON 형식을 표준으로 강제하려는 것은 아니다.

중요한 것은 에이전트와 자동화가 먼저 읽을 수 있는 작은 구조화 결과가 있다는 점이다.

```text
Task Scope
AUTH-142 / expired token
       +
Result Summary
AuthServiceTest.expiredToken FAIL
       ↓
Agent
```

현재 작업에 필요하지 않은 실패와 로그는 처음부터 읽지 않는다.

---

## 5. Artifact First

클라우드 실행기가 `FAIL` 한 줄만 반환하면 원인을 다시 재현해야 한다.

반대로 원본 로그 전체를 에이전트에게 보내는 것도 과하다.

작업 단위로 결과물을 보존한다.

```text
artifacts/AUTH-142/
├─ result.json
├─ junit.xml
├─ build.log
├─ git.diff
├─ screenshots/
├─ video/
└─ traces/
```

에이전트는 기본적으로 `result.json`부터 읽고 필요한 파일만 추가로 조회한다.

결과물을 먼저 보존하는 원칙의 목적은 파일을 많이 남기는 것이 아니다.

다음 질문에 답할 수 있어야 한다.

```text
무엇을 실행했는가?
어느 Git 상태에서 실행했는가?
성공했는가?
무엇이 실패했는가?
원본 결과가 남아 있는가?
```

---

## 6. 자연어 설명과 실행 사실을 분리한다

에이전트 요약은 변경 의도를 설명하는 데 유용하다.

```text
Agent Summary
expired token 처리 분기를 수정했습니다.
```

하지만 검증 결과와 같은 것으로 취급하지 않는다.

```text
Evidence
Result SHA: abc123
AuthServiceTest.expiredToken: PASS
AuthServiceTest: 24 / 24 PASS
```

작업별 검증 근거는 다르다.

오류 수정:

```text
Result SHA
Changed Files
Unit Test Result
```

UI 작업:

```text
E2E Result
Before / After Screenshot
Browser Video / Trace
```

Docker 빌드:

```text
Build Result
Image Digest
Build Log Reference
```

7장의 `Output / Evidence`에서 필요한 결과를 미리 정하고, 8장에서는 그 결과를 구조화한다.

---

## 7. UI 변경은 Demo Evidence를 먼저 볼 수 있다

UI 변경은 코드 변경 내역만 읽어서는 실제 결과를 빠르게 판단하기 어렵다.

예를 들어 화면 배치와 CSS가 바뀌었다면 다음 순서를 사용할 수 있다.

```text
Build PASS
E2E PASS
Before Screenshot
After Screenshot
Browser Video
      ↓
필요한 경우 Diff Review
```

이 책에서는 이를 `Demos over Diffs`, 즉 코드 변경 내역보다 동작 결과를 먼저 확인하는 관점으로 설명한다. 코드 검토를 생략한다는 뜻은 아니다. 동작 결과를 확인한 뒤 구현의 세부 내용을 검토하는 순서다.

명령줄 도구의 출력, API 응답, 생성된 보고서처럼 결과를 직접 확인할 수 있는 작업에도 같은 원칙을 적용할 수 있다.

---

## 8. Failure Fingerprint는 반복 실패를 구분한다

에이전트가 수정한 뒤 실행기가 다시 실패했다고 하자.

재시도마다 원본 로그 전체를 비교하지 않고 안정적인 실패 정보를 조합할 수 있다.

예:

```text
AuthServiceTest.expiredToken
JWTExpiredException
expected=401
actual=200
AuthServiceTest.java:94
```

실패 식별 정보 후보:

```text
Failing Test ID
Exception Type
Assertion Message
Error Code
Top Stack Frame
```

타임스탬프, 임의의 포트 번호, 컨테이너 ID처럼 실행마다 바뀌는 값은 제외한다.

```text
Retry #1 → Fingerprint A
Retry #2 → Fingerprint A
```

같은 실패 식별 정보가 반복되면 수정이 실패를 바꾸지 못한 것이다.

반대로 실패가 바뀌었다면 진행 중일 수 있다.

```text
Retry #1 → Compilation Error
Retry #2 → Unit Test Failure
```

이 장에서는 반복 실패를 식별하는 방법까지만 다룬다. 언제 중단하고 로컬로 되돌릴지는 17장에서 정리한다.

---

## 9. PASS에도 최소 Evidence를 남긴다

성공한 작업도 다음 정도의 결과는 남긴다.

```text
Task ID
Git SHA
Validation Command
Status
Duration
Artifact Reference if needed
```

예:

```text
Task: AUTH-142
Git SHA: def456
Validation: ./gradlew :auth:test
Status: PASS
Tests: 24 / 24
```

성공 결과까지 대형 로그를 LLM이 읽을 필요는 없지만 어떤 상태를 검증했는지는 추적할 수 있어야 한다.

---

## 10. AUTH-142 결과 흐름

7장에서 만든 AUTH-142 작업 명세를 그대로 사용한다.

```text
Task Contract
→ Cloud Agent Fix
→ Result SHA def456
       ↓
Cloud Runner
→ ./gradlew :auth:test
       ↓
Raw Result
├─ junit.xml
└─ build.log
       ↓
Result Filter
→ 24 / 24 PASS
       ↓
result.json
       ↓
Evidence
- Result SHA def456
- Test PASS
```

실패했다면 다음처럼 시작한다.

```text
result.json
→ AuthServiceTest.expiredToken FAIL
→ Failure Detail
→ 필요한 경우 Specific Log
```

입력과 출력이 대칭을 이룬다.

```text
7장
Task Contract
→ Small Input / Context
        ↓
Cloud Runner / Agent
        ↓
8장
Result Gateway / Evidence
→ Small Output / Tool Result
```

다음 장에서는 이 작업이 시작되기 전에 발생하는 환경 준비 비용을 줄인다.
