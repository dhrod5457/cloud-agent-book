# 6장. Cloud에 보내기 좋은 개발 작업

5장에서 Task를 `Local / Cloud / Hybrid / Runner-first` 중 어디에 배치할지 판단하는 기준을 만들었다.

이제 그 기준을 실제 개발 작업에 적용해 본다.

중요한 것은 작업 이름만 보고 Cloud Agent를 호출하지 않는 것이다.

예를 들어 `테스트 작업` 하나도 실제로는 다음 세 단계로 나뉠 수 있다.

```text
테스트 실행
→ Cloud Runner

실패 원인 분석
→ Cloud Agent

수정 후 재검증
→ Cloud Runner
```

따라서 이 장에서는 작업을 다음 기준으로 나눈다.

```text
Deterministic Work
→ Cloud Runner

Reasoning + Code Change
→ Cloud Agent

Internal / Interactive Work
→ Local 또는 Hybrid
```

> Cloud에 보내기 좋은 작업과 Cloud Agent가 직접 해야 하는 작업은 같은 개념이 아니다.

> Runner가 할 수 있으면 Runner에게 맡긴다.

> Agent는 판단과 수정이 필요한 구간에만 사용한다.

---

## 1. 작업 이름보다 실행 구조를 본다

같은 작업도 단계마다 실행 주체가 달라질 수 있다.

예를 들어 Dependency Update를 생각해 보자.

```text
Dependency Version 변경
      ↓
Cloud Runner
→ Build / Test
      ↓
+----+----+
|         |
PASS      FAIL
|         |
Done   Failure Summary
          ↓
      Cloud Agent
          ↓
   Compatibility Fix
          ↓
       Runner
```

처음부터 Agent가 Repository 전체를 분석할 필요는 없다.

먼저 일반 프로그램으로 검증하고, 실패가 생겼을 때만 판단을 요청한다.

이 구조를 다른 작업에도 반복 적용한다.

작업을 볼 때 다음 질문을 사용한다.

```text
실행 명령이 명확한가?
판정 기준이 명확한가?
코드 변경이 필요한가?
사람의 판단이 필요한가?
외부 시스템 의존성이 있는가?
어떤 Evidence를 반환해야 하는가?
```

---

## 2. Build는 기본적으로 Runner 작업이다

Java/Spring Boot 프로젝트의 Build를 보자.

```bash
./gradlew clean build
```

이 작업은 다음 특성을 가진다.

- 실행 명령이 정해져 있음
- CPU/RAM 사용량이 큼
- 성공/실패를 exit code로 판정 가능
- Artifact를 남길 수 있음
- 개발자 PC 자원을 점유할 수 있음

따라서 기본 구조는 단순하다.

```text
Git SHA
  ↓
Cloud Runner
  ↓
Build
  ↓
PASS / FAIL
```

PASS라면 Agent를 호출하지 않는다.

FAIL이라면 먼저 컴파일 오류를 구조화한다.

예:

```text
BUILD: FAIL

Error:
AuthService.java:142
cannot find symbol
JwtExpiredException
```

그다음 코드 수정이 필요하면 Agent를 호출한다.

좋지 않은 방식은 다음과 같다.

```text
Cloud Agent 시작
→ Repository 전체 탐색
→ build 방법 추론
→ build 실행
```

이미 `./gradlew build`가 표준 명령이라면 앞 단계의 LLM 추론은 불필요하다.

### 반환 Evidence

Build 작업은 최소한 다음 정보를 남긴다.

```text
Git SHA
Build Status
Build Duration
Artifact Path
Compile Error Summary
```

---

## 3. Unit Test는 Cloud Compute를 사용하기 좋은 작업이다

Unit Test는 결정론적이고 병렬화하기 쉽다.

예:

```bash
./gradlew test
```

또는 범위를 줄여 실행할 수 있다.

```bash
./gradlew test --tests AuthServiceTest
```

기본 흐름:

```text
Cloud Runner
→ Unit Test
→ PASS
   └─ 종료
→ FAIL
   └─ 실패 테스트 추출
```

실패했을 때도 전체 로그를 Agent에게 바로 넘기지 않는다.

먼저 다음처럼 필요한 정보만 뽑는다.

```text
failed test
assertion message
exception type
top stack frame
```

예:

```text
AuthServiceTest.expiredToken
expected: 401
actual: 200
AuthService.java:142
```

이 정도 정보로 수정 판단이 필요할 때만 Cloud Agent를 호출한다.

Unit Test가 Cloud에 특히 잘 맞는 경우는 다음과 같다.

