# 16장. 하나의 기능을 Local + Cloud로 끝까지 개발하기

15장에서는 `campus-platform`의 전체 운영 모델을 정리했다. 이번 장에서는 그 구조를 하나의 기능에 적용해 요구사항부터 병합까지 시간 순서대로 따라간다.

예제는 다음과 같다.

```text
학생 출결 API 인증 변경
```

요구사항은 단순화한다.

```text
만료된 인증 토큰으로 출결 API를 호출하면 HTTP 401을 반환한다.
정상 토큰의 기존 출결 등록 동작은 유지한다.
관리자 Web에서는 인증 실패 상태를 확인할 수 있어야 한다.
```

이 기능에는 백엔드 수정, 단위 / 통합 테스트, Web E2E, Docker 빌드, 내부 환경 검증이 필요하다고 가정한다.

이 장의 목적은 앞 장의 개념을 다시 설명하는 것이 아니다. 실제 기능 하나에서 **로컬과 클라우드가 언제 바뀌고, 무엇을 다음 단계의 입력과 검증 근거로 사용하는지** 보여주는 것이다.

> 클라우드 에이전트 활용은 별도 도구 사용법이 아니라 개발 작업 흐름 설계다.

---

## 1. Local에서 요구사항과 영향 범위를 고정한다

처음부터 기능 전체를 클라우드 에이전트에게 맡기지 않는다.

로컬에서 먼저 다음을 결정한다.

```text
정상 동작
실패 동작
영향 모듈
변경 금지 영역
내부 시스템 의존성
Cloud에서 검증 가능한 범위
```

이번 예제의 초기 판단은 다음과 같다.

```text
Architecture / 영향 분석
→ Local

Expired token 처리 수정
→ Cloud Agent 후보

Unit / Integration / E2E / Docker
→ Cloud Runner

Tibero / Internal API 최종 확인
→ Local
```

예상 관련 파일도 좁힌다.

```text
AuthService.java
JwtTokenProvider.java
AttendanceController.java
AuthServiceTest.java
AttendanceApiTest.java
```

그리고 이번 변경에서 건드리지 않을 영역을 정한다.

```text
DB Schema 변경 없음
HSM 변경 없음
OAuth 전체 구조 변경 없음
공통 Exception Format 변경 없음
```

이 판단을 바탕으로 다음 단계에서 무엇을 바꾸고 어디까지 검토할지 정한다.

---

## 2. 전달 가능한 Git 기준점을 만든다

클라우드 작업자가 로컬 작업 디렉터리의 미커밋 상태를 자동으로 아는 것을 기대하지 않는다.

필요한 로컬 작업을 정리한 뒤 기준 커밋을 만든다.

```text
Local
→ 빠른 검증
→ Commit
→ Push
```

예:

```text
Base SHA
abc123
```

이후 클라우드 작업과 검증 결과는 이 Git 상태에서 출발한다.

```text
Base SHA
abc123
   ↓
Result SHA
   ↓
Verification
   ↓
PR
```

11장과 13장에서 설명한 Git 작업 전달을 실제 기능의 시작점으로 사용하는 것이다.

---

## 3. 기능을 Agent Task와 Runner Task로 나눈다

하나의 기능을 한 에이전트에게 통째로 맡기지 않는다.

이번 기능은 다음처럼 나눌 수 있다.

```text
Task A
Expired token backend fix
→ Cloud Agent

Task B
Backend Unit / Integration Validation
→ Cloud Runner

Task C
Admin Web E2E Validation
→ Cloud Runner

Task D
Docker Build Validation
→ Cloud Runner
```

작업 A에는 판단과 코드 수정이 필요하다. B~D는 정해진 명령과 판정 기준이 있으므로 실행기가 우선한다.

작업 A의 입력은 7장의 작업 명세 형식을 그대로 사용한다.

```text
Task
Expired token 처리 수정

Goal
Attendance API 인증 실패 → HTTP 401

Base SHA
abc123

Scope
auth + attendance auth boundary

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Do Not Change
- DB Schema
- OAuth 전체 구조
- Common Exception Format

Validation
./gradlew test --tests AuthServiceTest

Expected Result
expired token → 401
normal token → 기존 테스트 PASS

Output / Evidence
- Result SHA
- Changed Files
- Validation Result
```

여기서 중요한 것은 형식 자체가 아니라 클라우드 작업자가 **어디서 시작하고, 어디까지 바꾸며, 무엇을 통과해야 하는지**가 명시되어 있다는 점이다.

---

## 4. Prepared Environment에서 작은 Context로 시작한다

