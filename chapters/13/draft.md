# 13장. Local → Cloud → Local Handoff

클라우드 에이전트를 실제 개발에 넣으면 하나의 작업이 로컬 또는 클라우드 한쪽에만 머물지 않는다.

로컬에서 문제를 좁히고, 클라우드에서 독립 작업과 검증을 수행한 뒤, 다시 로컬에서 내부 환경을 확인하고 통합할 수 있다.

이 장의 대상은 실행 위치 자체가 아니라 **실행 위치 사이에서 작업 상태를 안전하게 넘기는 방법**이다.

기본 구조는 다음과 같다.

```text
Local
→ Task 정리
→ Git + Task Contract
      ↓
Cloud
→ Work / Runner / Agent
→ Evidence + Result SHA / PR
      ↓
Local
→ Review / Internal Validation / Merge
```

핵심 경계는 두 개다.

```text
Local → Cloud
Git + Task Contract

Cloud → Local
Evidence + Result SHA / PR
```

> 작업은 로컬 또는 클라우드 중 하나에 영구적으로 속하지 않는다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

---

## 1. Handoff는 대화가 아니라 상태 전달이다

다음 지시는 작업 전달 기준이 없다.

```text
내 Mac에서 하던 인증 수정 이어서 해줘.
```

클라우드 작업자는 로컬 IDE의 미커밋 파일, 임시 설정, 로컬 DB 상태를 자동으로 알지 못한다.

작업 전달에는 최소한 다음 질문의 답이 있어야 한다.

```text
어느 Source 상태에서 시작하는가?
무엇을 해야 하는가?
어디까지 바꿀 수 있는가?
무엇이 성공인가?
어떤 결과를 반환해야 하는가?
```

즉 작업 전달은 자연어 대화를 옮기는 것이 아니라 **재현 가능한 작업 상태를 전달하는 과정**이다.

---

## 2. Local → Cloud 입력은 작게 고정한다

7장의 작업 명세를 작업 전달 입력 형식으로 사용한다.

예:

```text
Task ID
AUTH-142

Base SHA
abc123

Goal
Expired JWT → HTTP 401

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Validation
./gradlew test --tests AuthServiceTest.expiredToken

Environment
backend-test

Expected Evidence
- Result SHA
- Changed Files
- Validation Result
```

로컬에서 알고 있는 프로젝트 정보를 모두 전달할 필요는 없다. 클라우드 작업자가 독립적으로 작업을 시작하는 데 필요한 최소한의 상태를 정해 전달한다.

```text
Local Context
→ Task에 필요한 부분만 추출
→ Cloud Handoff
```

---

## 3. Git이 Source Handoff Boundary가 된다

클라우드 작업자가 어느 코드에서 작업했는지 명확해야 한다.

기본 흐름은 다음과 같다.

```text
Local
→ Commit / Push
      ↓
Git Repository
      ↓
Cloud Worker
→ Checkout Base SHA
→ Task Branch
→ Work
→ Commit
```

예:

```text
Repository: campus-platform
Base SHA: abc123
Branch: agent/auth-142
```

결과도 SHA로 연결한다.

```text
Base SHA
abc123
   ↓
Result SHA
def456
   ↓
Verification SHA
def456
```

중요한 것은 브랜치 이름보다 **실제로 검증한 커밋을 식별할 수 있는가**다.

11장의 소스 코드 격리가 여기서는 로컬과 클라우드 사이의 작업 전달 경계가 된다.

---

## 4. 미커밋 Local State는 먼저 정리한다

로컬에 다음 상태가 있다고 하자.

```text
modified: AuthService.java
untracked: debug.yml
local DB state changed
```

이 상태가 작업에 필요하다면 클라우드에 넘길 형태로 만들어야 한다.

선택지는 단순하다.

```text
필요한 변경을 Commit / Push
또는
현재 Task를 Local에 유지
```

클라우드에 보내기 위해 모든 실험 상태를 억지로 커밋할 필요는 없다.

아직 요구사항과 변경 범위가 정리되지 않았다면 로컬에서 계속 좁힌 뒤 작업을 전달한다.

> 클라우드 작업 전달이 어렵다는 것은 때로 작업이 아직 독립 실행 가능한 상태가 아니라는 신호다.

---

## 5. Cloud에서는 중간 확인보다 독립 실행을 우선한다

작업 전달이 끝난 뒤 클라우드에서는 9~12장에서 만든 구조를 사용한다.

```text
Prepared Environment
      ↓
Runner-first
      ↓
필요한 경우 Cloud Agent
      ↓
Verification
      ↓
Evidence
```

예를 들어 같은 SHA에서 다음 검증을 병렬로 수행할 수 있다.

```text
Unit Test
Integration Test
Docker Build
E2E
```

클라우드 작업을 위임한 뒤 개발자가 계속 상태를 확인하고 매 단계마다 방향을 정한다면 비동기 작업 전달의 이점이 줄어든다.

따라서 클라우드로 넘기는 작업은 가능한 한 다음 조건을 가진다.

```text
완료 조건 명확
중간 Human Steering 적음
독립 검증 가능
결과를 Evidence로 반환 가능
```

---

## 6. Cloud → Local Return Boundary는 Evidence다

클라우드 작업자의 자연어 응답만으로 작업 완료를 판단하지 않는다.

로컬이 받아야 할 것은 검증 가능한 결과다.

예:

```text
Task: AUTH-142
Result SHA: def456

Changed Files
- AuthService.java
- AuthServiceTest.java

Validation Result
AuthServiceTest.expiredToken: PASS
AuthServiceTest: 24 / 24 PASS

Artifacts
- result.json
- junit.xml
```