```text
테스트 수가 많음
실행시간이 김
모듈별 분할 가능
CPU 사용량 큼
로컬 개발환경 점유가 큼
```

---

## 4. Integration Test는 환경 준비가 핵심이다

Integration Test는 Unit Test보다 환경 의존성이 크다.

예를 들어 Spring Boot 프로젝트에서 다음이 필요할 수 있다.

```text
Spring Boot
PostgreSQL
Redis
Kafka
Testcontainers
Mock HTTP Server
```

이 자원을 Cloud에서 재현할 수 있다면 Integration Test는 Cloud Runner에 잘 맞는다.

```text
Prepared Cloud Environment
        ↓
Integration Runner
        ↓
PostgreSQL / Redis / Kafka
        ↓
Integration Test
        ↓
Evidence
```

반대로 실제 내부 시스템에 직접 의존하면 판단이 달라진다.

```text
Integration Test
→ VPN 내부 Tibero 필요
→ 내부 Redis 필요
→ 사내 API 필요
```

이 경우 선택지는 두 가지다.

```text
Cloud 재현 가능한 대체 환경 구성
```

또는:

```text
Cloud 일반 검증
→ Local 내부망 최종 검증
```

즉 Integration Test는 `Cloud에 좋다`보다 `Cloud에서 재현 가능한가`가 먼저다.

---

## 5. E2E와 Browser Test는 Cloud와 잘 맞는다

E2E는 개발자의 Local 환경을 오래 점유하기 쉽다.

예:

```text
Playwright
→ Login
→ Attendance
→ Validation Error
→ Timeout
→ Retry
→ Browser Screenshot
```

Cloud에서는 Browser를 별도 환경에 분리할 수 있다.

```text
Cloud Runner
→ Browser 시작
→ E2E 실행
→ Screenshot / Video / Trace 저장
```

테스트 수가 많다면 여러 Worker로 나눌 수도 있다.

```text
Worker #1 → auth scenarios
Worker #2 → attendance scenarios
Worker #3 → admin scenarios
```

실패가 나더라도 처음부터 Agent에게 전체 Browser Trace를 보여주지 않는다.

예:

```text
1,200 scenarios
1,187 PASS
13 FAIL
```

먼저 실패 목록과 핵심 오류를 추출한 뒤 필요한 경우에만 Agent가 특정 실패를 분석한다.

### 반환 Evidence

E2E 결과는 다음이 유용하다.

```text
Scenario Count
Failed Scenario List
Screenshot
Video
Browser Trace
Console Error Summary
```

UI 변경에서는 이후 8장에서 다룰 `Demos over Diffs`와 연결할 수 있다.

---

## 6. Docker Build도 Runner가 먼저다

Docker Image Build는 LLM 판단 없이 실행할 수 있다.

```bash
docker build -t campus-platform:abc123 .
```

Cloud에 특히 유리한 경우:

- Image Build 시간이 김
- Local Disk/CPU 점유가 큼
- 여러 서비스 Image를 병렬 Build 가능
- Registry에 결과를 남길 수 있음

기본 흐름:

```text
Git SHA
→ Cloud Runner
→ Docker Build
→ Image Digest / PASS / FAIL
```

실패했을 때만 Agent가 필요하다.

예:

```text
Dockerfile syntax error
Dependency download failure
Base image incompatibility
```

9장에서는 Docker Layer Cache와 Prepared Environment로 시작 비용을 줄이는 방법을 다룬다.

---

## 7. Static Analysis와 Lint는 도구를 먼저 사용한다

다음 작업은 기본적으로 일반 프로그램이 처리한다.

```text
Formatter
Lint
Checkstyle
SpotBugs
Architecture Check
Dependency Check
Security Scan
```

구조는 단순하다.

```text
Tool
→ PASS / FAIL
```

FAIL 자체를 Agent에게 판단시킬 필요는 없다.

또 자동 수정 도구가 있다면 Agent보다 먼저 사용한다.

```text
Lint FAIL
   ↓
Auto-fix 가능?
  ├─ YES → Tool Fix → Recheck
  └─ NO  → Agent 후보
```

핵심 원칙은 다음이다.

> 코드로 판정하고 코드로 수정할 수 있는 규칙은 Agent보다 도구를 먼저 사용한다.

---

## 8. Migration은 작성과 검증을 분리한다

DB Migration 작업은 한 덩어리로 보지 않는다.

```text
Migration 작성
→ Developer 또는 Agent

Migration 검증
→ Runner
```

