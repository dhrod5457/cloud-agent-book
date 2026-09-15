# 16장 설계 - 하나의 기능을 Local + Cloud로 끝까지 개발하기

## 장의 목표

15장에서 전체 운영 모델을 설계했다면, 16장에서는 하나의 기능을 요구사항 분석부터 Task 분해, Cloud 실행, 실패 수정, Evidence 수집, 내부망 검증, Review, Merge까지 시간 순서대로 따라간다.

핵심 질문:

> 실제 기능 하나를 개발할 때 Local과 Cloud를 언제 오가고, 어떤 Evidence를 기준으로 다음 단계로 넘어가는가?

---

## 핵심 주장

> Cloud Agent 활용은 별도 도구 사용법이 아니라 개발 Workflow 설계다.

> 각 단계는 다음 단계로 넘길 수 있는 명시적인 입력과 Evidence를 가져야 한다.

> Cloud에서 실패하면 Context를 한 번에 넓히지 말고 필요한 정보만 추가한다.

---

## 실전 시나리오

예제 기능:

```text
학생 출결 API 변경
```

요구사항 예:

```text
학생이 만료된 인증 토큰으로 출결 API를 호출하면 401을 반환한다.
정상 토큰의 기존 출결 등록 동작은 유지한다.
관리자 Web에서 실패 상태를 확인할 수 있어야 한다.
```

이 기능은 Backend, Integration, Web E2E, Docker 검증이 필요하다고 가정한다.

---

# 단계별 흐름

## 16.1 Local - 요구사항 해석

Local에서 먼저 결정한다.

- 정상/실패 동작
- 영향 모듈
- 변경 금지 영역
- 내부 시스템 의존성
- Cloud로 분리 가능한 검증

초기 판단:

```text
Architecture 결정
→ Local

Backend 구현
→ Local 또는 Cloud Agent 후보

Unit/Integration/E2E/Docker
→ Cloud Runner

Tibero 실제 검증
→ Local
```

---

## 16.2 Local - 영향 범위 확인

예상 파일:

```text
AuthService.java
JwtTokenProvider.java
AttendanceController.java
AuthServiceTest.java
AttendanceApiTest.java
```

확인:

- DB schema 변경 없음
- HSM 변경 없음
- 공통 Exception format 변경 없음

이 정보를 Task Contract에 사용한다.

---

## 16.3 Local - Base Commit을 만든다

Cloud Task 기준점을 고정한다.

```text
base_sha: abc123
```

필요한 로컬 변경을 Commit/Push한다.

```text
Local
→ Test
→ Commit
→ Push
```

미커밋 상태를 Cloud에 암묵적으로 기대하지 않는다.

---

## 16.4 Task를 분해한다

예:

```text
Task A
Auth expired token backend fix

Task B
Backend unit/integration validation

Task C
Admin E2E validation

Task D
Docker build validation
```

Task A는 Agent 작업이고 B~D는 Runner 중심이다.

---

## 16.5 Task Contract 작성

Task A:

```text
Task
Expired token 처리 수정

Goal
Attendance API 인증 실패 → HTTP 401

Base SHA
abc123

Scope
- auth
- attendance auth boundary

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Do Not Change
- DB schema
- OAuth 전체 구조
- Common Exception format

Validation
./gradlew test --tests AuthServiceTest

Expected
expired token → 401
normal token → existing tests PASS

Output
commit
test evidence
changed files
```

---

## 16.6 Cloud - 독립 Branch/Environment 준비

```text
branch: agent/task-auth-expired
environment: backend-test
```

Prepared Environment 사용:

- Java 21
- Gradle cache
- Docker/Testcontainers 준비

Source와 Branch는 Fresh하게 checkout한다.

---

## 16.7 Cloud Agent - 분석과 수정

Agent에게 처음 제공:

- Task Contract
- Relevant Files
- 실패 테스트

전체 Repository를 처음부터 읽지 않는다.

필요 시 Context 확장:

```text
Relevant Files
→ direct dependency
→ auth docs
→ wider source
```

수정 후 Commit을 만든다.

---

## 16.8 Cloud Runner - Target Test

```text
./gradlew test --tests AuthServiceTest
```

PASS라면 Regression 단계로 이동한다.

FAIL이라면 Result Gateway가 Summary를 만든다.

---

## 16.9 실패 시 작은 Context로 재진입

예:

```text
STATUS: FAIL
Test: AuthServiceTest.expiredToken
expected: 401
actual: 200
Stack: AuthServiceTest.java:94
```

Agent에게 전체 build.log 대신 이 Summary를 먼저 전달한다.

필요하면 특정 Test Log만 추가 조회한다.

---

## 16.10 Retry와 Failure Fingerprint

```text
Retry #1
expected 401 / actual 200

Retry #2
expected 401 / actual 200
```

동일 fingerprint 반복이면 무한 수정하지 않는다.

```text
same failure twice
→ stop
→ Local/Human escalation
```

---

## 16.11 Cloud - 병렬 Regression

수정 Commit이 나오면 같은 SHA 기준으로 병렬 검증한다.

