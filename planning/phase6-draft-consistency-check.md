# Phase 6 초고 정합성 점검

## 점검 목적

Phase 6에서 작성한 1~18장 초고가 `planning/concept.md`, `planning/scope.md`, `planning/toc.md`, 각 장의 `plan.md`와 일치하는지 확인한다.

점검 대상:

- `chapters/01/draft.md` ~ `chapters/18/draft.md`
- `planning/concept.md`
- `planning/scope.md`
- `planning/toc.md`
- 각 `chapters/NN/plan.md`
- `planning/future-topics.md`

Phase 6의 목적은 출판용 최종 교정이 아니라 **18개 장의 초고를 하나의 Cloud Agent 책으로 연결하는 것**이다.

---

# 1. 결과

Phase 6 초고 작성 완료.

```text
1~18장
plan.md 존재
+
draft.md 존재
```

전체 흐름은 다음과 같이 유지된다.

```text
Cloud Agent 정의
→ Local / Cloud 차이
→ Compute / Token 분리
→ Cloud의 실행 가치
→ Task Routing
→ 실제 Task 분류
→ Small Input / Context
→ Small Output / Evidence
→ Prepared Environment
→ Runner-first
→ Source / Runtime 격리
→ 병렬 Fan-out / Fan-in
→ Local ↔ Cloud Handoff
→ Event-driven Task
→ campus-platform 통합 운영 모델
→ 하나의 기능 End-to-End
→ Cloud를 쓰지 않을 조건
→ Harness / Orchestration 미래 방향
```

현재 목차의 중심 질문과 초고의 결론이 일치한다.

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

---

# 2. 중심 정의 점검

책 전체의 Cloud Agent 정의는 다음으로 유지된다.

```text
Cloud Agent
= LLM
+ Repository
+ Independent Execution Environment
+ CPU / RAM / Disk
+ Development Tools
```

확장 정의:

> Cloud Agent는 필요할 때 독립된 개발환경을 할당받고, Git을 통해 Task를 받아 비동기적으로 작업하며, 테스트와 Artifact를 포함한 검증 가능한 결과를 반환하는 Remote Development Worker다.

1장에서는 이 정의를 만들고, 뒤 장에서는 반복해서 새 정의를 추가하지 않고 실행 관점으로 확장한다.

---

# 3. 1장 제목 정합성

점검 과정에서 다음 불일치를 발견했다.

```text
planning/toc.md
Coding Agent에서 Remote Development Worker로

chapters/01/plan.md
Coding Agent에서 Cloud Worker로

chapters/01/draft.md
Coding Agent에서 Cloud Worker로

STATUS.md
Coding Agent에서 Cloud Worker로
```

본문과 장 설계가 이미 `Cloud Worker`를 장 제목으로 사용하고 있으며 `Remote Development Worker`는 핵심 정의로 사용하고 있다.

따라서 제목을 다음으로 통일했다.

```text
1장. Coding Agent에서 Cloud Worker로
```

정의는 그대로 유지한다.

```text
Cloud Agent = Remote Development Worker
```

---

# 4. 장별 역할 경계

## 1~4장 - 정의와 가치

### 1장

Cloud Agent를 Remote Development Worker로 정의한다.

### 2장

Local Agent와 Cloud Agent의 실행 위치 차이를 설명한다.

### 3장

Compute와 LLM Token을 분리한다.

### 4장

독립 실행환경, 장시간 작업, 비동기성, 병렬성의 가치를 설명한다.

판정:

```text
역할 충돌 없음
```

3장은 자원 구조, 4장은 개발 Workflow 효과에 집중한다.

---

## 5~6장 - 어떤 작업을 어디에 둘 것인가

### 5장

Task Routing Framework를 제공한다.

```text
Local / Cloud / Hybrid / Runner-first
```

### 6장

실제 개발 작업 유형을 Routing Framework에 적용한다.

```text
Build
Unit / Integration / E2E
Docker
Migration
Bug Fix
Refactoring
Documentation
PR Review
Dependency Update
CI Failure
```

판정:

```text
5장 = 판단 기준
6장 = 작업 Catalog
```

역할이 구분된다.

---

## 7~8장 - 작은 입력과 작은 출력

### 7장

```text
Task Contract
→ Small Input / Context
```

### 8장

```text
Result Gateway / Evidence
→ Small Output / Tool Result
```

두 장은 의도적으로 대칭 구조다.

판정:

```text
유지
```

---

## 9~10장 - 실행 준비와 실행 주체

