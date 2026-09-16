# 13장. Local → Cloud → Local Handoff

Cloud Agent를 실제 개발에 적용하면 하나의 기능이 Local과 Cloud 사이를 여러 번 이동하게 된다.

Local에서 요구사항과 Architecture를 정하고, Cloud에서 독립 실행과 검증을 수행한 뒤, 다시 Local에서 내부망 검증과 최종 통합을 진행하는 식이다.

이 장의 핵심은 Local과 Cloud를 둘 중 하나로 고르는 것이 아니다.

> Task는 Local 또는 Cloud 중 하나에 영구적으로 속하는 것이 아니다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

그리고 이 이동을 가능하게 만드는 경계가 있다.

```text
Local → Cloud
Git / Task Contract

Cloud → Local
Evidence / Commit / PR
```

따라서 Handoff는 단순히 “다른 환경에서 이어서 작업한다”는 의미가 아니다.

작업 상태를 전달 가능한 형태로 만들고, 결과를 다시 검증 가능한 형태로 회수하는 과정이다.

---

## 1. Local과 Cloud는 서로 다른 강점을 가진다

Local은 다음 작업에 강하다.

```text
요구사항 해석
Architecture 판단
넓은 Context 탐색
개발자와 빠른 질문/수정 반복
VPN / 내부망 접근
사내 DB / HSM / Jenkins 확인
최종 Review / Integration
```

Cloud는 다음 작업에 강하다.

```text
장시간 Build / Test
독립 Feature 작업
반복 Refactoring
E2E
Docker Build
CI Failure Fix
여러 독립 검증의 병렬 실행
```

따라서 실제 Workflow는 다음처럼 나뉠 수 있다.

```text
Local
→ 요구사항 분석
→ Architecture
→ 핵심 코드 작성
→ Task Split
→ Commit / Push
       ↓
Cloud
→ Build
→ Unit / Integration / E2E
→ Refactoring
→ CI Fix
→ Evidence / PR
       ↓
Local
→ 내부망 검증
→ Review
→ Integration
→ Merge
```

Cloud Agent는 Local Agent를 대체하지 않는다.

각 단계에서 더 적합한 실행 위치를 선택한다.

---

## 2. Handoff 전에 Local에서 Task를 정리한다

Cloud Worker가 Local IDE의 현재 상태를 그대로 볼 수 있다고 가정하면 Handoff가 불명확해진다.

예를 들어 다음 지시는 좋지 않다.

```text
내 Mac에서 하던 인증 수정 이어서 해줘.
```

이 문장에는 다음 정보가 없다.

```text
어느 Commit 기준인가?
어떤 파일이 수정 중인가?
어디까지 바꿔도 되는가?
무엇이 완료 조건인가?
어떤 환경에서 검증해야 하는가?
```

Cloud에 넘기기 전 최소한 다음을 준비한다.

```text
Base SHA
Task Branch 또는 Branch 정책
Task Contract
Relevant Files
Validation
Expected Result
Environment
Output / Evidence
```

예:

```text
Task
AuthService expired token fix

Base SHA
abc123

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Validation
./gradlew test --tests AuthServiceTest.expiredToken

Expected Result
expired token → HTTP 401
```

7장의 Task Contract를 Local→Cloud Handoff의 입력 형식으로 사용하는 셈이다.

---

## 3. Git이 Local → Cloud Handoff Boundary가 된다

Local에서 Cloud로 작업을 넘길 때 가장 명확한 기준점은 Git Commit이다.

```text
Local
→ 수정
→ Test
→ Commit
→ Push
      ↓
Git Repository
      ↓
Cloud Worker
```

예:

```text
Repository
campus-platform

Base SHA
abc123

Branch
agent/auth-expired-token
```

Cloud Worker는 이 상태를 checkout한다.

그리고 독립된 Branch에서 작업한다.

```text
Cloud Worker
→ Checkout abc123
→ Branch agent/auth-expired-token
→ Work
→ Test
→ Commit
```

이 구조의 장점은 시작점과 결과를 모두 SHA로 추적할 수 있다는 점이다.

```text
Base SHA
abc123
   ↓
Agent Commit
def456
   ↓
Runner Verification
on def456
```

11장에서 다룬 Git isolation이 여기서는 Local↔Cloud 전체 Workflow의 Handoff Protocol이 된다.

---

## 4. Local의 미커밋 상태는 자동으로 전달되지 않는다