로컬에서는 다음 순서로 확인할 수 있다.

```text
Evidence
→ Changed Files
→ 필요한 Diff
→ Internal Validation
```

8장의 Result Gateway와 검증 근거가 작업 전달의 반환 인터페이스가 된다.

---

## 7. PR은 Review 가능한 Return Package다

팀 개발에서는 Result SHA와 검증 근거를 PR로 묶을 수 있다.

```text
Cloud Worker
→ Result SHA
→ Push
→ PR
      ↓
Developer / Reviewer
```

PR에는 다음 관계가 연결되어야 한다.

```text
Task ID
Base SHA
Result SHA
Validation Result
Artifact Reference
```

이 책의 기본값은 자동 병합이 아니다.

```text
Cloud
→ 검증 가능한 변경 반환

Local / Human
→ Review / Integration / Merge 판단
```

클라우드 에이전트가 코드를 만들었다는 사실과 main에 반영해도 된다는 판단은 분리한다.

---

## 8. 내부망 검증은 마지막 경계로 남길 수 있다

기업 프로젝트에서는 클라우드에서 직접 재현하기 어려운 자원이 있다.

```text
Tibero / Oracle
HSM
Internal Jenkins
VPN-only API
사내 시스템
```

이 때문에 전체 작업을 로컬에서만 처리할 필요는 없다.

예를 들어 마이그레이션은 다음처럼 나눌 수 있다.

```text
Cloud
→ Migration 일반 검증
→ Disposable DB
→ Integration Test
→ Evidence
      ↓
Local / Internal
→ Tibero 실제 적용 검증
```

HSM 연동도 같은 원칙을 사용할 수 있다.

```text
Cloud
→ Pure Logic / Mock / Unit Test
      ↓
Local / Internal
→ Real HSM Integration
```

로컬과 클라우드를 함께 쓰는 작업 흐름의 핵심은 클라우드를 내부망처럼 만드는 것이 아니라 **클라우드가 끝내야 할 조건과 로컬이 확인해야 할 조건을 분리하는 것**이다.

---

## 9. Multi-Repository는 필요한 것만 넘긴다

하나의 기능이 여러 저장소에 걸쳐 있을 수 있다.

예:

```text
campus-api
campus-admin
campus-common
```

현재 작업이 API와 공통 데이터 전달 객체(DTO)를 함께 바꿔야 한다면 두 저장소가 필요할 수 있다.

반대로 관계없는 저장소까지 모두 연결하면 다음 비용이 늘어난다.

```text
Checkout
Context 탐색
검색 범위
잘못된 변경 가능성
```

따라서 각 저장소의 기준점을 명시하고 필요한 것만 전달한다.

```text
campus-api    @ abc123
campus-common @ 91de02
```

> 여러 저장소에 걸친 작업 전달에서도 원칙은 같다. 작업에 필요한 소스 코드만 제공하고 각 기준점을 고정한다.

---

## 10. Handoff 상태는 최소한으로 추적한다

작업이 로컬과 클라우드를 이동하면 현재 위치를 알 수 있어야 한다.

복잡한 작업 흐름 실행 엔진이 없어도 다음 정도면 시작할 수 있다.

```text
Task ID
Base SHA
Result SHA
Execution Location
Status
Validation Result
Artifact Path
PR
```

예:

```yaml
task_id: AUTH-142
base_sha: abc123
result_sha: def456
location: cloud
status: verifying
validation_result: PASS
artifact_path: artifacts/AUTH-142/result.json
pr: 142
```

내부망 검증으로 넘어가면 상태만 바꾼다.

```text
CLOUD_DONE
→ LOCAL_VALIDATION
→ READY_TO_MERGE
```

상태를 복잡하게 분류하려는 것이 아니다. **누가 어떤 소스 코드 상태를 검증하는지 계속 확인할 수 있게 하는 것**이 목적이다.

---

## 11. 실행 중 Routing 조건이 바뀔 수 있다

클라우드에서 작업하다 다음 사실이 드러날 수 있다.

```text
Internal Dependency 발견
Cloud에서 재현 불가
예상보다 Scope 확대
Architecture 판단 필요
```

이 경우 클라우드에 계속 머물러야 한다는 규칙은 없다.

```text
Cloud
→ 새 제약 발견
→ Routing 재판단
→ Local 또는 Task 재분해
```

반대로 로컬에서 조사하던 문제도 재현 테스트와 범위가 만들어지면 클라우드로 넘길 수 있다.

이 장에서는 작업 전달 가능성만 다룬다. 언제 클라우드 작업을 중단하고 로컬로 되돌릴지는 17장에서 정리한다.

---

## 12. Handoff의 목표는 Developer Blocking Time을 줄이는 것이다

좋은 작업 전달은 클라우드에 보낸 뒤 개발자가 다른 일을 할 수 있게 한다.

설명용 예시:

```text
10:00  Cloud Task 위임
10:02  Developer 다음 작업 시작
10:40  Cloud Evidence 생성
11:10  Developer Review
```

위 시간은 작업 전달과 개발자가 다른 일을 하지 못하고 기다리는 시간의 차이를 설명하기 위한 예시다.

클라우드 실행시간과 개발자가 다른 일을 하지 못하고 기다리는 시간은 다르다.

작업 명세와 검증 근거를 미리 정하는 이유도 중간 질문을 줄이기 위해서다.

```text
Local
→ 판단 / 분해
→ Handoff
      ↓
Cloud
→ 독립 실행
→ Evidence 반환
      ↓
Local
→ Review / Internal Validation / Merge
```

다음 장에서는 이 작업 전달의 시작점을 사람이 아니라 CI, 검토, 예약 실행 같은 이벤트로 바꾼다.
