# 6장 설계 - Cloud에 보내기 좋은 개발 작업

## 장의 목표

5장에서 `이 Task를 Local에서 할 것인가, Cloud로 보낼 것인가`를 판단하는 기준을 만들었다.

6장에서는 그 기준을 실제 개발 작업 유형에 적용한다.

이 장의 목적은 작업 이름만 보고 Cloud로 보내는 것이 아니라, 각 작업을 다음 세 가지 실행 방식으로 구분할 수 있게 하는 것이다.

```text
Deterministic Work
→ Cloud Runner

Reasoning + Code Change
→ Cloud Agent

Internal / Interactive Work
→ Local 또는 Hybrid
```

핵심 질문:

> 이 개발 작업은 Cloud에서 무엇이 실행되어야 하고, 그중 어디까지 LLM이 필요한가?

---

## 핵심 주장

> Cloud에 보내기 좋은 작업과 Cloud Agent가 직접 해야 하는 작업은 같은 개념이 아니다.

Build, Test, Docker Build, Lint처럼 실행 명령과 판정 기준이 명확한 작업은 Cloud에서 실행하기 좋지만 LLM이 필요하지 않을 수 있다.

반면 작은 Bug Fix, 반복 Refactoring, CI Failure Fix처럼 원인 분석과 코드 변경이 필요한 작업은 Cloud Agent가 적합할 수 있다.

따라서 기본 분류는 다음과 같다.

```text
Cloud Compute가 유리한가?
       ↓
YES
       ↓
판단이 필요한가?
   ├─ NO  → Cloud Runner
   └─ YES → Cloud Agent
```

또 다음 원칙을 유지한다.

> Runner가 할 수 있으면 Runner에게 맡긴다.

> Agent는 판단과 수정이 필요한 구간에만 사용한다.

---

## 독자가 얻는 것

- Build/Test 작업과 Agent 작업을 분리할 수 있다.
- Unit Test, Integration Test, E2E의 Cloud 적합성을 각각 판단할 수 있다.
- Docker Build, Static Analysis, Lint, Migration Validation을 Runner 작업으로 설계할 수 있다.
- 작은 Bug Fix와 반복 Refactoring을 Cloud Agent Task로 만들 수 있다.
- Documentation과 PR Review를 Cloud에 보낼 조건을 판단할 수 있다.
- Dependency Update와 CI Failure Fix를 `Runner → Agent → Runner` 흐름으로 만들 수 있다.
- 각 작업에서 Cloud에 전달해야 할 입력과 반환받아야 할 Evidence를 정의할 수 있다.
- 같은 작업이라도 단계별로 Runner, Agent, Local을 이동할 수 있음을 이해한다.

---

# 절 구성

## 6.1 작업 이름보다 실행 구조를 본다

같은 `테스트 작업`이라도 실제 실행 방식은 다를 수 있다.

예:

```text
전체 Unit Test 실행
→ Cloud Runner

실패한 Unit Test 원인 분석
→ Cloud Agent

수정 후 전체 Unit Test 재실행
→ Cloud Runner
```

따라서 작업을 다음 요소로 나눈다.

- 실행 명령
- 판단 필요 여부
- 코드 변경 필요 여부
- 검증 방법
- 실행시간/Compute 사용량
- Context 크기
- 외부 시스템 의존성
- 반환할 Evidence

이 장에서는 이 분해 방식으로 각 개발 작업을 살펴본다.

---

## 6.2 Build

Build는 Cloud Runner에 가장 먼저 보내기 좋은 작업 중 하나다.

예:

```text
./gradlew clean build
```

특징:

- 명령이 명확함
- CPU/RAM 사용
- 결과를 exit code로 판정 가능
- Artifact 생성 가능
- 개발자 PC 점유를 줄일 수 있음

권장 구조:

```text
Git SHA
  ↓
Cloud Runner
  ↓
Build
  ↓
PASS / FAIL
  ↓
Artifact
```

PASS라면 Agent가 필요하지 않다.

FAIL이라면 컴파일 오류 요약을 만든 뒤 Agent 호출 여부를 판단한다.

### 반환 Evidence