### 9장

Prepared Environment, Cache, Snapshot으로 Cold Start를 줄인다.

### 10장

Prepared Environment에서 Deterministic Runner를 먼저 실행하고 필요한 실패에만 Agent를 호출한다.

판정:

```text
Environment 준비
→ Runner-first 실행
```

연결이 명확하다.

---

## 11~12장 - 격리와 병렬화

### 11장

Task별 Source / Runtime / Artifact를 격리한다.

```text
Branch
→ Source Isolation

Container / VM
→ Runtime Isolation
```

### 12장

격리된 Task 중 실제 독립 Task만 Fan-out한다.

```text
Parallel Benefit
- Startup
- Duplicate Context
- Merge
- Review
- Coordination
```

판정:

```text
11장 = 병렬화의 안전 조건
12장 = 병렬화의 비용과 상한
```

역할이 구분된다.

---

## 13~14장 - Handoff와 Event

### 13장

사람이 Local에서 Cloud로 Task를 넘기고 Evidence를 다시 Local로 회수한다.

### 14장

CI, Review, Nightly, Dependency Update 같은 Event가 동일한 Cloud Task 흐름을 생성한다.

판정:

```text
13장 = Human-driven Handoff
14장 = Event-driven Handoff
```

역할이 구분된다.

---

## 15~16장 - 실전 통합

### 15장

`campus-platform` 전체 운영 모델을 설계한다.

```text
Local
+
Cloud Runner
+
Cloud Agent
+
Result Gateway
+
Internal Validation
```

### 16장

학생 출결 API 인증 변경 하나를 Requirement부터 Merge까지 시간 순서대로 따라간다.

판정:

```text
15장 = Architecture / Operating Model
16장 = Timeline / End-to-End Example
```

두 장의 반복은 의도적이며 역할이 다르다.

---

## 17장 - 역방향 Routing

5장이 `언제 Cloud로 보낼 것인가`를 정의한다면 17장은 다음을 다룬다.

```text
언제 Cloud를 사용하지 않을 것인가?
언제 Cloud Task를 중단할 것인가?
언제 Local로 Fallback할 것인가?
```

판정:

```text
5장과 중복 아님
정방향 Routing / 역방향 Routing 관계
```

---

## 18장 - 미래 전망

18장은 현재 책의 실행 Workflow가 안정화된 뒤 자연스럽게 나타나는 다음 주제만 소개한다.

- Harness Engineering
- Routing Automation
- Orchestration
- Brain / Hands 후속 모델
- Queryable Evidence
- Compute-aware Scheduling
- Warm Worker
- Best-of-N
- Replay

다음 주제는 독립 본문으로 확장하지 않는다.

- Agent Memory Architecture
- Agent Security Platform
- Agent OS
- Agent Chaos Engineering
- Garbage Collector Agent
- Shadow Agent
- Canary Agent
- Agent Governance Platform
- General Multi-Agent Theory

판정:

```text
Agent Platform 일반론으로 재확장되지 않음
```

---

# 5. 핵심 용어 정합성

## Cloud Agent

```text
Remote Development Worker
```

## Cloud Runner

```text
LLM 없이 결정론적 명령을 실행하는 Cloud 실행 노드
```

## Task Contract

```text
Agent의 사고 과정을 작성하는 Prompt가 아니라
탐색 / 변경 / 검증 경계를 정의하는 작업 입력
```

## Result Gateway

```text
Raw Artifact를 보존하고
Agent/Developer에게 필요한 결과만 단계적으로 제공하는 결과 경계
```

## Evidence

```text
Commit
Changed Files
Build/Test Result
Artifact
Screenshot / Video / Trace
PR
```

## Handoff Boundary

```text
Local → Cloud
Git / Base SHA / Branch / Task Contract

Cloud → Local
Commit / Evidence / PR
```

## Local Fallback

```text
Cloud 실패가 아니라 Routing 재분류
```

용어 간 충돌 없음.

---

# 6. 의도적으로 반복되는 핵심 문장

다음 메시지는 여러 장에서 반복된다.

```text
Cloud Agent는 Local Agent를 대체하지 않는다.

Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.

CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.

Runner가 할 수 있으면 Runner에게 맡긴다.

작은 Task와 작은 Context를 전달한다.

Agent에게 개발환경을 설치하게 하지 말고 바로 작업 가능한 환경을 제공한다.

Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.

병렬화의 대상은 Agent가 아니라 독립 Task다.
```

