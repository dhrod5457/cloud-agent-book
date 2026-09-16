# 5장. Task Routing: Local인가 Cloud인가

Cloud Agent를 잘 사용하는 팀은 모든 작업을 Cloud로 보내지 않는다.

더 중요한 것은 `이 Task를 어디에서 실행하는 것이 유리한가`를 빠르게 판단하는 것이다.

이 장에서는 Task를 다음 네 경로로 분류한다.

```text
Task
  ↓
Task Classification
  ↓
Local / Cloud / Hybrid / Runner-first
```

4장에서 Cloud의 독립 실행환경과 비동기·병렬 실행 가치를 설명했다면, 이번 장에서는 그 가치를 **어떤 Task에 적용할지** 결정한다.

> Cloud Agent를 잘 사용하는 핵심은 Agent 수를 늘리는 것이 아니라 어떤 작업을 Cloud로 보낼지 결정하는 것이다.

> Cloud에 보낼 수 있는 Task와 Cloud에 보내는 것이 유리한 Task는 다르다.

17장에서는 이 판단을 반대로 적용해 Cloud Task를 중단하거나 Local로 되돌릴 조건을 다룬다. 이번 장의 역할은 **Task 시작 시점의 Routing Framework**를 만드는 것이다.

---

## 1. Routing은 모델 선택보다 실행 위치 선택에서 시작한다

Task를 받으면 `어떤 모델이 잘할까`보다 먼저 다음을 본다.

```text
이 작업은 어디에서 실행해야 하는가?
어떤 실행 주체가 필요한가?
```

예를 들어 신규 인증 Architecture를 설계하는 Task에는 다음이 필요할 수 있다.

```text
여러 모듈 탐색
요구사항 비교
Developer와 반복 대화
내부 시스템 제약 확인
```

이 작업은 Local에 가깝다.

반면 다음은 Cloud Runner에 가깝다.

```bash
./gradlew test
```

명령과 성공 조건이 정해져 있기 때문이다.

다음처럼 재현 가능한 작은 Bug는 Cloud Agent 후보가 된다.

```text
Failure
AuthServiceTest.expiredToken
expected: 401
actual: 200

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

Routing은 작업 이름이 아니라 **작업 상태와 완료 조건**을 보고 결정한다.

---

## 2. 첫 질문은 `Runner로 끝낼 수 있는가`다

Cloud에 보내는 Task라고 모두 Agent가 필요한 것은 아니다.

가장 먼저 결정론적으로 끝낼 수 있는지 확인한다.

```text
Task
  ↓
결정론적 실행으로 판정 가능한가?
  ├─ YES → Runner / Tool
  └─ NO  → Agent 또는 Human 판단 후보
```

Runner에 적합한 대표 작업:

```text
Build
Unit Test
Integration Test
E2E
Docker Build
Lint
Static Analysis
Migration Validation
```

이 작업은 명령과 PASS / FAIL 판정 기준이 명확하다.

```text
Runner
→ 실행
→ PASS / FAIL
```

PASS라면 종료한다.

FAIL이고 원인 분석이나 코드 수정이 필요할 때 Agent가 들어간다.

이 원칙은 6장에서 작업 유형별로 적용하고, 10장에서 Runner-first 실행 구조로 상세히 다룬다.

---

## 3. Hard Constraint를 먼저 확인한다

Cloud에 유리한 조건을 여러 개 합산하기 전에 **Cloud 실행 자체를 막는 조건**이 있는지 본다.

대표 Hard Constraint:

```text
Internal Network 필수
Repository / 데이터를 Cloud에 제공할 수 없음
현재 Local 상태를 그대로 사용해야 함
특정 장비 / 디바이스에 직접 접근해야 함
```

예:

```text
HSM 실제 장비 검증
→ Local

Tibero 운영환경에서만 재현되는 SQL 문제
→ Local 또는 Hybrid
```

다른 조건이 좋아도 Hard Constraint가 있으면 실행 위치가 먼저 제한된다.

```text
Hard Constraint
→ 실행 가능한 위치 결정
        ↓
그 안에서 비용 / 시간 최적화
```

보안·정책·네트워크 제약을 우회하는 것이 Routing의 목적은 아니다.

---

## 4. Local에 가까운 Task

다음 신호가 많을수록 Local이 유리하다.

```text
요구사항이 아직 불명확함
Human Steering이 잦음
큰 Context가 필요함
미커밋 Local 상태 의존
내부망 / 장비 의존
재현 절차가 불명확함
Architecture 판단 비중이 큼
```

예:

```text
새 인증 구조를 JWT로 유지할지 다른 구조로 바꿀지 검토
```

이 작업은 독립 실행보다 탐색과 의사결정에 가깝다.

```text
Developer + Local Agent
→ 탐색
→ 선택지 비교
→ 결정
→ Task Split
```

방향이 정해진 뒤 구현이나 검증을 작은 Cloud Task로 나눌 수 있다.

Local의 핵심 강점은 **Developer와 Agent 사이의 짧은 Feedback Loop**다.

---

## 5. Cloud에 가까운 Task

다음 신호가 많을수록 Cloud로 보내기 쉽다.

```text
Scope 명확
완료 조건 명확
Git으로 상태 전달 가능
독립 검증 가능
중간 질문 적음
장시간 실행
Compute 사용량 큼
다른 Task와 충돌 적음
Evidence로 결과 반환 가능
```

예:

```text
Task
attendance 모듈 Java 21 호환성 수정