- build status
- compile error summary
- artifact path
- build duration
- Git SHA

### 좋지 않은 방식

```text
Cloud Agent 시작
→ Repository 전체 분석
→ build 명령 찾기
→ build 실행
```

Build 명령이 이미 결정되어 있다면 앞 단계의 LLM 추론은 불필요하다.

---

## 6.3 Unit Test

Unit Test는 Cloud의 병렬성과 Compute를 사용하기 좋은 작업이다.

예:

```text
./gradlew test
```

또는 범위를 줄인 실행:

```text
./gradlew test --tests AuthServiceTest
```

기본 흐름:

```text
Cloud Runner
→ Unit Test
→ PASS: 종료
→ FAIL: 실패 테스트 추출
```

실패가 발생해도 먼저 Agent를 부르지 않고 다음 정보를 결정론적으로 추출한다.

- failed test name
- assertion message
- exception type
- top stack frame

그 정보만으로 코드 수정 판단이 필요할 때 Agent를 호출한다.

### Cloud에 특히 적합한 조건

- 테스트 수가 많음
- 실행시간이 김
- 여러 모듈로 분할 가능
- 개발자 PC의 CPU/RAM 점유가 큼

---

## 6.4 Integration Test

Integration Test는 Unit Test보다 Cloud 환경 준비가 중요하다.

대표 구성:

- Spring Boot
- PostgreSQL/Testcontainers
- Redis
- Kafka
- Mock HTTP Server

권장 구조:

```text
Prepared Cloud Environment
        ↓
Integration Runner
        ↓
Testcontainers / Mock
        ↓
Test
        ↓
Evidence
```

Cloud에 적합하려면 외부 의존성을 Cloud에서 재현할 수 있어야 한다.

다음처럼 실제 내부 시스템만을 요구하면 Cloud 적합도가 낮아진다.

```text
Integration Test
→ VPN 내부 Tibero 필요
→ 내부 Redis 필요
→ 사내 API 필요
```

이 경우 테스트를 격리하거나 Local/Hybrid로 보내야 한다.

9장에서는 Prepared Environment, 13장에서는 Hybrid를 상세히 설명한다.

---

## 6.5 E2E와 Browser Test

E2E는 Cloud의 독립 실행환경을 활용하기 좋은 작업이다.

예:

```text
Playwright
→ 회원가입
→ 로그인
→ 잘못된 입력
→ timeout
→ retry
→ accessibility
```

장점:

- Browser 실행을 Local에서 분리
- 여러 시나리오 병렬 실행
- screenshot/video/trace 저장
- 장시간 regression 위임

Cloud Runner가 먼저 실행하고 실패한 경우에만 Agent가 분석한다.

```text
1,200 scenarios
→ 1,187 PASS
→ 13 FAIL
→ Failure Summary
→ 필요 시 Cloud Agent
```

### 반환 Evidence

- scenario count
- failed scenario list
- screenshot
- video
- browser trace
- console error summary

UI에서는 `Demos over Diffs`를 1차 검증 방식으로 연결할 수 있지만 상세 내용은 8장에서 다룬다.

---

## 6.6 Docker Build

Docker Build는 Cloud Runner 작업으로 분류한다.

```text
docker build ...
```

Cloud가 유리한 경우:

- image build가 오래 걸림
- Local disk/CPU 사용량이 큼
- 여러 서비스 image를 병렬 build 가능
- 결과 image/digest를 Artifact로 남길 수 있음

주의할 점:

- base image pull 비용
- Docker layer cache
- registry 접근
- Secret 전달 범위

Cache와 Prepared Environment는 9장에서 다룬다.

Agent는 Dockerfile 오류나 build failure의 원인 판단이 필요할 때만 호출한다.

---

## 6.7 Static Analysis와 Lint

다음 작업은 기본적으로 Runner다.

- formatter
- lint
- Checkstyle
- SpotBugs
- dependency check
- architecture check
- vulnerability scan

특징:

```text
명령
→ 결과
→ PASS / FAIL
```

FAIL 자체를 LLM에게 판단시킬 필요는 없다.