Phase 6에서는 책의 논리를 연결하기 위해 의도적으로 반복했다.

다음 편집 단계에서는 같은 문장이 연속 장에서 과도하게 반복되는 부분을 줄이되, 각 Part의 기준 문장으로 필요한 반복은 유지한다.

---

# 7. 반복 예제 점검

책의 대표 반복 예제:

```text
AuthService expired token
AUTH-142
expected 401 / actual 200
```

이 예제는 다음 흐름을 같은 문제로 연결하는 역할을 한다.

```text
Routing
→ Task Contract
→ Failure Summary
→ Agent Fix
→ Runner Revalidation
→ Evidence
→ Handoff
```

따라서 반복 자체는 유지할 가치가 있다.

다만 편집 단계에서는 모든 장에서 동일한 숫자와 문장을 다시 설명하는 부분을 줄이고 앞 장 참조로 대체할 수 있다.

---

# 8. 제품 의존성 점검

핵심 원칙은 특정 Cloud Agent 제품에 종속되지 않는다.

제품별 다음 값은 핵심 본문에서 일반 규칙으로 고정하지 않는다.

```text
CPU / RAM
Disk
동시 Session 수
가격
Rate Limit
제품별 UI
```

변경 가능한 사실은 `research/` 문서에서 기준일과 출처를 관리한다.

판정:

```text
유지
```

---

# 9. campus-platform 예제 범위

`campus-platform`은 실제 대학 내부 시스템 전체를 공개하는 예제가 아니라 책의 원칙을 연결하기 위한 축약 프로젝트다.

반복 기술 요소:

```text
Java 21
Spring Boot
Gradle
MyBatis
PostgreSQL / Testcontainers
Docker
Playwright
```

내부 환경 예:

```text
Tibero
HSM
Internal Jenkins
Internal API
```

내부망 값, Secret, 실제 운영 설정은 본문에 넣지 않는다.

판정:

```text
범위 적절
```

---

# 10. Phase 7 편집 대상

Phase 6 초고 자체의 구조적 충돌은 없지만 출판용 편집에서는 다음을 우선 점검한다.

## 10.1 반복 압축

특히 다음 장 사이의 반복을 줄인다.

```text
2 ↔ 5 ↔ 17
3 ↔ 8 ↔ 10
4 ↔ 12
7 ↔ 16
13 ↔ 15 ↔ 16
```

개념 설명은 최초 장에 두고 후반 장에서는 적용 중심으로 줄인다.

## 10.2 예제 숫자 통일

설명용 Test Count, 시간, Retry 횟수는 모두 예시임을 명확히 하고 서로 모순되는 값이 실제 시스템 수치처럼 보이지 않게 한다.

## 10.3 한글/영문 용어 스타일

다음 표기를 전 장에서 통일한다.

```text
Task
Cloud Agent
Cloud Runner
Local Agent
Evidence
Artifact
Result Gateway
Task Contract
Prepared Environment
Human Steering
Developer Blocking Time
```

## 10.4 장간 참조 점검

```text
7장 → 8장
8장 → 10장
9장 → 10장
10장 → 11장
12장 → 13장
13장 → 14장
15장 → 16장
17장 → 18장
```

잘못된 과거 장 번호 참조가 없는지 최종 교정한다.

## 10.5 제품 사례 근거 점검

제품명을 직접 언급한 문장은 출판 직전 기준일과 공식 출처를 다시 확인한다.

---

# 11. Phase 6 완료 판정

다음 조건을 충족했다.

```text
[완료] 1~18장 설계 존재
[완료] 1~18장 초고 존재
[완료] Cloud Agent 중심 범위 유지
[완료] Local / Cloud / Hybrid Routing 유지
[완료] Runner-first 유지
[완료] Task Contract / Result Gateway 연결
[완료] Prepared Environment / Isolation / Parallelism 연결
[완료] Git Handoff / Evidence Return 연결
[완료] Event-driven Workflow 연결
[완료] campus-platform 운영 모델 및 End-to-End 예제 작성
[완료] Cloud를 쓰지 않을 조건 작성
[완료] 미래 주제를 18장 Advanced Topic 수준으로 제한
[완료] 1장 제목 정합성 수정
```

## 결론

Phase 6 본문 초고 작성과 구조 정합성 점검을 완료한다.

다음 단계에서는 새 개념을 추가하기보다 **중복 제거, 문장 교정, 사례/수치 정리, 장간 연결 강화, 제품 사례 출처 재검증**을 우선한다.
