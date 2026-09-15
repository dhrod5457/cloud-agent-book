# 5장. Task Routing: Local인가 Cloud인가

Cloud Agent를 잘 사용하는 팀은 모든 작업을 Cloud로 보내지 않는다.

더 중요한 것은 `이 Task를 어디에서 실행하는 것이 유리한가`를 빠르게 판단하는 것이다.

같은 작업이라도 조건에 따라 실행 위치가 달라질 수 있다.

```text
Task
  ↓
Task Classification
  ↓
Local / Cloud / Hybrid / Runner-first
```

이 장의 목적은 제품별 기능을 비교하는 것이 아니다.

Task의 특성을 보고 실행 위치와 실행 주체를 선택하는 기준을 만드는 것이다.

> Cloud Agent를 잘 사용하는 핵심은 Agent 수를 늘리는 것이 아니라 어떤 작업을 Cloud로 보낼지 결정하는 것이다.

또 하나의 원칙을 함께 사용한다.

> Cloud에 보낼 수 있는 Task와 Cloud에 보내는 것이 유리한 Task는 다르다.

---

## 1. Routing은 모델 선택이 아니라 실행 위치 선택이다

Task Routing을 다음 질문으로 시작한다.

```text
이 작업은 어떤 모델이 잘하는가?
```

보다 먼저 확인할 것은 다음이다.

```text
이 작업은 어디에서 실행해야 하는가?
```

예를 들어 신규 인증 Architecture를 설계한다고 하자.

이 작업에는 다음이 필요할 수 있다.

- 기존 인증 구조 전체 이해
- 여러 모듈 비교
- 개발자와 반복 대화
- 요구사항 변경
- DB와 외부 시스템 영향 검토

이런 작업은 Local에서 Developer와 Agent가 짧은 주기로 대화하는 편이 자연스럽다.

반대로 전체 Unit Test는 다르다.

```text
./gradlew test
```

명령과 성공 조건이 명확하다.

이 경우 Cloud Runner에서 실행하고 결과만 받아도 된다.

또 다음처럼 실패가 명확한 작은 Bug Fix는 Cloud Agent 후보가 된다.

```text
Failure
AuthServiceTest.expiredToken
expected: 401
actual: 200

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

즉 Routing의 출발점은 작업의 성격이다.

---

## 2. 먼저 Runner로 끝낼 수 있는지 본다

Cloud에 보낸다고 해서 반드시 Cloud Agent를 호출할 필요는 없다.

가장 먼저 확인할 것은 결정론적 실행으로 끝낼 수 있는가다.

```text
Task
  ↓
Deterministic?
  ├─ YES → Runner
  └─ NO  → Agent 후보
```

예를 들면 다음 작업은 기본적으로 Runner가 처리한다.

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

이 작업에서 필요한 것은 LLM의 판단이 아니라 실행과 판정이다.

```text
Runner
→ 명령 실행
→ exit code
→ PASS / FAIL
```

PASS라면 Agent를 호출할 이유가 없다.

FAIL이고 원인 분석이나 코드 수정이 필요할 때 Agent를 호출한다.

따라서 Routing 순서는 다음처럼 잡을 수 있다.

```text
1. Runner로 해결 가능한가?
2. Cloud에 독립 실행 가능한가?
3. Human Steering이 많이 필요한가?
4. Internal Network가 필요한가?
```

이 순서만 적용해도 불필요한 Agent 호출을 많이 줄일 수 있다.

---

## 3. Local에 남겨야 하는 신호

다음 조건이 많을수록 Local이 유리하다.

```text
요구사항이 불명확함
Human Steering이 잦음
큰 Context 필요
현재 미커밋 상태 의존
VPN / 내부망 필요
사내 DB / HSM 필요
Production 환경에서만 재현
Architecture 판단 비중 큼
수정 범위가 너무 작음
```

예를 들어 다음 Task를 보자.

```text
신규 학생 인증 구조를 전체적으로 다시 검토하고
JWT를 유지할지 OAuth 기반으로 갈지 정리해줘.
```

이 작업은 독립적인 실행 Task라기보다 탐색과 의사결정에 가깝다.

Local에서 Developer와 Agent가 여러 선택지를 비교하고 방향을 먼저 정하는 편이 낫다.

또 HSM이나 Tibero 운영 환경처럼 내부망이 필수라면 Cloud 사용 여부보다 Network Boundary가 먼저 결정한다.

```text
HSM 연동 오류
→ Local