수정이 단순하고 규칙이 명확한 경우에는 자동 formatter/fixer를 먼저 사용한다.

```text
Lint FAIL
→ auto-fix 가능?
   ├─ YES → tool fix → recheck
   └─ NO  → Agent 후보
```

핵심:

> 코드로 수정할 수 있는 규칙은 Agent보다 도구를 먼저 사용한다.

---

## 6.8 Migration Validation

DB Migration은 `작성`과 `검증`을 분리한다.

예:

```text
Migration 작성
→ Agent 또는 Developer

Disposable DB에서 Migration 적용
→ Cloud Runner
```

검증 항목:

- migration 순서
- syntax
- clean DB 적용 가능 여부
- 이전 schema에서 upgrade 가능 여부
- rollback 정책이 있다면 rollback 검증

PostgreSQL/Testcontainers처럼 Cloud에서 재현 가능한 DB는 Runner로 검증한다.

Tibero/Oracle 실환경 검증처럼 내부망 의존성이 있다면:

```text
Cloud
→ 일반 migration validation

Local
→ Tibero 실제 검증
```

으로 Hybrid화한다.

---

## 6.9 반복 Refactoring

반복적인 Refactoring은 Cloud Agent에 적합할 수 있다.

예:

- deprecated API 변경
- 반복 코드 정리
- Java 17 → 21 호환 수정
- 패키지 import 변경
- 동일 패턴 적용

좋은 Task:

```text
attendance 모듈에서 deprecated API 6건을 새 API로 교체하고
./gradlew :attendance:test 로 검증
```

좋지 않은 Task:

```text
프로젝트 구조를 전반적으로 더 좋게 리팩터링해.
```

Cloud에 보내기 좋은 Refactoring은 다음 특성이 있다.

- 변경 패턴 명확
- Scope 제한
- 검증 명령 존재
- 독립 Branch에서 작업 가능
- Review 가능한 diff 크기

Architecture 수준 Refactoring은 Local에서 먼저 방향을 결정하고 Cloud에는 잘게 나눈 실행 Task를 보낸다.

---

## 6.10 작은 Bug Fix

Cloud Agent에 보내기 좋은 Bug Fix는 실패와 완료 조건이 명확하다.

예:

```text
Failure:
AuthServiceTest.expiredToken
expected: 401
actual: 200

Relevant Files:
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Validation:
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

반대로 다음 버그는 Local에 더 적합할 수 있다.

- 재현되지 않음
- Production에서만 간헐 발생
- 내부 DB/HSM 상태 의존
- 여러 시스템을 동시에 조사해야 함

---

## 6.11 Documentation

Documentation도 Cloud Task가 될 수 있다.

적합한 예:

- 변경 코드에 맞춘 API 문서 갱신
- 특정 모듈 README 업데이트
- release note 초안
- 명확한 코드 변경을 설명하는 문서 수정

주의:

문서 전체 구조나 Architecture 의도를 새로 정의하는 작업은 넓은 Context와 사람의 판단이 필요할 수 있다.

따라서:

```text
사실 기반 문서 갱신
→ Cloud 후보

새 Architecture 원칙 정의
→ Local 중심
```

문서 Task에도 변경 파일과 근거 Commit을 제한한다.

---

## 6.12 PR Review

Cloud Agent는 PR Review 보조에도 사용할 수 있다.

검토 후보:

- 변경 범위 확인
- 테스트 누락 확인
- 명확한 오류 탐지
- 반복 패턴 위반
- 문서/코드 불일치

그러나 최종 승인 권한과는 분리한다.

권장 구조:

```text
PR
→ deterministic checks
→ Cloud Review Agent
→ Findings
→ Human Review
```

제품별 자동 Review 기능 설명이 아니라 `검증 가능한 변경 범위를 비동기적으로 검토하는 Remote Worker` 사례로만 사용한다.

---

## 6.13 Dependency Update

Dependency Update는 Runner와 Agent를 결합하기 좋은 사례다.

```text
Dependency Update
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

FAIL하면 Agent에게 다음만 제공한다.

- 변경 dependency
- 이전/신규 version
- failed build/test
- 관련 파일

