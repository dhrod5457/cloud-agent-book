# 8장. Tool Output을 줄이고 Evidence를 남기기

7장에서는 Cloud Agent에게 전달하는 입력을 줄였다.

```text
작은 Task
작은 Context
명확한 Validation
```

하지만 입력만 줄여서는 충분하지 않다.

Cloud Runner가 Build, Test, E2E, Docker 작업을 실행하면 많은 결과가 나온다.

예를 들어 전체 테스트가 다음을 만들 수 있다.

```text
8,214 tests
100MB build/test log
JUnit XML
Coverage XML
Docker logs
Screenshots
Browser video
```

이 결과를 그대로 LLM Context에 넣으면 다시 비용이 커진다.

중요한 실패 몇 건이 수천 줄의 정상 로그에 묻힐 수도 있다.

따라서 8장의 기본 구조는 다음과 같다.

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

> 큰 결과를 요약해서 버리는 것이 아니라, 큰 결과를 저장하고 필요한 부분만 조회한다.

그리고 Cloud Agent의 완료 응답도 자연어 한 줄로 끝내지 않는다.

> Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

---

## 1. Tool Output도 Context다

Cloud Agent의 Context는 Source Code와 문서만으로 구성되지 않는다.

Agent가 읽는 다음 출력도 모두 Context가 된다.

```text
Gradle build log
JUnit output
Spring Boot startup log
Testcontainers log
Docker build output
Playwright trace
Browser console log
Static analysis report
Git diff
```

예를 들어 전체 테스트가 다음처럼 끝났다고 하자.

```text
Tests: 8,214
Passed: 8,211
Failed: 3
```

그러나 Raw Log는 100MB다.

좋지 않은 방식:

```text
Cloud Runner
→ 100MB log
→ Agent에게 전체 전달
→ Agent가 실패 3건 탐색
```

권장 방식:

```text
Cloud Runner
→ 100MB log + report
→ Artifact Store
→ Result Filter
→ 실패 3건 Summary
→ Agent
```

이 구조에서는 `실행 결과의 크기`와 `LLM에 전달하는 결과의 크기`를 분리한다.

3장에서 설명한 Compute와 Token 분리가 결과 처리까지 이어지는 셈이다.

---

## 2. Result Filter는 프로그램으로 시작한다

대형 로그에서 다음 값은 LLM이 읽지 않아도 추출할 수 있다.

```text
exit code
build status
total / passed / failed test count
failed test name
exception type
assertion message
top stack frame
error / warning count
artifact path
```

예:

```text
BUILD: FAIL

Tests
total: 8214
passed: 8211
failed: 3

Failures
1. AuthServiceTest.expiredToken
2. UserServiceTest.deleteUser
3. UserMapperTest.insert
```

이 정보는 shell script, Python/Java utility, JUnit XML parser, CI post-processing script로 만들 수 있다.

Result Filter 자체에 LLM을 기본 사용하지 않는다.

> 코드로 추출할 수 있는 결과를 다시 LLM에게 읽혀서 찾게 하지 않는다.

이 원칙은 10장의 Runner-first 구조와 동일하다.

---

## 3. Result Filter와 Result Gateway는 다르다

Result Filter가 로그에서 필요한 정보를 뽑는 프로그램이라면 Result Gateway는 한 단계 더 나아간다.

원본을 보존하고 필요한 부분을 다시 조회할 수 있게 한다.

예:

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

Agent는 처음부터 `build.log` 전체를 읽지 않는다.

Summary를 보고 필요한 경우에만 Failure Detail을 요청한다.

이 구조는 요약 과정에서 중요한 원본이 사라지는 문제를 피한다.

---

## 4. Agent가 처음 읽는 Result는 작아야 한다

Task가 끝났을 때 Agent가 처음 보는 정보는 다음 단계의 판단에 충분한 정도면 된다.

예:

```text
Task
AUTH-142

Git SHA
abc123

BUILD
PASS

Tests
total: 8214
passed: 8211
failed: 3

Failures
- AuthServiceTest.expiredToken
- UserServiceTest.deleteUser
- UserMapperTest.insert

Artifacts
- result.json
- junit.xml
- build.log
```

Agent가 맡은 Task가 `AuthServiceTest.expiredToken` 수정이라면 다른 두 실패의 전체 로그는 처음부터 읽을 필요가 없다.

7장의 Scope와 결합하면 더 작게 만들 수 있다.

```text
Task Scope
AuthService expired token
       +
Result Summary
AuthServiceTest.expiredToken FAIL
       ↓
Agent
```

이 구조에서 Agent는 자신이 처리해야 할 Failure부터 읽는다.

---

## 5. Result 조회도 Progressive Context를 사용한다

7장에서 Source와 문서를 단계적으로 읽었다.

Result도 같은 방법을 사용할 수 있다.

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

개념적으로 다음과 같은 조회가 가능하다.

```text
get_failure("AuthServiceTest.expiredToken")
```

그다음 Exception을 본다.

```text
get_stacktrace("JWTExpiredException")
```

필요하면 특정 테스트 로그 100줄만 읽는다.

```text
get_log(
  test="AuthServiceTest.expiredToken",
  lines=100
)
```

이 인터페이스를 반드시 HTTP API로 구현해야 하는 것은 아니다.

다음 형태도 가능하다.

```text
CLI
JSON report
CI artifact link
File index
Object storage path
```

중요한 것은 `전체 로그를 전달할 것인가 / 아무것도 전달하지 않을 것인가`의 이분법에서 벗어나는 것이다.

---

## 6. Artifact First

Cloud Runner가 다음 한 줄만 반환한다고 하자.

```text
FAIL
```

이 정보는 충분하지 않다.

반대로 전체 로그를 Agent에게 다 넣는 것도 과하다.

중간 구조가 필요하다.

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

Agent가 기본으로 읽는 파일은 `result.json`이다.

나머지는 필요할 때 조회한다.

사람도 Repository를 다시 checkout해서 오류를 재현하기 전에 Artifact를 먼저 확인할 수 있다.

Artifact First의 목적은 파일을 많이 남기는 것이 아니다.

다음 질문에 답할 수 있어야 한다.

```text
무엇을 실행했는가?
어느 Git 상태에서 실행했는가?
성공했는가?
무엇이 실패했는가?
다시 분석할 원본이 남아 있는가?
```

---

## 7. result.json은 첫 번째 결과 인터페이스가 될 수 있다

책의 예제에서는 다음 정도의 구조를 사용할 수 있다.

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

이 형식을 표준으로 강제하려는 것은 아니다.

프로젝트에 따라 YAML이나 다른 JSON 구조를 사용할 수도 있다.

중요한 것은 사람용 긴 로그와 별개로 Agent와 자동화가 처음 읽을 **구조화된 결과**가 있다는 점이다.

---

## 8. 자연어 완료 선언과 실행 사실을 분리한다

Agent가 다음처럼 말할 수 있다.

```text
Expired token 처리 로직을 수정했습니다.
테스트도 정상입니다.
```

이 설명은 읽기 쉽지만 검증 결과 자체는 아니다.

권장 결과는 다음처럼 분리한다.

```text
Agent Summary
expired token 처리 분기를 수정했습니다.

Evidence
Commit: abc123
AuthServiceTest.expiredToken: PASS
AuthServiceTest: 24 / 24 PASS
```

Cloud Agent의 자연어 Summary는 사람이 변경 의도를 빠르게 이해하는 데 사용한다.

실행 사실은 Evidence로 확인한다.

> 설명과 검증은 같은 것이 아니다.

---

## 9. Evidence는 Task별로 필요한 것만 요구한다

모든 Task가 모든 Artifact를 만들 필요는 없다.