Tibero 운영 환경에서만 재현되는 SQL 문제
→ Local
```

Cloud Agent가 기술적으로 코드를 읽을 수 있어도 실제 검증 환경에 접근하지 못하면 Task를 끝낼 수 없다.

---

## 4. Cloud에 보내기 좋은 신호

다음 조건이 많을수록 Cloud로 보내기 쉽다.

```text
Scope 명확
완료 조건 명확
Git으로 상태 전달 가능
독립 검증 가능
중간 질문 적음
장시간 실행
CPU/RAM 사용량 큼
파일 충돌 적음
Artifact로 결과 반환 가능
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

이런 Task는 입력과 출력이 명확하다.

Cloud Worker가 작업하는 동안 Developer는 다른 일을 계속할 수 있다.

---

## 5. Hybrid가 필요한 Task

실제 기업 프로젝트에서는 Local 또는 Cloud 하나만으로 끝나지 않는 Task가 많다.

예를 들어 DB Migration을 수정한다고 하자.

Cloud에서 다음까지는 가능할 수 있다.

```text
Migration 작성
→ PostgreSQL/Testcontainers 적용
→ 기본 Integration Test
```

하지만 최종 대상이 내부 Tibero라면 마지막 검증은 Local 또는 내부망에서 해야 한다.

```text
Local
→ 요구사항 / 제약 확인
→ Task Split
      ↓
Cloud
→ 일반 Migration Validation
→ Build / Test
      ↓
Local
→ Tibero 실제 적용 검증
```

이 경우 Task의 위치는 하나가 아니다.

> Task는 Local 또는 Cloud 중 하나에 영구적으로 속하지 않는다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

Hybrid는 예외가 아니라 기업 환경에서 자주 사용하는 기본 패턴이다.

---

## 6. Human Steering 빈도를 본다

Cloud Agent는 비동기 위임에 유리하다.

그러나 작업 중 사람이 계속 방향을 바꿔야 한다면 이 장점은 줄어든다.

다음 작업을 생각해 보자.

```text
관리자 화면 구조를 개선해줘.
```

실제 작업은 다음처럼 진행될 수 있다.

```text
Agent: A안 구현
Developer: 메뉴 구조를 줄여보자
Agent: B안 구현
Developer: 데이터 흐름도 바꾸자
Agent: 다시 수정
```

이 경우 Developer가 계속 개입한다.

반대로 다음 Task는 다르다.

```text
Goal
Expired JWT → HTTP 401

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

작업 중간의 질문이 거의 필요 없다.

Routing 시 다음 질문을 사용한다.

> 사람이 작업 중간에 몇 번이나 개입해야 하는가?

개입이 많으면 Local에 남기는 편이 낫다.

---

## 7. Context 크기는 비용이자 Routing 조건이다

Agent가 작업하려면 Context가 필요하다.

문제는 Context가 얼마나 넓은가다.

작은 Context 예:

```text
AuthService.java
JwtTokenProvider.java
AuthServiceTest.java
```

큰 Context 예:

```text
Auth
Student
Permission
Payment
DB Schema
Common Exception
External API
Admin Web
```

작은 Context에서 해결 가능한 Task는 Cloud Agent에 보내기 쉽다.

반대로 여러 도메인을 동시에 이해해야 한다면 Local에서 먼저 문제를 좁히는 편이 낫다.

```text
큰 문제
→ Local 탐색
→ 결정
→ 작은 Task로 분해
→ Cloud 위임
```

Cloud Agent의 Context를 줄이는 상세 방법은 7장에서 다룬다.

---

## 8. Internal Network는 Hard Constraint가 될 수 있다

기업 개발에서는 다음 자원이 내부망에만 있을 수 있다.

```text
Internal Git
Nexus
Jenkins
Tibero / Oracle
Redis / Kafka
HSM
Internal API
VPN-only Server
```

Cloud에서 접근할 수 없다면 다른 조건이 좋아도 실행 위치가 제한된다.

예를 들어:

```text
Scope 명확
독립 검증 가능
중간 질문 없음
실행시간 30분
```

이어도 HSM 접근이 반드시 필요하다면 Local이 될 수 있다.

따라서 Routing Score보다 Hard Constraint를 먼저 확인한다.

```text
Hard Constraint
→ 먼저 확인