예를 들어 Cloud에서 PostgreSQL/Testcontainers를 사용할 수 있다면 다음을 자동 검증할 수 있다.

```text
clean DB 적용
이전 schema에서 upgrade
syntax
migration 순서
기본 Integration Test
```

하지만 실제 운영 대상이 Tibero나 Oracle이고 내부망에서만 접근 가능하다면 마지막 검증은 Local/Hybrid가 된다.

```text
Cloud
→ 일반 Migration Validation

Local
→ Tibero 실제 적용 검증
```

Migration도 `Agent가 작성했으니 Agent가 검증한다`는 구조를 사용하지 않는다.

검증은 Runner가 담당한다.

---

## 9. 반복 Refactoring은 Cloud Agent 후보가 된다

반복 Refactoring은 판단과 코드 변경이 필요하지만 범위를 제한하기 쉽다.

예:

```text
deprecated API 교체
Java 17 → 21 호환 수정
package import 변경
동일 패턴 반복 수정
```

좋은 Task:

```text
attendance 모듈에서 deprecated API 6건을 교체하고
./gradlew :attendance:test를 통과한다.
```

좋지 않은 Task:

```text
프로젝트 전체 구조를 더 좋게 리팩터링해.
```

Cloud Agent에 보내기 좋은 Refactoring은 다음 조건을 가진다.

```text
변경 패턴 명확
Scope 제한
검증 명령 존재
독립 Branch 가능
Review 가능한 Diff 크기
```

Architecture 수준 Refactoring은 Local에서 방향을 결정하고, 실행 가능한 작은 Task로 나눠 Cloud에 보낸다.

---

## 10. 작은 Bug Fix는 재현 가능성이 중요하다

Cloud Agent에 보내기 좋은 Bug Fix는 실패가 명확하다.

예:

```text
Failure
AuthServiceTest.expiredToken
expected: 401
actual: 200

Relevant Files
AuthService.java
JwtTokenProvider.java
AuthServiceTest.java

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

권장 흐름:

```text
Reproducible Failure
       ↓
Cloud Agent
       ↓
Analyze / Fix
       ↓
Cloud Runner
       ↓
PASS
```

반대로 다음 Bug는 Local에 더 적합할 수 있다.

```text
Production에서만 간헐 발생
내부 DB 상태 의존
HSM 실제 장비 의존
재현 절차 불명확
여러 시스템을 동시에 조사해야 함
```

Cloud Agent의 성능보다 재현 가능성이 먼저다.

---

## 11. Documentation도 Cloud Task가 될 수 있다

문서 작업이라고 모두 Agent에 보내는 것은 아니다.

Cloud에 적합한 예:

```text
API 변경에 맞춘 README 수정
특정 모듈 문서 갱신
release note 초안
변경 코드 기반 운영 문서 수정
```

이 작업은 Source Code와 Commit을 근거로 작성할 수 있다.

반대로 다음 작업은 Local 중심이 낫다.

```text
새 Architecture 원칙 정의
조직 전체 개발 정책 작성
요구사항이 아직 정해지지 않은 문서
```

즉 사실 기반 갱신은 Cloud에 적합할 수 있지만 방향을 정하는 문서는 Human Steering이 더 필요하다.

---

## 12. PR Review는 보조 Worker로 사용한다

Cloud Agent는 PR Review 보조에 사용할 수 있다.

검토 후보:

```text
변경 범위 확인
테스트 누락
명확한 오류
반복 규칙 위반
문서/코드 불일치
```

권장 구조:

```text
PR
→ Deterministic Checks
→ Cloud Review Agent
→ Findings
→ Human Review
```

Cloud Agent의 Review를 최종 승인과 같은 것으로 취급하지 않는다.

먼저 CI와 규칙 기반 검사를 수행하고, 그 위에서 Agent가 의미 기반 검토를 보조한다.

---

## 13. Dependency Update는 Runner-first의 대표 사례다

Dependency Update는 Cloud Agent를 언제 호출하지 않아도 되는지 보여주는 좋은 예다.

```text
Version Update
      ↓
Cloud Runner
→ Build / Test
      ↓
+----+----+
|         |
PASS      FAIL
|         |
PR      Agent
          ↓
     Compatibility Fix
          ↓
       Runner
```

PASS하면 LLM이 필요 없다.

FAIL하면 Agent에게 다음만 전달한다.

```text
변경 Dependency
이전 Version
신규 Version
실패 Build/Test
관련 파일
```

Repository 전체를 처음부터 다시 분석시키지 않는다.

---

## 14. CI Failure Fix는 Event-driven Agent와 잘 맞는다

CI 실패는 Cloud Agent 호출 조건으로 사용하기 좋다.

```text
Git Push / PR
     ↓
    CI
     ↓