Bug Fix라면 다음 정도면 충분할 수 있다.

```text
Commit SHA
Changed Files
Unit Test Result
```

UI Task라면 다음이 중요해질 수 있다.

```text
E2E Result
Before Screenshot
After Screenshot
Browser Video
```

Docker Build라면 다음을 사용할 수 있다.

```text
Build Result
Image Digest
Build Log Reference
```

Migration이라면 다음이 Evidence가 된다.

```text
Migration Apply Result
Upgrade Result
Rollback Result if applicable
DB Version
```

즉 7장의 `Output / Evidence`에서 필요한 결과를 Task별로 미리 정한다.

---

## 10. Demos over Diffs

UI 변경은 Diff만 보고 판단하기 어렵다.

예를 들어 Button 위치가 바뀌고 Form Layout이 수정됐다고 하자.

Git Diff를 보면 CSS와 JSX 변경은 확인할 수 있다.

하지만 실제 화면이 기대한 모습인지 판단하려면 결과를 보는 편이 빠르다.

권장 순서:

```text
Build PASS
E2E PASS
Before Screenshot
After Screenshot
Browser Video
      ↓
필요한 경우 Diff Review
```

이 방식을 `Demos over Diffs`라고 설명할 수 있다.

코드 Review를 생략한다는 뜻은 아니다.

먼저 동작 결과를 확인하고, 그다음 구현 세부를 검토한다는 뜻이다.

UI뿐 아니라 CLI, API Response, Generated Report처럼 결과를 직접 확인할 수 있는 작업에도 같은 원칙을 적용할 수 있다.

---

## 11. Failure Fingerprint로 같은 실패 반복을 구분한다

Agent가 수정한 뒤 Runner가 다시 실패했다고 하자.

Retry할 때마다 전체 로그를 비교할 필요는 없다.

실패에서 안정적인 값을 추출해 Fingerprint를 만들 수 있다.

예:

```text
AuthServiceTest.expiredToken
JWTExpiredException
expected=401
actual=200
AuthServiceTest.java:94
```

Fingerprint 구성 후보:

```text
failing test id
exception type
assertion message
error code
top stack frame
```

Timestamp, random port, container id처럼 매 실행 달라지는 값은 제외한다.

Retry #1과 #2가 같은 Fingerprint라면 수정이 실패를 바꾸지 못한 것이다.

```text
Retry #1
Fingerprint A

Retry #2
Fingerprint A

→ 같은 실패 반복
```

이 경우 Agent에게 같은 작업을 무한히 반복시키는 것보다 중단하는 편이 낫다.

---

## 12. Retry는 횟수와 실패 변화 여부를 같이 본다

다음 지시는 위험하다.

```text
성공할 때까지 계속 수정해.
```

Cloud Agent의 Retry에는 Compute, Token, Review 비용이 계속 들어간다.

대신 실패가 어떻게 변했는지 본다.

```text
Retry #1
Compilation Error

Retry #2
Unit Test Failure

Retry #3
Same Unit Test Failure
```

첫 번째와 두 번째는 실패 종류가 바뀌었다.

Compilation은 해결됐고 Unit Test까지 진행했다는 의미일 수 있다.

하지만 세 번째에서 같은 Unit Test Failure가 반복된다면 중단 후보가 된다.

기본 흐름:

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
새 실패 / 변화        동일 실패 반복
|                       |
Retry 후보              중단 / Escalation
```

---

## 13. Budget을 Result 처리와 연결한다

7장의 Task Contract에서 Budget을 선택 필드로 두었다.

예:

```text
max_retry: 2
max_turns: 8
```

이 값을 단순 카운터로만 사용하지 않는다.

Result Gateway와 연결하면 다음처럼 판단할 수 있다.

```text
Retry 1
Fingerprint A

Retry 2
Fingerprint B
→ 실패 변화 있음
→ 한 번 더 시도할 가치 검토
```

반대로:

```text
Retry 1
Fingerprint A