전체 Repository를 다시 분석시키지 않는다.

---

## 6.14 CI Failure Fix

CI 실패 수정은 Event-driven Cloud Agent와 연결하기 좋은 작업이다.

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
     Runner / CI
```

좋은 조건:

- 실패가 재현 가능
- 관련 테스트가 명확
- Branch가 독립적
- 수정 후 동일 CI로 재검증 가능

14장에서 Event-driven 호출과 Nightly/Review Comment 흐름을 상세히 다룬다.

---

## 6.15 작업별 기본 실행 주체 표

| 작업 | 기본 실행 주체 | Agent가 필요한 시점 |
| --- | --- | --- |
| Build | Cloud Runner | build 원인 분석/수정 |
| Unit Test | Cloud Runner | 실패 분석/수정 |
| Integration Test | Cloud Runner | 실패 분석/환경 판단 |
| E2E | Cloud Runner | 실패 UI/코드 분석 |
| Docker Build | Cloud Runner | Dockerfile/build 오류 수정 |
| Lint/Format | Tool/Runner | 자동 수정 불가 규칙 |
| Static Analysis | Cloud Runner | 구조적 오류 수정 |
| Migration Validation | Cloud Runner | migration 코드 수정 |
| 반복 Refactoring | Cloud Agent | 기본적으로 판단/수정 필요 |
| 작은 Bug Fix | Cloud Agent | 재현 실패 시 Local 전환 가능 |
| Documentation | Cloud Agent | 넓은 설계 판단은 Local |
| PR Review | Cloud Agent 보조 | 최종 승인 Human |
| Dependency Update | Runner-first | 실패 시 Agent |
| CI Failure Fix | Agent-on-failure | CI 실패 시 호출 |

이 표는 기본값이며 5장의 Task Routing 기준이 우선한다.

---

## 6.16 campus-platform Cloud Task Catalog

책 전체에서 반복해서 사용할 예제 Task를 정의한다.

### Task A - Backend Unit Test

```text
Execution: Cloud Runner
Command: ./gradlew test
Evidence: junit result
```

### Task B - Integration Test

```text
Execution: Cloud Runner
Environment: PostgreSQL/Testcontainers
Evidence: integration test result
```

### Task C - Web E2E

```text
Execution: Cloud Runner
Environment: Node + Playwright + Browser
Evidence: test result + screenshot/video
```

### Task D - Docker Build

```text
Execution: Cloud Runner
Evidence: image build result / digest
```

### Task E - Auth Bug Fix

```text
Execution: Cloud Agent
Task: expired token → 401
Validation: targeted AuthService test
Output: commit + changed files + test result
```

### Task F - Migration

```text
Cloud Runner
→ PostgreSQL/Testcontainers migration validation

Local
→ Tibero 실제 환경 최종 검증
```

이 Task Catalog는 15~16장의 실전 Workflow에서 재사용한다.

---

# 좋은 사례와 나쁜 사례

## 사례 A - 전체 테스트

좋지 않은 방식:

```text
Cloud Agent
→ Repository 분석
→ 전체 테스트 실행
→ 성공
```

Agent 추론이 성공 경로에 불필요하게 포함된다.

권장 방식:

```text
Cloud Runner
→ 전체 테스트
→ PASS
→ 종료
```

## 사례 B - CI Failure

좋지 않은 방식:

```text
Agent
→ CI 전체 로그 읽기
→ 원인 추측
```

권장 방식:

```text
CI
→ failed job/test 추출
→ 관련 로그만 제공
→ Cloud Agent 수정
→ CI 재검증
```

## 사례 C - Refactoring

좋지 않은 방식:

```text
전체 프로젝트를 깔끔하게 리팩터링해.
```

권장 방식:

```text
notification 모듈의 deprecated API 4건 교체
검증: ./gradlew :notification:test
```

---

# 필요한 구조/그림

## 작업 유형별 분기

```text
Development Task
       ↓
Deterministic Execution?
   ├─ YES → Cloud Runner
   │          ↓
   │      PASS / FAIL
   │          ↓
   │       FAIL only
   │          ↓
   │      Cloud Agent
   │
   └─ NO → Reasoning Required?
              ├─ Cloud suitable → Cloud Agent
              └─ Local constraint → Local / Hybrid