작업 A는 `backend-test` 같은 준비된 실행환경에서 시작한다.

```text
Prepared Environment
+ Fresh Source / Branch
        ↓
Cloud Task
```

에이전트에게 처음 제공하는 정보도 작게 유지한다.

```text
Task Contract
Relevant Files
Known Failure
```

필요할 때만 맥락 정보를 넓힌다.

```text
Relevant Files
→ Direct Dependency
→ Related Document
→ Wider Module Context
```

저장소 전체를 처음부터 읽는 것이 기본 경로가 아니다.

> 클라우드에서 막혔을 때 맥락 정보를 한 번에 넓히지 않고 필요한 정보만 추가한다.

---

## 5. 수정 뒤에는 Target Test로 가장 먼저 검증한다

에이전트가 코드를 수정했다고 하자.

첫 번째 검증은 좁게 시작한다.

```bash
./gradlew test --tests AuthServiceTest
```

PASS라면 회귀 검증 단계로 이동한다.

FAIL이면 8장의 Result Gateway를 통해 작은 실패 요약부터 확인한다.

```text
STATUS: FAIL
Test: AuthServiceTest.expiredToken
Expected: 401
Actual: 200
Top Frame: AuthServiceTest.java:94
```

필요한 경우에만 상세 결과를 추가한다.

```text
Failure Detail
→ Stack Trace
→ Specific Test Log
→ Raw Artifact
```

에이전트가 코드를 수정했다는 사실만으로 성공했다고 볼 수는 없다. 클라우드 실행기에서 같은 검증을 다시 실행해 통과해야 다음 단계로 넘어간다.

---

## 6. 같은 실패가 반복되면 무한 Retry하지 않는다

에이전트가 수정한 뒤에도 같은 실패가 반복될 수 있다.

```text
Retry #1
AuthServiceTest.expiredToken
expected 401 / actual 200

Retry #2
AuthServiceTest.expiredToken
expected 401 / actual 200
```

위 재시도 횟수는 설명용 예다.

실패 식별 정보가 그대로이고 허용 한도도 소진되고 있다면 맥락 정보와 프롬프트만 계속 늘리지 않는다.

```text
Same Failure
+ Retry Budget 소진
        ↓
Stop / Escalate
```

반대로 실패가 `Compilation Error → Target Test Failure`처럼 바뀌었다면 작업이 진행된 것일 수 있다.

재시도 횟수와 실패 변화 여부를 같이 본다.

구체적인 로컬 복귀 판단은 17장에서 정리한다.

---

## 7. Result SHA에서 Regression을 병렬 실행한다

에이전트가 다음 Result SHA를 만들었다고 하자.

```text
Result SHA
def456
```

이제 같은 SHA에서 독립 검증을 병렬로 실행한다.

```text
Result SHA def456
      ↓
+---------+-------------+---------+---------+
|         |             |         |         |
Unit   Integration    Docker     E2E
|         |             |         |         |
+---------+-------------+---------+---------+
          ↓
        Fan-in
```

예:

```text
Unit        PASS
Integration PASS
Docker      PASS
E2E         FAIL
```

이 단계에서 중요한 조건은 모든 검증 근거가 같은 소스 코드 상태를 검증한다는 점이다.

UI/E2E 실패라면 화면 캡처, 영상, 실행 추적 기록 같은 결과부터 본다.

```text
Scenario
attendance expired token error view

Expected
401 error state visible

Actual
generic error page

Artifact
screenshot-fail.png
```

필요한 경우 에이전트가 수정하고, 새 Result SHA에서 다시 클라우드 실행기가 검증한다.

---

## 8. Cloud 단계가 끝나면 Evidence와 PR을 반환한다

클라우드 검증이 끝났다면 로컬로 자연어 완료 보고가 아니라 검증 가능한 결과를 돌려준다.

설명용 예:

```text
Task: AUTH-142
Result SHA: def789

Build: PASS
Unit: PASS
Integration: PASS
Docker: PASS
E2E: PASS

Artifacts:
- junit.xml
- integration-junit.xml
- screenshot-after.png
- e2e-trace.zip

PR: #142
```

실제 테스트 수나 실행시간은 프로젝트 측정값을 사용한다. 여기서는 결과 구조만 보여준다.

로컬 검토에서는 다음 순서로 확인할 수 있다.

```text
Evidence
→ Changed Files
→ Diff
→ Architecture 영향
```

UI 변경이라면 화면 캡처 / 영상을 먼저 확인한 뒤 코드 변경 내역을 본다.

---

## 9. 내부망 검증을 Local에서 이어간다