Optimization
→ 그다음 판단
```

---

## 9. Git으로 상태를 넘길 수 있는가

Cloud Worker는 Local IDE의 상태를 자동으로 아는 것이 아니다.

다음 입력에 강하게 의존하면 Cloud Handoff가 어렵다.

```text
미커밋 파일
IDE 내부 설정
로컬 DB 임시 데이터
untracked config
특정 프로세스 상태
```

Cloud Task는 가능한 한 다음처럼 명시적 상태에서 시작하는 것이 좋다.

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

결과도 Git과 Artifact로 회수한다.

```text
commit: def456
test: PASS
artifact: junit.xml
pr: #142
```

Git을 Handoff Boundary로 사용하는 상세 구조는 11장에서 다룬다.

---

## 10. File Conflict와 Dependency를 본다

독립 Branch가 있다고 해서 논리적 충돌까지 사라지는 것은 아니다.

좋은 병렬 Task:

```text
Agent A
→ attendance 모듈 테스트

Agent B
→ notification 모듈 테스트
```

충돌 가능성이 낮다.

반대로 다음 작업은 문제가 된다.

```text
Agent A
→ UserService 구조 변경

Agent B
→ UserService 구조 변경

Agent C
→ UserService 기반 DTO 변경
```

Cloud에서 각각 독립적으로 성공해도 Merge 시점에 충돌할 수 있다.

DB Migration도 같은 문제를 가진다.

```text
Agent A
→ V120 migration 추가