```text
Commit def456
      ↓
+---------+---------+---------+---------+
|         |         |         |         |
Unit   Integration Docker    E2E
|         |         |         |         |
+---------+---------+---------+---------+
```

결과 예:

```text
Unit: PASS
Integration: PASS
Docker: PASS
E2E: FAIL
```

---

## 16.12 E2E 실패 분석

E2E Artifact:

- screenshot
- browser video
- trace
- console log

Agent는 먼저 Summary와 실패 Screenshot을 본다.

UI 문제라면 Demos over Diffs를 사용한다.

---

## 16.13 Evidence Result 생성

최종 Cloud 결과:

```text
Task: task-auth-expired
Commit: def789

Build: PASS
Unit: 314/314 PASS
Integration: 42/42 PASS
Docker: PASS
E2E: PASS

Artifacts:
- junit.xml
- integration-junit.xml
- screenshot-after.png
- e2e-trace.zip
- build.log

PR: #142
```

---

## 16.14 Local - PR Review

Developer/Local Agent가 확인:

1. Evidence
2. Changed Files
3. Diff
4. Architecture 영향

UI가 있다면 Screenshot/Video를 먼저 보고 Diff를 확인할 수 있다.

---

## 16.15 Local - 내부망 검증

Cloud에서 검증하지 못한 부분:

```text
Tibero
Internal API
Jenkins deployment job
```

필요한 경우 실제 환경에서 검증한다.

HSM 변경이 없는 Task라면 HSM 검증은 생략할 수 있다.

Task별 필요한 내부 검증만 수행한다.

---

## 16.16 Merge 전 Full Validation

최종 기준 SHA에서 필요한 full validation을 수행한다.

```text
PR SHA
→ required checks
→ internal validation
→ Review complete
→ Merge
```

Cloud Worker 개별 결과만 보고 merge하지 않는다.

---

## 16.17 Merge 후 Task 종료

작업 종료 시 남기는 것:

- Merge Commit
- PR
- Evidence
- 주요 실패/Retry 정보

장기 Agent Memory 시스템을 만드는 것이 아니라 이번 Task를 나중에 추적할 최소 기록을 남긴다.

---

## 16.18 시간 흐름 예

```text
09:30 Requirement / Design
10:00 Base Commit
10:05 Cloud Task 시작
10:06 Developer 다음 작업
10:22 Agent Fix Commit
10:23 Parallel Validation
10:40 E2E Failure
10:45 Agent Fix
10:58 All PASS
11:20 Developer Review
11:35 Internal Validation
11:45 Merge
```

중요:

```text
Cloud elapsed time
!= Developer blocking time
```

---

## 16.19 비용 흐름 표시

### Compute

- Unit/Integration
- Docker
- E2E

### LLM

- Backend failure analysis/fix
- E2E failure fix

### Human

- Requirement
- Review
- Internal validation

어떤 단계가 병목인지 확인한다.

---

## 16.20 같은 기능을 전부 Local에서 했을 때와 비교

Local-only:

```text
Developer
→ 구현
→ Test 대기
→ Integration 대기
→ Docker 대기
→ E2E 대기
→ 오류 분석
```

Hybrid:

```text
Developer
→ 설계 / 구현 / 위임
→ 다음 작업

Cloud
→ 검증/수정

Developer
→ Review / Internal Validation
```

항상 Hybrid가 더 빠르다고 주장하지 않고 실제 Blocking Time/Overhead를 비교한다.

---

# 실패 분기 사례

## Cloud에서 재현 안 됨

```text
Cloud Runner
→ cannot reproduce
→ Local Fallback
```

## 내부 DB 필요 발견

```text
Cloud analysis
→ Tibero-specific behavior
→ Local validation
```

## Task Scope 확대

```text
Auth 3개 파일 예상
→ 실제로 8개 module 영향
→ Cloud 중단
→ Local 재분해
```

---

# 필요한 그림

1. Requirement → Merge 전체 Timeline
2. Task Split / Runner-Agent 구분
3. Failure Context Expansion
4. Parallel Regression
5. Evidence → Local Review → Internal Validation
6. Blocking Time 비교

---

# Phase 6 실제 예제 후보

가능하면 작은 실행 가능한 샘플로 구성한다.

```text
Auth expired token
+ Spring Boot test
+ Testcontainers
+ Playwright mock UI
```

실제 대학 시스템 내부 정보에 종속되지 않게 책용 예제를 단순화한다.

---

# 앞뒤 장 연결

15장:
전체 운영 아키텍처

16장:
실제 기능의 시간 순 실행

17장:
이 Workflow를 적용하지 말아야 하는 Task를 정리

---

# 의도적으로 다루지 않을 내용

- 특정 Cloud Agent 제품 클릭 가이드
- 실제 사내 Secret/Network 값
- 완전 자동 Merge
- Agent Platform 장기 Memory

---

# 장의 결론 메시지

> Cloud Agent는 Workflow 중간에 들어가는 Remote Worker다.

> 작은 Task를 Git으로 넘기고, Runner와 Agent가 작업한 Evidence를 다시 Local로 가져온다.

> Cloud 실패 시 Context를 무작정 늘리지 말고 필요한 정보만 단계적으로 추가한다.