+----+----+
|         |
PASS      FAIL
|         |
Done   Failure Summary
          ↓
     Cloud Agent
          ↓
        Fix
          ↓
       Runner
          ↓
        PASS
```

이 구조에서 Agent는 상시 실행되지 않는다.

실패가 생기고 수정 판단이 필요할 때만 호출된다.

14장에서는 CI Failure, Review Comment, Nightly Test, Dependency Update 같은 이벤트를 Cloud Task로 만드는 구조를 상세히 다룬다.

---

## 15. 같은 작업도 Runner와 Agent를 오간다

중요한 것은 `이 작업은 Runner 작업`, `이 작업은 Agent 작업`이라고 영구적으로 분류하는 것이 아니다.

하나의 Task 안에서도 실행 주체가 바뀐다.

예:

```text
Unit Test 실행
→ Runner

실패 원인 분석
→ Agent

코드 수정
→ Agent

전체 Test 재실행
→ Runner
```

또 Migration은 다음처럼 이동한다.

```text
Migration 작성
→ Agent / Developer

PostgreSQL Validation
→ Cloud Runner

Tibero 실검증
→ Local
```

즉 개발 Workflow는 실행 주체를 단계별로 바꾸는 구조다.

---

## 16. 작업별 기본 분류표

책에서 사용할 기본값은 다음과 같다.

| 작업 | 기본 실행 주체 | Agent 호출 조건 |
| --- | --- | --- |
| Build | Cloud Runner | Build 실패 분석/수정 필요 |
| Unit Test | Cloud Runner | 실패 원인 분석/수정 필요 |
| Integration Test | Cloud Runner | 재현 가능한 실패 수정 필요 |
| E2E | Cloud Runner | 실패 시 UI/코드 분석 필요 |
| Docker Build | Cloud Runner | Dockerfile/Dependency 수정 필요 |
| Lint / Static Analysis | Tool / Runner | 자동 수정 불가 |
| Migration Validation | Cloud Runner | Migration 수정 필요 |
| 반복 Refactoring | Cloud Agent | 처음부터 판단/수정 필요 |
| 작은 Bug Fix | Cloud Agent | 재현 가능해야 함 |
| Documentation | Cloud Agent 후보 | 사실 기반 범위가 명확할 때 |
| PR Review | Cloud Agent 보조 | Human Review 전 보조 |
| Dependency Update | Runner-first | FAIL일 때 Agent |
| CI Failure Fix | Agent-on-failure | CI FAIL 후 호출 |

이 표는 시작점일 뿐이다.

Internal Network, Context 크기, Human Steering 같은 조건은 5장의 Routing 기준을 다시 적용한다.

---

## 17. campus-platform Cloud Task Catalog

`campus-platform`에서 반복 사용할 Task를 미리 정의할 수 있다.

### Backend Unit Test

```text
Input
Git SHA
Module

Runner
./gradlew :auth:test

Output
PASS / FAIL
JUnit Result
```

### Integration Test

```text
Input
Git SHA

Environment
Java 21
PostgreSQL/Testcontainers

Runner
integrationTest
```

### Web E2E

```text
Environment
Node
Playwright
Chrome

Runner
npx playwright test

Evidence
Screenshot
Video
Trace
```

### Docker Build

```text
Runner
docker build

Evidence
Image Digest
Build Log
```

### Auth Bug Fix

```text
Agent
AuthService expired token 수정

Validation
AuthServiceTest.expiredToken
```

### Migration

```text
Cloud
PostgreSQL/Testcontainers Validation

Local
Tibero 실제 검증
```

이런 Catalog가 있으면 Task를 만들 때마다 실행 방법을 다시 설명할 필요가 줄어든다.

---

## 18. 이 장에서 기억할 실행 순서

실제 작업을 받으면 다음 순서로 본다.

```text
1. Cloud에서 실행할 가치가 있는가?
2. Runner만으로 처리 가능한가?
3. 실패했는가?
4. 판단이나 코드 수정이 필요한가?
5. Agent가 수정했으면 Runner로 재검증했는가?
6. 필요한 Evidence를 남겼는가?
```

이를 한 줄로 줄이면 다음과 같다.

```text
Runner
→ 필요할 때 Agent
→ 다시 Runner
```

다음 장에서는 Agent에게 실제 작업을 넘길 때 Repository 전체를 다시 탐색하지 않도록 `Task Contract`와 작은 Context를 구성한다.
