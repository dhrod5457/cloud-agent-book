# Cloud Agent Remote Worker 운영 모델

## 목적

이 문서는 현재 책의 Cloud Agent 중심 방향을 실제 개발 Workflow 관점에서 구체화한다.

본문이 아니라 Phase 5 설계 자료다.

핵심 질문은 다음과 같다.

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

## Cloud Agent 정의

Cloud Agent를 단순히 클라우드에서 실행되는 LLM으로 정의하지 않는다.

```text
Cloud Agent
= LLM
+ Repository
+ Independent Execution Environment
+ CPU
+ RAM
+ Disk
+ Development Tools
```

책의 후반부에서 다시 사용할 정의:

> Cloud Agent는 필요할 때 독립된 개발환경을 할당받고, Git을 통해 Task를 받아 비동기적으로 작업하며, 테스트와 Artifact를 포함한 검증 가능한 결과를 반환하는 Remote Development Worker다.

핵심 메시지:

> 클라우드 에이전트의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

---

# 1. Prepared Cloud Environment를 팀 자산으로 관리한다

Cloud Agent가 시작될 때마다 개발환경을 처음부터 구성하지 않는다.

좋지 않은 흐름:

```text
Agent 시작
→ JDK 확인/설치
→ Node 설치
→ Gradle dependency 다운로드
→ npm install
→ Playwright 설치
→ Docker image pull
→ Test 시작
```

권장 흐름:

```text
Prepared Cloud Environment
- Java 21
- Gradle
- Gradle dependency cache
- Node
- npm dependencies/cache
- Docker
- Playwright
- Static Analysis Tools
- Test configuration
        ↓
Cloud Agent
→ Branch checkout
→ 바로 작업
```

이를 `Prepared Cloud Environment` 또는 `Cloud Environment as Code` 관점으로 설명한다.

환경도 Repository 코드처럼 버전과 변경 이력을 관리할 수 있어야 한다.

원칙:

> Agent에게 개발환경을 설치하게 하지 말고, 바로 작업 가능한 환경을 제공한다.

## 작업별 Environment

모든 Task에 동일한 최대 환경을 제공할 필요는 없다.

```text
backend-test
- Java 21
- Gradle
- Testcontainers
- Docker
- DB client

frontend-e2e
- Node
- Chrome
- Playwright

migration-test
- Java
- migration tool
- DB client
- migration scripts

fullstack
- Java
- Node
- Docker
- Browser
```

Task Scheduler 또는 개발자가 Task 성격에 따라 실행환경을 선택한다.

```text
Backend Unit Test  → backend-test
Frontend E2E       → frontend-e2e
Database Migration → migration-test
```

---

# 2. Cold Start를 별도 비용으로 본다

Cloud Agent의 실제 작업 시작 시간에는 모델 응답시간뿐 아니라 다음이 포함될 수 있다.

- VM/Container 생성
- Repository clone
- Dependency 설치
- Build cache 생성
- Docker image pull
- Development tool 준비

이를 `Cloud Agent Cold Start`로 구분한다.

```text
Cold Start
VM 생성
→ clone
→ install
→ build/cache
→ Agent 실제 작업
```

개선:

```text
Prepared Snapshot
→ Branch checkout
→ Agent 실제 작업
```

최적화 대상은 모델 latency 하나가 아니라 `Task submitted → useful work started` 전체 시간이다.

---

# 3. Cache와 Fresh State를 분리한다

재사용 영역:

- JDK / Node runtime
- Gradle/Maven dependency
- npm cache
- Docker layer
- Playwright browser
- compiler cache

항상 최신이어야 하는 영역:

- Source Code
- Branch
- Task
- Test Result
- Temporary Data

```text
Prepared Environment
Dependencies + Tools + Cache
            +
Fresh State
Source + Branch + Task
```

이 구조는 Dependency 설치 비용을 줄이면서 Code/Task 상태는 최신으로 유지한다.

---

# 4. Local → Cloud → Local Handoff

Local Agent와 Cloud Agent를 경쟁 관계로 보지 않는다.

하나의 작업도 단계에 따라 실행 위치를 이동할 수 있다.

```text
Local
→ 요구사항 분석
→ Architecture 결정
→ 핵심 코드 작성 / Task 분해
       ↓
Cloud
→ 전체 Test
→ Integration Test
→ E2E
→ Docker Build
→ 반복 Refactoring
→ CI 문제 수정
       ↓
Local
→ 최종 Review
→ 내부망 검증
→ 통합
→ Merge
```

핵심:

> Task는 Local 또는 Cloud 중 하나에 영구적으로 속하는 것이 아니다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

---

# 5. Git을 Handoff Boundary로 본다

Cloud Agent는 보통 Remote Repository의 clean state에서 작업하기 쉽다.