```

## 대표 Workflow

```text
Developer / Local Agent
          ↓
      Task Split
          ↓
+---------+----------+---------+
|                    |         |
Runner             Runner     Agent
Unit Test           E2E      Bug Fix
|                    |         |
+---------+----------+---------+
          ↓
       Evidence
          ↓
        Review
```

---

# 예제 Stage

Stage 2 - Cloud Task Catalog 정의

이 단계에서 `campus-platform`의 기능 구현 자체보다 다음을 명시한다.

- 어떤 명령을 Runner가 실행하는가
- 어떤 작업에서 Agent를 호출하는가
- 각 Task가 어떤 Evidence를 반환하는가
- 어떤 외부 의존성은 Cloud에서 대체하는가
- 어떤 검증은 Local에 남기는가

아직 Prepared Image나 Cache 구현은 하지 않는다. 해당 내용은 9장에서 다룬다.

---

# 필요한 코드/설정 예제

본문 작성 단계에서 필요한 예제:

- Gradle 전체/선택 테스트 명령
- module test 명령
- Testcontainers integration test 실행 예
- Playwright E2E 명령
- Docker build 명령
- migration validation script 예
- lint/static analysis command 예
- CI failure summary 예

이 장의 목적은 각 도구 사용법 교육이 아니라 `어떤 실행 주체에 배치할 것인가`를 보여주는 것이다.

---

# 필요한 공식 자료 조사

본문 작성 시 최신 공식 자료로 다음 사례를 확인한다.

- 주요 Cloud Coding Agent가 지원하는 비동기 작업/PR 흐름
- GitHub 계열 Cloud Agent의 CI failure/review 연동 사례
- Claude Code Web의 PR/CI follow-up 사례
- 제품별 E2E/browser 실행 지원 여부가 필요한 경우 해당 공식 문서

변경 가능성이 높은 제품 기능은 일반 원칙과 분리해 `research/`에서 기준일과 출처를 관리한다.

---

# 앞 장과 뒤 장의 연결

## 앞 장 - 5장

5장은 Task 특성을 보고 Local / Cloud / Hybrid / Runner-first 중 하나를 선택했다.

6장은 Cloud 후보로 분류된 Task를 실제 Build/Test/Refactoring/Bug Fix 등의 개발 작업으로 구체화한다.

## 뒤 장 - 7장

6장에서 Cloud Agent가 필요한 작업을 식별했다면 7장에서는 그 작업을 Agent에게 넘길 때 Scope와 Context를 어떻게 작게 유지하는지 `Cloud Agent Task Contract`로 정의한다.

즉:

```text
5장: 어디서 할 것인가
6장: 무엇을 Cloud에서 할 것인가
7장: Cloud Agent에게 어떻게 작게 넘길 것인가
```

---

# 본문에서 의도적으로 다루지 않을 내용

- 각 빌드/테스트 도구의 입문 사용법
- 제품별 Cloud Agent UI 사용법
- Prepared Image/Cache 구현 상세
- Branch/Worktree 병렬 격리 상세
- Result Gateway 상세 구조
- CI 이벤트 자동화 상세
- Agent Platform 일반론

각 항목은 후속 장에서 필요한 범위만 다룬다.

---

# 장 설계 체크리스트

- [x] Cloud Runner와 Cloud Agent를 작업별로 구분한다.
- [x] Build/Test 성공 경로에 불필요한 LLM을 넣지 않는다.
- [x] Integration/E2E처럼 Compute-heavy 작업을 포함한다.
- [x] Bug Fix/Refactoring처럼 reasoning이 필요한 작업도 구체화한다.
- [x] Internal 시스템 의존 시 Hybrid로 전환한다.
- [x] Evidence 반환을 각 작업에 연결한다.
- [x] campus-platform에서 재사용할 Task Catalog를 정의한다.
- [x] 5장 Routing과 7장 Task Contract 사이의 역할을 분리한다.
- [x] Agent Platform 일반론으로 확장하지 않는다.
