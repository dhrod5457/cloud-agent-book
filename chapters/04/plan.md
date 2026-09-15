# 4장 설계 - 독립 실행환경, 장시간 작업, 병렬성, 시간 분리

## 장의 목표

Cloud Agent를 사용하는 이유를 `더 많은 AI 추론`이 아니라 다음 네 가지 실행 특성에서 설명한다.

- 개발자 PC와 분리된 독립 실행환경
- 장시간 작업의 비동기 위임
- 서로 독립적인 작업의 병렬 실행
- Agent 실행시간과 개발자 대기시간의 분리

이 장의 핵심 질문은 다음과 같다.

> 같은 Agent 작업이라도 왜 Local보다 Cloud에서 실행하는 것이 더 유리한 경우가 있는가?

1~3장에서 Cloud Agent를 Remote Development Worker로 정의하고 CPU/RAM/Disk와 Token을 분리했다면, 4장에서는 그 독립 실행환경을 실제 개발 생산성에 어떻게 활용하는지 설명한다.

---

## 핵심 주장

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

그리고 다음 관점을 추가한다.

> Cloud Agent의 작업시간과 개발자가 기다리는 시간은 같은 값이 아니다.

Cloud Task가 40분 걸려도 개발자가 40분 동안 멈춰 있을 필요는 없다.

Cloud Agent가 별도 환경에서 작업하는 동안 개발자는 다음 작업을 계속할 수 있다.

---

## 독자가 얻는 것

- Cloud Agent의 독립 실행환경이 Local Agent와 어떤 차이를 만드는지 설명할 수 있다.
- 로컬 CPU/RAM을 점유하지 않고 장시간 Build/Test를 위임할 수 있다.
- `Agent Execution Time`과 `Developer Blocking Time`을 구분할 수 있다.
- 독립 작업을 여러 Cloud Worker로 병렬 분산할 수 있다.
- 병렬화 가능한 Task와 의존성이 있는 Task를 구분할 수 있다.
- Cloud Agent가 느려도 개발자 전체 Workflow는 빨라질 수 있는 조건을 설명할 수 있다.
- Cloud 병렬화가 무조건 생산성을 선형 증가시키지 않는 이유를 이해한다.

---

# 절 구성

## 4.1 독립 실행환경이 Cloud Agent의 첫 번째 가치다

Local Agent는 개발자의 현재 Workspace와 컴퓨팅 자원을 공유하는 경우가 많다.

```text
Developer Mac
├─ IDE
├─ Local Agent
├─ Gradle Build
├─ Docker
├─ Browser
└─ Testcontainers
```

Build/Test가 길어질수록 개발자 환경도 영향을 받는다.

Cloud에서는 Task를 별도 실행환경으로 분리할 수 있다.

```text
Developer
   |
   +-- Local Workspace
   |
   +-- Cloud Worker
        ├─ Repository
        ├─ CPU
        ├─ RAM
        ├─ Disk
        └─ Development Tools
```

Cloud Worker는 Local Workspace와 별개로 다음 작업을 수행할 수 있다.

- Gradle build
- 전체 unit test
- integration test
- Testcontainers
- Docker build
- E2E
- static analysis

이 장에서는 Cloud 환경이 항상 더 빠르다고 주장하지 않는다.

핵심은 `작업 환경을 개발자 환경과 분리할 수 있다`는 점이다.

---

## 4.2 Cloud Agent를 Remote Worker로 본다

1장의 정의를 실제 작업 관점으로 확장한다.

```text
Cloud Worker
= Task
+ Repository State
+ Independent Runtime
+ Compute
+ Tools
+ Result
```

Developer가 Cloud Worker에게 넘기는 것은 단순 Prompt가 아니다.

최소 작업 단위에는 다음이 포함된다.

- Task
- Git commit/branch 기준점
- 필요한 Context
- 실행 명령
- 완료 조건
- 반환할 Evidence

Task Contract의 상세 형식은 7장에서 다룬다.

이 장에서는 Cloud Worker가 `별도 개발자 한 명`이라기보다 `독립 실행환경을 가진 원격 작업 노드`라는 관점을 확정한다.

---

## 4.3 장시간 작업을 Local에서 분리한다

Cloud Agent에 적합한 대표적인 작업은 시간이 오래 걸리지만 작업 중 사람의 지속적인 개입이 필요하지 않은 작업이다.

예:

- 전체 Test Suite
- Docker Image Build
- Spring Boot Integration Test
- Testcontainers 기반 DB 검증
- Frontend E2E
- 대규모 Static Analysis
- Migration Validation
- 반복 Refactoring 후 전체 검증

Local에서 실행하면 다음 상황이 생길 수 있다.

```text
Developer
→ 전체 테스트 시작
→ 로컬 CPU/RAM 사용 증가
→ 결과 대기
→ 다음 작업 전환 어려움
```

Cloud에서는 다음처럼 분리할 수 있다.

```text
Developer
→ Cloud Task 위임
→ 다음 Feature 작업

Cloud Worker
→ Build / Test
→ Result 생성
```