Retry 2
Fingerprint A
→ 동일 실패
→ Budget 남아도 중단 후보
```

제품별 Token이나 비용 수치는 다르므로 절대값을 표준으로 제시하지 않는다.

중요한 것은 Agent 자율 실행에 명시적인 종료 조건이 있다는 점이다.

---

## 14. PASS도 Evidence를 남긴다

실패만 Artifact를 남기고 PASS는 버리는 경우가 있다.

하지만 나중에 다음 질문이 생길 수 있다.

```text
이 Commit에서 실제로 어떤 테스트를 실행했는가?
몇 개가 통과했는가?
어느 환경에서 실행했는가?
```

따라서 PASS에서도 최소 Evidence를 남기는 편이 좋다.

예:

```text
Git SHA: abc123
Build: PASS
Unit: 314 / 314 PASS
Integration: 42 / 42 PASS
E2E: PASS
Duration: 18m 42s
```

Raw Log를 영구 보존할지 여부는 프로젝트 정책에 따라 다를 수 있다.

하지만 Summary와 핵심 Artifact Reference는 남길 수 있다.

---

## 15. campus-platform 실패 결과 예제

`campus-platform`의 `AUTH-142` Task가 다음 결과를 만들었다고 하자.

```text
Task
AUTH-142

Base SHA
4f29abc

Result Commit
9c10def

Build
PASS

Auth Tests
24 total
23 passed
1 failed

Failure
AuthServiceTest.expiredToken
expected: 401
actual: 200

Artifacts
- result.json
- auth-test.xml
- auth-test.log
```

Agent에게 처음 전달하는 Context는 이 정도다.

Agent가 자세한 Exception이 필요하면 다음 단계로 조회한다.

```text
Failure Detail
→ stack trace
→ specific test log
```

수정 후 Runner가 재실행한다.

```text
Result Commit
bb8e312

Auth Tests
24 / 24 PASS
```

최종 Evidence:

```text
Implementation: DONE
Commit: bb8e312
Changed Files: 2
Auth Tests: 24 / 24 PASS
Artifact: auth-test.xml
```

---

## 16. 작은 Input과 작은 Output이 하나의 구조가 된다

7장과 8장을 합치면 Cloud Agent의 Context 흐름이 다음처럼 바뀐다.

기존 방식:

```text
Repository 전체
+ 문서 전체
+ 로그 전체
+ Diff 전체
→ Agent
```

개선된 방식:

```text
Task Contract
→ Relevant Files
→ 필요한 문서
      ↓
Cloud Work
      ↓
Result Summary
→ Failure Detail
→ 필요한 Artifact
```

즉 입력과 출력 양쪽 모두에서 Progressive Context를 사용한다.

```text
Small Input
   ↓
Cloud Compute
   ↓
Small Output
```

이 구조를 사용하면 Cloud CPU에는 많은 작업을 시키면서 LLM에는 필요한 정보만 보여줄 수 있다.

---

## 이 장에서 기억할 것

Cloud 결과를 설계할 때 다음을 확인한다.

```text
Raw Result를 보존하는가?
Agent에게 처음 전달하는 Summary는 작은가?
Failed Test를 구조화해 찾을 수 있는가?
필요한 Log만 다시 조회할 수 있는가?
자연어 설명과 Evidence를 구분하는가?
같은 Failure 반복을 감지할 수 있는가?
Retry 종료 조건이 있는가?
Task별 필요한 Artifact가 정해져 있는가?
```

핵심은 세 문장으로 정리할 수 있다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

> 큰 결과를 요약해서 버리는 것이 아니라, 큰 결과를 저장하고 필요한 부분만 조회한다.

> Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

다음 장에서는 Cloud Worker가 Task를 받았을 때 환경 설치부터 시작하지 않도록 Prepared Environment, Cache, Snapshot을 설계한다.