Scope
attendance module only

Validation
./gradlew :attendance:test
```

또는:

```text
Task
전체 Web E2E 실행

Validation
npx playwright test
```

Cloud가 유리한 이유는 단순히 원격에서 실행되기 때문이 아니다.

Task를 독립적으로 위임하고, Developer는 다른 작업을 진행하며, 결과를 나중에 Evidence로 받을 수 있기 때문이다.

---

## 6. Hybrid는 단계별 Routing이다

기업 프로젝트에서는 Local과 Cloud 하나만으로 끝나지 않는 Task가 많다.

예를 들어 DB Migration을 수정한다고 하자.

```text
Local
→ 요구사항 / 실제 DB 제약 확인
        ↓
Cloud
→ Migration 작성 / 일반 검증
→ Testcontainers 기반 Test
        ↓
Local
→ Tibero 실제 적용 검증
```

이 경우 하나의 기능 안에서 실행 위치가 바뀐다.

> Task는 Local 또는 Cloud 중 하나에 영구적으로 속하지 않는다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

Hybrid는 예외 처리라기보다 **재현 가능한 부분은 Cloud로 보내고 실제 내부 경계는 Local에 남기는 방식**이다.

13장에서 이 Handoff를 전체 Workflow로 확장한다.

---

## 7. Human Steering과 Context 크기를 함께 본다

Cloud Agent는 비동기 위임에 유리하다. 그러나 작업 중 사람이 계속 방향을 바꿔야 한다면 이 장점은 줄어든다.

예:

```text
관리자 UI 개선
→ A안 구현
→ Developer 확인
→ 구조 변경
→ B안 구현
→ 다시 확인
```

이런 Task는 Local에 가깝다.

반대로 다음 Task는 중간 개입이 적다.

```text
Goal
Expired JWT → HTTP 401

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

Context도 같은 방식으로 본다.

작은 Context:

```text
AuthService.java
JwtTokenProvider.java
AuthServiceTest.java
```

큰 Context:

```text
Auth
Student
Permission
Database
Common Exception
External API
Admin Web
```

큰 문제는 Local에서 먼저 범위를 좁힌 뒤 작은 Task로 Cloud에 보내는 편이 낫다.

```text
큰 문제
→ Local 탐색
→ 결정
→ Task 분해
→ Cloud 위임
```

Context를 실제로 어떻게 작게 구성하는지는 7장에서 다룬다.

---

## 8. Git으로 기준 상태를 전달할 수 있는가

Cloud Worker는 Local IDE의 현재 상태를 자동으로 공유하지 않는다.

다음 상태에 강하게 의존하면 Handoff가 어렵다.

```text
미커밋 파일
untracked config
로컬 DB 임시 데이터
IDE 내부 상태
특정 프로세스 상태
```

Cloud Task는 가능한 한 다음처럼 명시적인 기준점에서 시작하는 편이 좋다.

```text
Repository
Base SHA
Branch
Task
Validation
```

예:

```text
repository: campus-platform
base_sha: abc123
branch: agent/auth-expired-token
```

Git과 Branch를 실제 Handoff Boundary로 사용하는 방법은 11장과 13장에서 구체화한다.

이번 장에서의 판단은 단순하다.

> 현재 작업 상태를 다른 실행환경으로 명확히 전달할 수 있는가?

---

## 9. 독립 검증과 변경 충돌을 본다

Cloud Task는 독립 검증이 가능할수록 관리하기 쉽다.

좋은 예:

```text
Task A → attendance module test
Task B → notification module test
```

반대로 다음은 독립 Task라고 보기 어렵다.

```text
Task A → UserService 구조 변경
Task B → UserService 오류 처리 변경
Task C → UserService 기반 DTO 변경
```

각 Branch에서 성공하더라도 통합 시 충돌할 수 있다.

Routing 단계에서는 다음 정도만 확인한다.

```text
예상 변경 파일
shared / common module
DB Schema / Migration
공통 DTO / API
순서 의존성
```

Source / Runtime 격리는 11장에서, 병렬화 비용과 Fan-in은 12장에서 상세히 다룬다.

---

## 10. Task 크기는 Cloud Overhead와 함께 본다

Cloud에는 작업 내용 외의 준비 비용이 있다.