따라서 Git을 Local과 Cloud 사이의 작업 전달 경계로 설명한다.

```text
Local Developer / Agent
→ 수정
→ Test
→ Commit
→ Push
       ↓
Git Repository
       ↓
Cloud Worker
→ Task Branch
→ 작업
→ Test
→ Commit
→ Push
→ PR
```

핵심:

> Cloud Agent 시대에는 Git이 개발자와 Remote Worker 사이의 작업 전달 프로토콜 역할까지 수행한다.

Cloud Task 상태 후보:

- Task ID
- Cloud Session ID
- Branch
- Commit SHA
- Status
- Test Result
- PR

예:

```text
Task: 142
Session: cloud-agent-142
Branch: agent/task-142
Status: testing
Commit: abc123
PR: pending
```

---

# 6. 독립 Branch가 병렬성의 기반이다

```text
Repository
|
+-- Agent A → branch/feature-a
+-- Agent B → branch/test-auth
+-- Agent C → branch/fix-ci
+-- Agent D → branch/refactor-user
```

장점:

- Local workspace와 격리
- Agent 간 Git index 충돌 감소
- Task별 상태 추적
- 독립 PR 생성

단 같은 file/schema를 수정하는 Task는 병렬화하지 않는다.

---

# 7. 결과가 아니라 Evidence를 반환한다

Cloud Agent의 완료 메시지는 검증 가능한 결과물과 함께 와야 한다.

좋지 않은 결과:

```text
구현 완료했습니다.
```

권장 결과:

```text
Task Result

Implementation: DONE
Build: PASS
Unit Tests: 314 / 314 PASS
Integration Tests: 42 / 42 PASS
E2E: PASS

Artifacts:
- junit.xml
- screenshot
- video
- build.log

Commit: abc123
PR: #142
```

핵심:

> Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

Evidence 후보:

- Commit / Diff
- Unit Test Result
- Integration Test Result
- Build Result
- Screenshot
- Browser Video
- E2E Result
- Log Reference
- PR

## Demos over Diffs

특히 UI 변경은 빠른 1차 검증에서 실행 결과를 먼저 볼 수 있다.

```text
Build PASS
Test PASS
Before Screenshot
After Screenshot
E2E Video
       ↓
필요한 Diff 확인
```

> Cloud Agent의 결과는 Diff보다 Demo가 먼저일 수 있다.

이는 코드 리뷰를 생략한다는 의미가 아니다. 실행환경이 이미 존재한다는 장점을 Review에 사용하는 방법이다.

---

# 8. Multi-Repository는 필요한 범위만 제공한다

실제 시스템은 여러 Repository로 나뉠 수 있다.

```text
campus-api
campus-admin
campus-common
```

학생 프로필 API 변경처럼 여러 Repository 수정이 필요한 Task라면 하나의 Cloud Workspace에 관련 Repository를 제공할 수 있다.

하지만 모든 Repository를 항상 연결하지 않는다.

원칙:

> 현재 Task에서 함께 수정될 가능성이 높은 Repository만 제공한다.

이유:

- Context 증가
- 탐색량 증가
- Token 증가
- 잘못된 변경 범위 증가

---

# 9. Cloud Task Queue와 Event-driven 실행

Cloud Agent의 시작점이 개발자 PC일 필요는 없다.

Task Source:

- Issue
- CI Failure
- PR Review
- Scheduled Test
- Dependency Update
- Nightly Build

```text
Task Queue
├─ Issue #142
├─ CI Failure #91
├─ PR Review #52
└─ Nightly Test
        ↓
Cloud Worker Pool
├─ Agent A
├─ Agent B
└─ Agent C
```

이벤트 기반 예:

```text
Git Push
→ CI
   ├─ PASS → 종료
   └─ FAIL → Cloud Agent
                ↓
             분석/수정
                ↓
             재검증/PR
```

또는:

```text
PR Review Comment → Cloud Agent
Nightly Failure   → Cloud Agent
Dependency Update → Runner → FAIL일 때 Cloud Agent
```

핵심:

> 이벤트가 없으면 Agent도 실행하지 않는다.

이 패턴은 Cloud Agent 사용량과 Token을 줄이는 방법으로 다룬다.

---

# 10. Execution Time과 Developer Blocking Time을 분리한다

Cloud Agent가 40분 동안 실행됐다고 개발자가 40분 동안 기다린 것은 아니다.

```text
10:00 Cloud Task 위임
10:01 Developer는 다음 Feature 진행
10:40 Cloud Task 완료
11:20 Developer Review
```

측정 지표를 구분한다.

- Agent Execution Time
- Developer Blocking Time

Cloud Agent의 생산성은 단순 실행시간뿐 아니라 개발자가 대기하지 않고 다른 일을 할 수 있었는지도 함께 본다.

---

# 11. Cloud Agent에 적합한 Task 특성