Agent B
→ V120 migration 추가
```

따라서 Routing 단계에서 다음을 확인한다.

```text
변경 예상 파일
shared/common module
DB schema
migration sequence
공통 DTO / API
순서 의존성
```

---

## 11. Task가 너무 작아도 Cloud가 불리하다

Cloud에는 고정 준비 비용이 있다.

```text
Worker Start
Repository Checkout
Branch 준비
Environment 준비
Context Load
Commit / PR
```

예를 들어:

```text
README 한 줄 수정
검증 5초
```

라면 Cloud 준비 비용이 작업 자체보다 클 수 있다.

개념적으로는 다음과 같다.

```text
Cloud Overhead > Task Work
→ Local 후보
```

여기서 `5분 이하이면 Local` 같은 절대 규칙은 사용하지 않는다.

프로젝트마다 Cold Start와 Review 비용이 다르기 때문이다.

실제로 측정한 값을 기준으로 판단한다.

---

## 12. Task가 너무 커도 Cloud가 불리하다

반대로 Task가 너무 크면 다음 비용이 증가한다.

```text
Repository 탐색
Context
Token
변경 파일 수
Retry 범위
Test 범위
Review 부담
Merge Risk
```

좋지 않은 Task:

```text
프로젝트 전체 Architecture를 개선해.
```

더 나은 Task:

```text
attendance 모듈의 deprecated API를 Java 21 기준으로 교체하고
./gradlew :attendance:test를 통과한다.
```

즉 적정 Task는 다음 조건에 가깝다.

```text
독립 실행 가능
독립 검증 가능
Review 가능한 Diff
명확한 완료 조건
```

7장의 Task Contract는 이 적정 범위를 명시하는 도구가 된다.

---

## 13. Routing Decision Matrix

기본 판단표는 다음과 같다.

| 조건 | Local | Cloud | Hybrid |
| --- | --- | --- | --- |
| Scope 명확 | 가능 | 유리 | 유리 |
| 요구사항 불명확 | 유리 | 불리 | 불리 |
| Human Steering 잦음 | 유리 | 불리 | 가능 |
| 큰 Context 필요 | 유리 | 불리 | 가능 |
| 장시간 Build/Test | 가능 | 유리 | 유리 |
| CPU/RAM 사용 큼 | 가능 | 유리 | 유리 |
| 내부망/VPN 필요 | 유리 | 불리 | 유리 |
| Git Handoff 가능 | 가능 | 유리 | 유리 |
| 독립 검증 가능 | 가능 | 유리 | 유리 |
| 동일 파일 충돌 큼 | 유리 | 불리 | 불리 |
| Architecture 판단 | 유리 | 불리 | 가능 |

이 표를 절대 규칙으로 사용하지 않는다.

한 항목이 Hard Constraint가 될 수 있기 때문이다.

예를 들어 `내부망 필수`가 Cloud 불가능 조건이라면 다른 조건의 합보다 우선한다.

---

## 14. Routing Score는 참고용이다

팀에서는 간단한 점수표를 사용할 수 있다.

예:

```text
Scope 명확성           +2
독립 검증 가능          +2
장시간 실행             +1
CPU/RAM 사용 큼         +1
Human Steering 많음     -2
Internal Network 필요   -3
큰 Context 필요         -2
File Conflict 높음      -2
```

점수가 높으면 Cloud 후보로 볼 수 있다.

하지만 이 점수로 자동 결정하지 않는다.

다음 순서를 유지한다.

```text
1. Hard Constraint
2. Task Independence
3. Verification
4. Handoff 가능성
5. 비용/시간 최적화
```

Score는 대화를 빠르게 하기 위한 도구일 뿐이다.

---

## 15. campus-platform 작업을 실제로 분류해보자

다음 Task를 가정한다.

```text
A. 신규 인증 Architecture 설계
B. AuthService expired token 수정
C. 전체 Unit Test
D. Web E2E
E. Tibero Migration 실검증
F. HSM 오류 분석
G. Docker Build
H. Dependency Update
I. README 문구 한 줄 수정
```

기본 Routing은 다음과 같다.

| Task | 실행 위치 / 주체 | 이유 |
| --- | --- | --- |
| 신규 인증 Architecture | Local | 큰 Context, Human Steering |
| expired token 수정 | Cloud Agent 후보 | 작은 Scope, 독립 검증 |
| 전체 Unit Test | Cloud Runner | 결정론적, Compute 중심 |
| Web E2E | Cloud Runner | Browser, 장시간 검증 |
| Tibero Migration 실검증 | Local/Hybrid | 내부 DB 의존 |
| HSM 오류 분석 | Local | 내부 장비/네트워크 |
| Docker Build | Cloud Runner | 결정론적 Build |
| Dependency Update | Runner-first | PASS면 Agent 불필요 |
| README 한 줄 수정 | Local | Cloud overhead가 더 큼 |

이 표의 목적은 정답을 외우는 것이 아니다.

같은 Task라도 프로젝트 환경에 따라 결과는 달라진다.

예를 들어 Cloud에서 내부 DB를 안전하게 제공할 수 있다면 Migration 검증 위치가 바뀔 수 있다.

중요한 것은 판단 기준을 일관되게 적용하는 것이다.

---

## 16. Routing은 한 번 하고 끝나는 결정이 아니다

Cloud로 보낸 Task가 진행 중 다음 사실을 발견할 수 있다.

```text
예상보다 Context가 큼
내부 API 필요
같은 파일을 다른 Agent가 수정 중
재현되지 않음
Human 판단 필요
```

이 경우 실행 위치를 다시 판단한다.

```text
Cloud
→ 조건 변경 발견
→ Local Fallback 또는 Task 재분해
```

반대로 Local에서 시작한 작업도 범위가 명확해지면 Cloud로 넘길 수 있다.

```text
Local 탐색
→ 원인 축소
→ 재현 테스트 생성
→ 작은 Cloud Task로 위임
```

Routing은 Task 생성 시점의 단발성 결정이 아니라 Workflow 중 반복되는 판단이다.

---

## 17. 이 장에서 사용할 판단 순서

실제 Task를 받으면 다음 순서로 본다.

```text
1. Runner로 끝낼 수 있는가?
2. Scope와 완료 조건이 명확한가?
3. 독립 검증 가능한가?
4. Git으로 상태를 전달할 수 있는가?
5. Human Steering이 많이 필요한가?
6. 큰 Context가 필요한가?
7. Internal Network가 필요한가?
8. File/Schema 충돌 가능성이 큰가?
9. Cloud Overhead보다 Task 가치가 큰가?
```

이 질문에 답하면 대부분의 작업은 `Local / Cloud / Hybrid / Runner-first` 중 하나로 좁혀진다.

다음 장에서는 이 Routing 기준을 Build, Unit Test, Integration Test, E2E, Docker Build, Refactoring, Bug Fix, Documentation, PR Review 같은 실제 개발 작업에 적용한다.