Local에서 다음 상태라고 하자.

```text
modified: AuthService.java
modified: JwtTokenProvider.java
untracked: debug.yml
```

이 상태를 Cloud Worker가 자동으로 안다고 가정하면 안 된다.

선택지는 다음과 같다.

```text
1. 필요한 변경을 Commit하고 Cloud에 전달
2. Task에 필요한 Patch를 명시적으로 전달
3. 아직 범위가 정리되지 않았다면 Local에서 계속 작업
```

책의 기본 흐름은 Commit/Push를 사용한다.

```text
Local Working State
→ 정리
→ Commit
→ Cloud Handoff
```

Cloud에 보내기 위해 억지로 모든 실험 상태를 Commit할 필요는 없다.

아직 요구사항이나 방향이 불명확하다면 그 Task는 Local에 더 오래 남겨도 된다.

---

## 5. Cloud에서는 독립 실행과 검증을 수행한다

Cloud로 넘어간 Task는 개발자 PC와 분리된 환경에서 실행된다.

예:

```text
Cloud #1
→ Unit Test

Cloud #2
→ Integration Test

Cloud #3
→ Docker Build

Cloud #4
→ Admin E2E
```

또 작은 코드 수정 Task를 Agent에게 맡길 수도 있다.

```text
Cloud Agent
→ AuthService expired token 분석
→ 코드 수정
→ Commit
→ Runner Verification
```

이 단계에서는 Local Developer가 계속 상태를 확인하는 것이 목표가 아니다.

Cloud의 비동기성을 활용한다.

```text
Developer
→ Task 위임
→ 다음 Feature 작업

Cloud
→ 실행 / 검증
→ 완료 후 Result 반환
```

Cloud를 동기 Remote Desktop처럼 사용하면 Handoff의 장점이 줄어든다.

---

## 6. Cloud → Local Return Boundary는 Evidence다

Cloud Task가 끝났을 때 다음 응답만 받는 것은 부족하다.

```text
작업 완료했습니다.
```

Local에서 결과를 검토하려면 실행 사실이 필요하다.

반환 후보:

```text
Result Commit SHA
Changed Files
Build Result
Unit Test Result
Integration Test Result
E2E Result
Artifact
Screenshot / Video
PR
```

예:

```text
Result
commit: def456

Changed Files
- AuthService.java
- AuthServiceTest.java

Validation
./gradlew test --tests AuthServiceTest
24 / 24 PASS

PR
#142
```

Local에서는 Agent의 자연어 설명보다 Evidence를 먼저 볼 수 있다.

```text
Cloud Result
→ Evidence 확인
→ 필요한 Diff 확인
→ Internal Validation
```

8장의 Evidence-based Result가 Handoff의 반환 인터페이스가 된다.

---

## 7. PR은 Review 가능한 Return Package가 된다

Cloud Worker가 결과를 Commit만 만들고 끝내도 된다.

하지만 팀 Workflow에서는 PR이 더 편리한 Return Package가 될 수 있다.

```text
Cloud Worker
→ Commit
→ Push
→ PR
      ↓
Local Developer / Reviewer
```

PR에 다음 정보가 연결될 수 있다.

```text
Task ID
Base SHA
Result Commit
Changed Files
Validation Result
Artifact Link
```

중요한 것은 자동 Merge가 아니다.

이 책의 기본값은 다음이다.

```text
Cloud Worker
→ 검증 가능한 변경 생성
→ PR 반환
→ Human Review
```

Merge는 Local Integration 단계에서 결정한다.

---

## 8. 내부망 검증은 Local에 남겨도 된다

기업 프로젝트에서는 Cloud에서 재현하기 어려운 자원이 있다.

예:

```text
Tibero
Oracle
HSM
내부 Jenkins
VPN-only API
사내 Nexus
```

이 자원을 모두 Cloud 환경으로 복제해야 하는 것은 아니다.

예를 들어 Migration Task를 다음처럼 나눌 수 있다.

```text
Cloud
→ PostgreSQL/Testcontainers validation
→ Unit / Integration
→ Docker Build
      ↓
Local
→ Tibero 실제 적용 검증
```

HSM 작업도 같다.

```text
Local
→ 실제 HSM protocol / constraint 확인
      ↓
Cloud
→ pure Java logic
→ unit test
      ↓
Local
→ HSM 실제 integration
```