중요한 판단 기준은 `작업시간이 길다` 하나가 아니다.

다음 조건을 함께 본다.

- Scope가 명확한가
- 중간 Human Steering이 필요한가
- 독립적으로 검증 가능한가
- Git을 통해 결과를 돌려받을 수 있는가

Task Routing 상세 기준은 5장에서 다룬다.

---

## 4.4 Agent Execution Time과 Developer Blocking Time을 분리한다

Cloud Agent의 생산성을 단순히 `몇 분 만에 끝냈는가`로 평가하지 않는다.

예:

```text
10:00
Cloud Agent에 테스트 작성 위임

10:01
Developer는 다른 Feature 구현 시작

10:40
Cloud Task 완료

11:20
Developer가 결과 Review
```

측정값을 분리한다.

```text
Agent Execution Time
= Cloud Task가 시작해서 끝날 때까지 걸린 시간

Developer Blocking Time
= 개발자가 해당 Task 때문에 실제로 멈춰 있던 시간
```

Cloud Task가 40분 걸려도 Developer Blocking Time은 몇 분일 수 있다.

따라서 Cloud Agent 도입 효과를 평가할 때 다음을 함께 본다.

- Agent Execution Time
- Queue Time
- Developer Blocking Time
- Review Time
- Retry Time
- Local Resource Occupancy

핵심 메시지:

> Cloud Agent의 생산성은 Agent가 얼마나 빨리 끝나는지만이 아니라 개발자가 얼마나 덜 기다리는지로도 평가해야 한다.

---

## 4.5 비동기 위임이 가능한 Task와 불가능한 Task

### 비동기 위임에 적합

```text
Task가 명확함
+ 완료 조건 존재
+ 중간 질문이 적음
+ 독립 검증 가능
+ 결과를 Git/Artifact로 반환 가능
```

예:

- 특정 Service 테스트 추가
- Module 전체 테스트
- Docker Build
- E2E Regression
- Dependency Update 검증

### 비동기 위임에 부적합

```text
요구사항 자체가 불명확함
+ 개발자 판단을 계속 요구함
+ 매우 큰 Context 필요
+ Local Runtime 상태에 의존
```

예:

- 새 시스템 Architecture 방향 탐색
- 간헐적 Production 장애의 원인 탐색
- 요구사항이 계속 바뀌는 UI 설계

이 구분은 5장 Task Routing의 기반이 된다.

---

## 4.6 병렬 실행의 기본 형태

독립 Task가 여러 개라면 하나의 Agent가 순차 실행할 이유가 없다.

```text
Developer / Local Agent
          |
       Task Split
          |
  +-------+-------+-------+
  |       |       |       |
Cloud A Cloud B Cloud C Cloud D
  |       |       |       |
Unit    Integ.   E2E    Docker
Test     Test    Test    Build
  |       |       |       |
  +-------+-------+-------+
          |
        Result
```

이 구조의 목적은 Agent 수 자체를 늘리는 것이 아니다.

다음과 같이 서로 다른 컴퓨팅 작업을 동시에 처리하는 것이 목적이다.

- CPU-heavy test
- browser-based E2E
- container build
- migration validation

3장의 `Parallel Compute` 개념을 실제 Workflow에 적용한다.

---

## 4.7 좋은 병렬화와 나쁜 병렬화

### 좋은 병렬화

```text
Cloud #1 → Backend Unit Test
Cloud #2 → Integration Test
Cloud #3 → Frontend E2E
Cloud #4 → Docker Build
```

서로 변경 범위가 겹치지 않거나 read-only 검증이다.

또 다른 예:

```text
Service A Java 21 migration
Service B Java 21 migration
Service C Java 21 migration
```

각 서비스가 독립 Repository 또는 독립 Module이고 별도 검증이 가능하다면 병렬화하기 좋다.

### 나쁜 병렬화

```text
Agent A → UserService 구조 변경
Agent B → UserService 구조 변경
Agent C → UserService 구조 변경
```

또는:

```text
Agent A → schema migration 수정
Agent B → 같은 schema migration 수정
```

병렬 실행 자체는 가능하지만 Merge Conflict와 Review Cost가 커질 수 있다.

11장에서는 Branch/Worktree/Container 격리를, 12장에서는 병렬 Worker와 Context 중복 비용을 상세히 다룬다.

---

## 4.8 병렬화의 효과는 선형이 아니다

Cloud Worker를 10개 실행한다고 생산성이 정확히 10배가 되지는 않는다.

병렬도가 높아질수록 다음 비용이 증가한다.

- Repository checkout
- dependency/environment 준비
- 같은 Context의 중복 분석
- LLM usage
- Merge Conflict
- PR 수
- Review 부담
- 결과 통합 비용

따라서 다음 구조로 판단한다.

```text
Independent Task
→ Parallelize

Dependent Task
→ Sequence
```

그리고 병렬화 전에 확인한다.

- 파일 Scope가 분리되는가
- 완료 조건이 독립적인가
- 결과를 개별적으로 검증 가능한가
- Merge 순서 의존성이 없는가

12장에서 비용 모델을 더 상세히 다룬다.