```text
Worker Start
Checkout
Environment 준비
Context Load
Commit / PR
Review
```

따라서 너무 작은 Task는 Cloud가 불리할 수 있다.

```text
문구 한 줄 수정
검증 수초
```

반대로 너무 큰 Task도 문제가 된다.

```text
프로젝트 전체 Architecture 개선
```

Task가 커지면 다음 비용이 함께 커진다.

```text
Context
변경 파일 수
Retry 범위
검증 범위
Review 부담
Merge Risk
```

절대적인 `몇 분 이하이면 Local` 같은 규칙은 두지 않는다.

프로젝트마다 Cold Start와 Review 비용이 다르기 때문이다.

적정 Task는 대체로 다음 특징을 가진다.

```text
독립 실행 가능
독립 검증 가능
Review 가능한 변경 범위
명확한 완료 조건
```

7장의 Task Contract가 이 범위를 고정하는 입력이 된다.

---

## 11. Routing Decision Matrix

다음 표는 시작점이다.

| 조건 | Local | Cloud | Hybrid |
| --- | --- | --- | --- |
| 요구사항 불명확 | 유리 | 불리 | 불리 |
| Human Steering 잦음 | 유리 | 불리 | 가능 |
| 큰 Context 필요 | 유리 | 불리 | 가능 |
| 장시간 Build / Test | 가능 | 유리 | 유리 |
| Compute 사용 큼 | 가능 | 유리 | 유리 |
| Internal Network 필수 | 유리 | 불리 | 유리 |
| Git Handoff 가능 | 가능 | 유리 | 유리 |
| 독립 검증 가능 | 가능 | 유리 | 유리 |
| 동일 파일 / DB Schema 충돌 큼 | 유리 | 불리 | 불리 |
| Architecture 판단 | 유리 | 불리 | 가능 |

이 표는 점수 합산으로 정답을 만드는 도구가 아니다.

Hard Constraint 하나가 다른 조건보다 우선할 수 있다.

---

## 12. Score는 보조 수단일 뿐이다

팀 내에서 빠르게 대화하기 위해 간단한 Score를 사용할 수 있다.

아래 점수는 설명용 예다.

```text
Scope 명확성             +2
독립 검증 가능            +2
장시간 실행               +1
Compute 사용 큼           +1
Human Steering 많음       -2
Internal Network 필요     -3
큰 Context 필요           -2
파일 충돌 가능성 높음     -2
```

점수만으로 자동 결정하지 않는다.

판단 순서는 다음이 낫다.

```text
1. Hard Constraint
2. Runner로 끝낼 수 있는가
3. Scope / 완료 조건
4. 독립 검증 가능성
5. Handoff 가능성
6. Human Steering / Context
7. 충돌 가능성
8. Cloud Overhead와 Task 가치
```

Score는 이 대화를 짧게 하기 위한 보조 도구다.

---

## 13. campus-platform 작업을 분류해보자

| Task | 기본 경로 | 이유 |
| --- | --- | --- |
| 신규 인증 Architecture | Local | 큰 Context, Human Steering |
| expired token 수정 | Cloud Agent 후보 | 작은 Scope, 재현 / 검증 가능 |
| 전체 Unit Test | Cloud Runner | 결정론적, Compute 중심 |
| Web E2E | Cloud Runner | Browser 기반 독립 검증 |
| Tibero Migration 실제 검증 | Local / Hybrid | 내부 DB 의존 |
| HSM 오류 분석 | Local | 내부 장비 / 네트워크 의존 |
| Docker Build | Cloud Runner | 결정론적 Build |
| Dependency Update | Runner-first | PASS면 Agent 불필요 |
| README 한 줄 수정 | Local 후보 | Cloud Overhead가 상대적으로 큼 |

프로젝트의 실행환경이 바뀌면 Routing 결과도 바뀔 수 있다.

그래서 작업 이름보다 기준을 유지한다.

---

## 14. Routing 판단 순서

실제 Task를 받으면 다음 순서로 본다.

```text
1. Hard Constraint가 있는가?
2. Runner로 끝낼 수 있는가?
3. Scope와 완료 조건이 명확한가?
4. 독립 검증 가능한가?
5. Git으로 상태를 전달할 수 있는가?
6. Human Steering이 많이 필요한가?
7. Context가 과도하게 큰가?
8. 파일 / DB Schema 충돌 가능성이 큰가?
9. Cloud Overhead보다 Task 가치가 큰가?
```

이 질문에 답하면 대부분의 작업은 `Local / Cloud / Hybrid / Runner-first` 중 하나로 좁혀진다.

Routing은 고정된 분류표가 아니라 현재 Task 상태에 대한 판단이다.

다음 장에서는 이 기준을 Build, Test, E2E, Docker, Migration, Refactoring, Bug Fix, Documentation, PR Review, Dependency Update, CI Failure 같은 실제 작업에 적용한다.