Cloud에서 할 수 있는 구간만 분리해도 충분한 효과를 얻을 수 있다.

> Cloud에 없는 자원을 억지로 복제하기보다 Handoff로 해결할 수 있다.

---

## 9. Hybrid Task는 단계로 분해한다

하나의 Task가 Local 또는 Cloud 하나에만 있어야 한다고 생각하면 기업 환경에서 Cloud 활용 범위가 좁아진다.

예를 들어 DB Migration 작업을 보자.

```text
Local
→ 요구사항 확인
→ Schema 영향 분석
      ↓
Cloud
→ Migration 작성
→ Disposable DB Validation
→ Integration Test
      ↓
Local
→ Tibero 실제 적용
→ 운영 제약 확인
```

또 출입 카드/HSM 연동 작업이라면:

```text
Local
→ 장비 Protocol 확인
→ 실제 APDU/HSM 제약 확인
      ↓
Cloud
→ Encoding / Crypto utility unit test
→ pure business logic test
      ↓
Local
→ 실제 장비 Integration
```

Hybrid Workflow의 핵심은 경계를 명확히 만드는 것이다.

```text
Cloud가 끝내야 하는 조건
Local이 확인해야 하는 조건
```

을 미리 나눈다.

---

## 10. Cloud에서 실패했다고 바로 Local로 가져오지 않는다

Cloud Task가 실패하면 가장 먼저 Local로 되돌리는 것도 비효율적이다.

우선 Cloud 안에서 재현 가능한 실패인지 확인한다.

```text
Runner FAIL
→ Result Gateway
→ Failure Summary
→ Cloud Agent
→ Fix
→ Runner 재검증
```

Local로 되돌릴 조건은 따로 둔다.

```text
Internal Network 필요
Context가 예상보다 커짐
Cloud 환경에서 재현 불가
동일 Failure 반복
Budget 소진
Architecture 판단 필요
```

예:

```text
Cloud Integration Test FAIL
→ 원인: 내부 Tibero 전용 SQL
→ Local Fallback
```

이 경우 Cloud Task 실패가 아니다.

Routing 조건이 바뀐 것이다.

---

## 11. Local Fallback은 정상 Workflow다

Cloud Agent를 도입하면 모든 Task를 Cloud에서 끝내야 할 것처럼 생각하기 쉽다.

하지만 중요한 것은 Cloud 성공률이 아니다.

전체 개발 비용이다.

예:

```text
Cloud Attempt
→ 내부 HSM 필요 발견
→ Local Fallback
```

또는:

```text
Cloud Agent
→ 관련 모듈 3개 예상
→ 실제로 12개 모듈 영향 발견
→ Scope 확대
→ Local Architecture Review로 전환
```

이 흐름은 정상이다.

> 끝까지 Cloud에서 해결하는 것이 목표가 아니라 가장 적합한 실행 위치로 Task를 이동시키는 것이 목표다.

5장의 Task Routing을 실행 중에도 다시 적용한다.

---

## 12. Handoff 상태를 최소한으로 관리한다

Task가 Local과 Cloud를 오가면 현재 상태를 알 수 있어야 한다.

복잡한 Workflow Engine이 반드시 필요한 것은 아니다.

다음 정도의 상태면 충분할 수 있다.

```text
LOCAL_ANALYSIS
READY_FOR_CLOUD
CLOUD_RUNNING
CLOUD_VERIFYING
CLOUD_DONE
LOCAL_VALIDATION
READY_TO_MERGE
```

예:

```yaml
task_id: AUTH-142
status: CLOUD_VERIFYING
base_sha: abc123
cloud_branch: agent/auth-142
result_sha: def456
```

Local 검증이 필요하다면 다음을 기록할 수 있다.

```yaml
local_validation:
  - tibero
  - internal-jenkins
```

목적은 상태 모델을 크게 만드는 것이 아니라 사람이 현재 Task의 위치를 빠르게 아는 것이다.

---

## 13. Developer Blocking Time을 줄이는 방식으로 Handoff한다

Cloud의 장점 중 하나는 비동기성이다.

좋은 흐름:

```text
10:00
Developer → Cloud Task 위임

10:02
Developer → 다음 Feature 작업

10:40
Cloud Task 완료

11:10
Developer → Evidence Review
```

좋지 않은 흐름:

```text
Developer
→ Cloud Task 위임
→ 계속 상태 확인
→ Agent 질문 대기
→ Retry 여부 계속 판단
```