클라우드에서 모든 검증을 끝낼 필요는 없다.

이번 기능에서 최종 대상이 내부 Tibero와 내부 API라고 가정한다.

```text
Cloud
→ 일반적으로 재현 가능한 Build / Test / E2E
      ↓
Local / Internal
→ Tibero
→ Internal API
→ Jenkins Deployment Validation
```

HSM 변경이 없다면 HSM 검증까지 추가하지 않는다. 작업과 관련된 내부 경계만 확인한다.

클라우드 검증 근거와 내부 검증도 같은 PR SHA를 기준으로 맞춘다.

```text
PR SHA
→ Cloud Required Checks
→ Internal Validation
→ Review
```

검증 사이에 코드가 바뀌었다면 필요한 검증을 다시 실행한다.

---

## 10. Merge와 Task 종료

병합 전에는 최종 기준 SHA에서 필요한 검증이 모두 연결되어 있는지 확인한다.

```text
PR SHA
→ Required Checks
→ Internal Validation
→ Review Complete
→ Merge
```

작업 종료 후에는 모든 에이전트 대화나 원본 로그를 영구 보관할 필요는 없다.

최소 작업 이력은 다음 정도면 충분할 수 있다.

```text
Task ID
PR
Merge Commit
Final Evidence
주요 Failure / Retry 정보
```

이 기록의 목적은 에이전트의 장기 기억을 만드는 것이 아니라 나중에 `무엇을 변경했고 무엇으로 검증했는가`를 추적하는 것이다.

---

## 11. 시간 흐름으로 보면

설명용 시간표를 만들어보자.

```text
09:30 Requirement / Impact Analysis
10:00 Base Commit
10:05 Cloud Task 시작
10:06 Developer 다음 작업 시작
10:22 Agent Fix Commit
10:23 Parallel Validation
10:40 E2E Failure
10:45 Fix
10:58 All Required Cloud Checks PASS
11:20 Developer Review
11:35 Internal Validation
11:45 Merge
```

이 시간은 실제 성능 수치가 아니라 작업 흐름을 설명하기 위한 예시다.

핵심은 다음 관계다.

```text
Cloud Elapsed Time
!=
Developer Blocking Time
```

개발자가 클라우드 실행 전체 시간 동안 기다리지 않았다는 점이 중요하다.

비용도 분리해서 본다.

```text
Compute
→ Unit / Integration / Docker / E2E

LLM
→ Failure 분석 / 코드 수정

Human
→ Requirement / Review / Internal Validation / Merge 판단
```

어느 구간이 병목인지 구분해야 다음 개선 지점을 찾을 수 있다.

---

## 12. 실패 분기는 실행 위치를 다시 판단하는 신호다

정상 작업 흐름이 항상 끝까지 클라우드에서 진행되는 것은 아니다.

### Cloud에서 재현되지 않음

```text
Cloud Runner
→ cannot reproduce
→ Evidence 반환
→ Local 조사
```

### 내부 DB 의존성 발견

```text
Cloud Analysis
→ Tibero-specific behavior 발견
→ Local / Internal Validation으로 이동
```

### Scope가 예상보다 크게 확대됨

```text
3개 파일 예상
→ 여러 Module / DB Schema 영향 발견
→ Cloud Task 중단
→ Local에서 재분해
```

위 파일 수는 설명용 예다.

이 분기에서 클라우드 작업은 실패한 것이 아니다. 실행 중 발견된 조건에 따라 실행 위치 결정을 다시 한 것이다.

17장에서는 이런 중단과 로컬 복귀 기준을 체계적으로 정리한다.

---

## 13. 하나의 기능을 끝까지 연결하면

이번 기능의 전체 흐름은 다음과 같다.

```text
Local
Requirement / Impact Analysis
        ↓
Base Commit
        ↓
Task Split / Task Contract
        ↓
Git Handoff
        ↓
Cloud Agent
Analyze / Fix
        ↓
Target Runner
        ↓
Parallel Regression
        ↓
Evidence / PR
        ↓
Local Review
        ↓
Internal Validation
        ↓
Merge
```

이 흐름에서 클라우드 에이전트는 전체 개발을 대신하지 않는다.

```text
Local
→ 결정과 통합

Cloud Runner
→ 실행과 검증

Cloud Agent
→ 필요한 판단과 수정
```

> 작은 작업을 Git으로 넘기고, 실행기와 에이전트가 만든 검증 근거를 다시 로컬로 가져온다.

이것이 이 책에서 사용하는 로컬과 클라우드를 함께 쓰는 작업 흐름의 실제 실행 형태다.