Cloud에 적합할수록 다음 특성이 강하다.

- Scope 명확
- 완료 조건 정의 가능
- 독립 검증 가능
- 다른 작업과 파일 충돌 적음
- 지속적인 Human Steering 불필요
- 오래 걸리는 Build/Test 포함
- Git을 통해 결과 회수 가능

적합 예:

- AuthService Unit Test 추가
- 특정 API Refactoring
- Docker Build 검증
- E2E Test
- Lint 수정
- Dependency Update
- CI Failure 수정

부적합 예:

- 프로젝트 전체 구조 개선
- 원인 불명의 간헐적 성능 문제 탐색
- 새 Architecture 결정

후자는 Local에서 사람과 Agent가 빠르게 상호작용하는 편이 적합할 수 있다.

---

# 12. Cloud Agent Task Contract

Cloud Task는 Prompt를 길게 만드는 대신 탐색 범위를 줄이는 계약으로 만든다.

```text
Task
AuthService expired token 처리 수정

Goal
Expired JWT 사용 시 HTTP 401 반환

Scope
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Validation
./gradlew test --tests AuthServiceTest

Expected
expired token → 401

Do Not Change
- DB Schema
- OAuth 전체 구조
- 공통 Exception format

Output
- commit
- test result
- changed files
- short summary
```

Task Contract 목적:

- 작은 Scope
- 작은 Context
- 명확한 Validation
- 변경 금지 범위
- 검증 가능한 Output

---

# 13. Task 크기에는 최적 구간이 있다

너무 작은 Task:

- Environment start overhead
- Repository checkout overhead
- Agent startup
- Context loading 비용 비중 증가

너무 큰 Task:

- Context 증가
- Token 증가
- 실패 범위 증가
- Review 어려움
- Retry 비용 증가

```text
너무 작음
→ Cloud overhead가 상대적으로 큼

적당함
→ Cloud 효율이 높음

너무 큼
→ Context / Retry / Review 비용 증가
```

절대적인 분 단위 기준을 제시하지 않는다. 프로젝트별로 측정해서 Task 크기를 정한다.

---

# 14. 병렬화에도 상한이 있다

Cloud Agent 수를 늘리는 것이 선형적인 생산성 증가를 보장하지 않는다.

증가 비용:

- Repository Context 중복
- Dependency setup 중복
- Merge Conflict
- Review 증가
- LLM usage 증가
- PR 관리 증가

```text
Parallelizable Task
→ Cloud 병렬 실행

Dependent Task
→ 순차 실행
```

좋은 병렬화:

- Backend Unit Test
- Frontend E2E
- Docker Build
- Documentation

나쁜 병렬화:

```text
Agent A → UserService 구조 변경
Agent B → UserService 구조 변경
Agent C → UserService 구조 변경
```

---

# 15. 책의 권장 Hybrid Workflow

```text
                Developer / Local Agent
                         |
                    Architecture
                         |
                     Task Split
                         |
                         v
                        Git
                         |
          +--------------+--------------+
          |              |              |
      Cloud #1       Cloud #2       Cloud #3
        Test           Build            Fix
          |              |              |
          +--------------+--------------+
                         |
                  Evidence Result
                         |
                         v
                        PR
                         |
                         v
                Developer / Local Agent
                         |
                       Review
                         |
                       Merge
```

책의 Local + Cloud Hybrid Development 기본 구조로 사용한다.

---

# 16. 이 설계가 다른 장에 들어가는 위치

- 1장: Remote Development Worker 정의
- 3장: CPU/RAM/Disk와 Token 분리
- 4장: Cold Start, 장시간 비동기 작업, Developer Blocking Time
- 5장: Cloud 적합 Task 조건, Task 크기
- 7장: Cloud Agent Task Contract
- 8장: Evidence Result와 Result Gateway 연결
- 9장: Prepared Cloud Environment, Environment as Code, Cache/Fresh State, 작업별 환경
- 11장: Git Handoff Boundary, Branch/Session/Task 상태
- 12장: 병렬화 상한, 중복 Context
- 13장: Local→Cloud→Local Handoff, Multi-Repository 범위
- 14장: Task Queue, Event-driven Cloud Agent
- 15~16장: Evidence-based Result, Demos over Diffs, 전체 Hybrid Workflow
- 17장: Cloud에 적합하지 않은 Task 재확인

# 범위 경계

이 문서는 다음으로 확장하지 않는다.

- Agent Memory Architecture
- Agent Security Platform
- Agent OS
- Agent Chaos Engineering
- Garbage Collector Agent
- General Multi-Agent Theory
- Agent Governance

모든 개념은 다음 질문을 통과해야 한다.

> 이 내용이 개발자가 Cloud Agent를 더 잘 사용하는 데 직접적인 도움이 되는가?