후자는 실행 위치만 Cloud일 뿐 사실상 동기 작업이다.

Cloud에 보내는 Task는 가능한 한 중간 Human Steering을 줄인다.

Task Contract, Validation, Evidence를 미리 정의하는 이유도 여기에 있다.

---

## 14. Multi-Repository Handoff는 필요한 Repository만 연결한다

실제 기능은 여러 Repository에 걸칠 수 있다.

예:

```text
campus-api
campus-common
campus-admin
```

학생 프로필 변경이 API, Common DTO, Admin UI에 걸친다고 하자.

Cloud Workspace에는 필요한 Repository만 포함한다.

```text
Task Scope
├─ campus-api
├─ campus-common
└─ campus-admin
```

무관한 Repository까지 모두 연결하면 다음 비용이 늘어난다.

```text
Checkout
Context
Search Range
Dependency 이해
잘못된 변경 가능성
```

Multi-Repo에서도 7장의 작은 Context 원칙을 유지한다.

---

## 15. Multi-Repo에서는 각 Base Version을 고정한다

Repository가 여러 개라면 하나의 Base SHA로는 부족하다.

예:

```text
campus-api: abc123
campus-common: def456
campus-admin: ghi789
```

Cloud 결과도 각각 연결한다.

```text
campus-api: aaa111
campus-common: bbb222
campus-admin: ccc333
```

Cross-repo 변경은 Local Integration에서 의존성과 Merge 순서를 확인한다.

예:

```text
common DTO merge
      ↓
API merge
      ↓
Admin UI merge
```

모든 Repository 변경이 항상 병렬로 Merge될 수 있는 것은 아니다.

---

## 16. campus-platform Hybrid Workflow

`campus-platform`에서 출결 API 기능을 변경한다고 가정하자.

Local에서 먼저 다음을 수행한다.

```text
Requirement
→ 출결 인증 규칙 확인
→ Architecture 영향 확인
→ Task Split
→ Commit / Push
```

Cloud에서는 독립 검증을 fan-out한다.

```text
Cloud #1
→ Unit Test

Cloud #2
→ Integration Test

Cloud #3
→ Docker Build

Cloud #4
→ Admin E2E
```

필요하면 작은 Bug Fix는 Cloud Agent가 수행한다.

```text
runner-integration FAIL
→ Result Gateway
→ Cloud Agent Fix
→ Runner PASS
```

결과가 준비되면 Local로 돌아온다.

```text
Evidence / PR
      ↓
Local
→ Tibero 최종 검증
→ 내부 Jenkins
→ Review
→ Merge
```

전체 구조:

```text
Local
Requirement / Architecture / Task Split
      ↓
Git Handoff
      ↓
Cloud
Runner / Agent / Parallel Validation
      ↓
Evidence / PR
      ↓
Local
Internal Validation / Review / Integration
```

이 흐름이 이 책에서 사용하는 Hybrid Workflow의 기본형이다.

---

## 17. Handoff 전에 확인할 질문

Local에서 Cloud로 보내기 전:

```text
Base SHA가 정해졌는가?
Task Scope가 명확한가?
Validation이 실행 가능한가?
Cloud Environment에서 재현 가능한가?
Expected Result가 관찰 가능한가?
필요한 Repository만 포함했는가?
```

Cloud에서 Local로 받을 때:

```text
Result Commit이 있는가?
Verification SHA가 같은가?
Test Result가 있는가?
Artifact가 남아 있는가?
Internal Validation이 필요한가?
PR이 Review 가능한 크기인가?
```

Handoff의 품질은 메시지 길이보다 기준점과 Evidence의 명확성으로 판단한다.

---

## 이 장에서 기억할 것

```text
Local
→ 설계 / 탐색 / 내부망

Git
→ Handoff Boundary

Cloud
→ 독립 실행 / 장시간 검증 / 병렬 처리

Evidence / PR
→ Return Boundary

Local
→ 내부 검증 / Review / Merge
```

Cloud Agent 활용에서 중요한 것은 Local을 없애는 것이 아니다.

작업 단계에 맞게 실행 위치를 이동시키는 것이다.

다음 장에서는 이 Handoff의 시작점이 사람이 아닐 수도 있다는 점을 다룬다.

CI Failure, Issue, Review Comment, Scheduled Test 같은 이벤트가 Task를 만들고 Cloud Worker를 호출하는 구조로 확장한다.