---

## 4.9 Local 컴퓨팅 자원을 비워 두는 것도 가치다

개발자가 Mac에서 다음 작업을 동시에 수행한다고 가정한다.

```text
IDE
Local Agent
Docker
Database
Browser
Android Emulator
```

여기에 전체 Gradle Test, Testcontainers, Docker Build까지 추가하면 로컬 환경이 개발 병목이 될 수 있다.

Cloud Worker에 Build/Test를 넘기면 Local에서는 다음에 집중할 수 있다.

- 코드 탐색
- Architecture 판단
- 빠른 수정 반복
- 내부망 작업
- 최종 Review

Cloud Agent의 효율을 Token 비용만으로 평가하지 않고 Local Resource Occupancy 감소도 함께 본다.

---

## 4.10 campus-platform 병렬 작업 예제

`campus-platform`에서 하나의 기능 변경 후 다음 검증을 수행한다고 가정한다.

```text
Local Developer / Agent
→ 기능 구현
→ Commit / Push
      |
      +-- Cloud #1
      |    ./gradlew test
      |
      +-- Cloud #2
      |    Integration Test
      |
      +-- Cloud #3
      |    Web E2E
      |
      +-- Cloud #4
           Docker Build
```

Developer는 Cloud 검증이 진행되는 동안 다른 작업을 계속한다.

Cloud Worker들은 최종적으로 Evidence를 반환한다.

```text
Cloud #1 → unit-test result
Cloud #2 → integration result
Cloud #3 → screenshot / video / E2E result
Cloud #4 → image build result
```

Evidence의 상세 형식은 8장에서 다룬다.

---

# 좋은 사례와 나쁜 사례

## 사례 A - 전체 테스트

좋지 않은 방식:

```text
Developer Laptop
→ 전체 Test 실행
→ CPU/RAM 점유
→ Developer 대기
```

권장 방식:

```text
Cloud Worker
→ 전체 Test 실행

Developer
→ 다른 작업 계속
```

## 사례 B - 독립 검증 병렬화

좋지 않은 방식:

```text
한 Worker
→ Unit
→ Integration
→ E2E
→ Docker Build
```

독립 실행이 가능한데도 모두 순차 수행한다.

권장 방식:

```text
Unit / Integration / E2E / Docker Build
→ 각각 독립 Worker
→ 병렬 실행
→ 결과 통합
```

## 사례 C - 의존 Task의 무리한 병렬화

좋지 않은 방식:

```text
Agent A → 공통 DTO 변경
Agent B → 같은 DTO 기반 API 변경
Agent C → 같은 DTO 기반 UI 변경
```

의존관계를 고려하지 않고 동시에 수정한다.

권장 방식:

```text
공통 Contract 확정
→ commit
→ dependent tasks fan-out
```

---

# 필요한 그림

1. Local Workspace vs Independent Cloud Worker
2. Long-running Task의 비동기 위임
3. Agent Execution Time vs Developer Blocking Time
4. 여러 Cloud Worker 병렬 실행
5. Parallelizable Task vs Dependent Task
6. `campus-platform` Unit/Integration/E2E/Docker 병렬 검증

---

# 필요한 측정 항목

실전 프로젝트에서는 다음 값을 측정할 수 있게 한다.

```text
Task ID
Agent Execution Time
Queue Time
Developer Blocking Time
Review Time
Cloud Worker Count
Result Status
Retry Count
```

Token/Compute Cost의 상세 최적화는 8~10장에서 다룬다.

---

# 다른 장과의 연결

## 3장

CPU/RAM/Disk, Token, Parallel Compute 개념을 가져온다.

## 5장

이 장에서 설명한 `Cloud가 유리한 조건`을 Task Routing 판단표로 구체화한다.

## 7장

병렬 Cloud Task에 전달할 Task/Context 범위를 정의한다.

## 8장

각 Worker가 반환할 Evidence와 로그 축약 방식을 다룬다.

## 9장

Cloud Worker Cold Start를 줄이는 Prepared Environment/Cache/Snapshot을 다룬다.

## 11장

병렬 Worker의 Branch/Worktree/Container 격리를 상세히 다룬다.

## 12장

병렬 Worker 증가 시 중복 Context와 통합 비용을 다룬다.

## 13장

Local과 Cloud 사이의 전체 Handoff Workflow로 연결한다.

---

# 본문에서 의도적으로 다루지 않을 내용

- 상세 Task Routing Matrix: 5장
- Task Contract 상세: 7장
- Result Gateway 구현: 8장
- Prebuilt Environment/Cache: 9장
- Runner-first: 10장
- Git Branch/Worktree 상세: 11장
- Best-of-N: 12장 Advanced Topic
- Agent Platform 일반론
- PM Agent 전체 Orchestration

---

# 장의 결론 메시지

4장은 다음 세 문장으로 정리한다.

> Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

> Cloud Agent가 오래 실행되는 것과 개발자가 오래 기다리는 것은 같은 의미가 아니다.

> 병렬화의 대상은 Agent가 아니라 서로 독립적으로 실행하고 검증할 수 있는 Task다.
