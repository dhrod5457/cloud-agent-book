# 11장. Git, Branch, Worktree, Container로 작업 격리하기

클라우드 작업자를 여러 개 사용하려면 먼저 작업 상태를 분리해야 한다.

에이전트 수를 늘리면서 같은 작업 디렉터리와 Git 인덱스, DB, 임시 경로를 함께 쓰게 하면 작업끼리 충돌한다.

클라우드 작업의 격리는 크게 두 층으로 나눈다.

```text
Source Isolation
→ Git Branch / Worktree / Independent Clone

Runtime Isolation
→ Container / VM / Process / Port / DB / Artifact Path
```

이 둘은 다른 문제를 해결한다.

> 브랜치는 소스 코드 변경을 격리하고, 컨테이너나 VM은 실행 상태를 격리한다.

> Git은 로컬과 클라우드 사이의 작업을 넘기는 경계이자 작업 상태의 기준점이다.

---

## 1. Cloud Task는 명시적인 Git 상태에서 시작한다

로컬 개발자의 IDE에는 Git에 없는 상태가 많다.

```text
modified files
untracked files
local config
local DB
running process
IDE state
```

클라우드 작업자는 이 상태를 자동으로 알지 못한다.

따라서 시작점을 Git으로 고정한다.

```text
Repository
campus-platform

Base SHA
abc123

Task Branch
agent/auth-expired-token-142
```

기본 작업 전달은 다음처럼 본다.

```text
Local
→ Commit / Push
      ↓
Git Repository
      ↓
Cloud Worker
→ Checkout
→ Task Branch
→ Work / Test
→ Commit / Push
→ PR
```

Git은 다음 질문에 답하는 기준점이 된다.

```text
어떤 코드에서 시작했는가?
무엇이 변경됐는가?
어떤 Commit이 검증됐는가?
어떤 결과가 어느 Source와 연결되는가?
```

---

## 2. Base SHA를 고정한다

브랜치 이름만 기록하면 작업이 실제 어느 시점에서 시작했는지 불명확할 수 있다.

```text
Task 시작
main = abc123

이후
main = xyz999
```

따라서 작업 상태에 Base SHA를 남긴다.

```text
Task: AUTH-142
Base SHA: abc123
Branch: agent/auth-expired-token-142
```

에이전트가 수정 결과를 만들면 검증 결과도 그 Result SHA와 연결한다.

```text
Base SHA abc123
   ↓
Result SHA def456
   ↓
Runner Verification def456
   ↓
PR
```

다음 결과는 검증 근거가 아니다.

```text
Result SHA: def456
Validation Result: abc123 기준 PASS
```

> 소스 코드 상태와 검증 근거는 같은 커밋 기준으로 연결한다.

---

## 3. Branch per Task

독립 클라우드 작업은 가능한 한 독립 브랜치와 연결한다.

```text
AUTH-142
→ agent/auth-expired-token-142

ATTEND-211
→ agent/attendance-retry-211
```

브랜치 이름 규칙 자체보다 다음 대응관계가 중요하다.

```text
Task
↕
Branch
↕
Result SHA
↕
Validation Result
↕
PR
```

이 구조가 있으면 대화 기록을 뒤지지 않고 Git 상태로 작업을 추적할 수 있다.

작업 취소나 재시도도 특정 브랜치 기준으로 처리하기 쉽다.

---

## 4. Task 상태를 Git과 연결한다

여러 작업자를 운영할 때 최소 상태는 다음 정도면 된다.

```text
Task ID
Session ID
Base SHA
Branch
Result SHA
Status
Validation Result
Artifact Path
PR
```

예:

```yaml
task_id: AUTH-142
session_id: cloud-142
base_sha: abc123
branch: agent/auth-expired-token-142
result_sha: def456
status: verifying
validation_result: PASS
artifact_path: artifacts/AUTH-142/def456/
pr: 142
```

상태를 거대한 에이전트 플랫폼 UI로 관리할 필요는 없다.

핵심은 `Task → Git SHA → Evidence → PR` 연결이 깨지지 않는 것이다.

---

## 5. Local에서는 Worktree로 Source를 나눌 수 있다

로컬에서 여러 브랜치를 동시에 열어야 한다면, 하나의 저장소에 별도 작업 디렉터리를 만드는 Git Worktree가 유용하다.

```text
worktrees/
├─ auth-task/
├─ attendance-task/
└─ notification-task/
```

각 Worktree는 별도 작업 디렉터리와 브랜치를 가진다.

```text
auth-task
→ agent/auth

attendance-task
→ agent/attendance
```

다만 Worktree가 분리하는 것은 소스 코드 작업공간이다. 두 Worktree는 여전히 다음 자원을 함께 사용할 수 있다.

```text
localhost:8080
local PostgreSQL
/tmp/campus-test
same Docker network
same artifact folder
```

따라서 Worktree를 만들었다고 실행환경까지 격리됐다고 보지 않는다.

---

## 6. Cloud에서는 독립 Workspace와 Runtime이 단순하다

클라우드 작업자는 작업마다 독립 작업공간을 주는 편이 관리하기 쉽다.

```text
Worker A
├─ clone A
├─ branch A
├─ runtime A
├─ temp A
└─ artifacts A

Worker B
├─ clone B
├─ branch B
├─ runtime B
├─ temp B
└─ artifacts B
```

격리 대상은 소스 코드만이 아니다.

```text
Working Directory
Git Index
Process
Port
Temp Directory
Disposable DB
Test Output
Artifact Path
```

통합 테스트가 PostgreSQL을 띄운다면 DB 상태도 작업자별로 분리한다.

```text
Worker A → postgres-A
Worker B → postgres-B
```

한 작업의 테스트 데이터가 다른 작업의 결과에 영향을 주지 않게 하는 것이 목적이다.

---

## 7. Branch와 Container는 다른 문제를 해결한다

다음 구분을 유지한다.

```text
Branch / Worktree / Clone
→ Source Isolation

Container / VM / Process Boundary
→ Runtime Isolation
```

브랜치가 달라도 같은 DB를 쓰면 실행환경 충돌이 생길 수 있다.

```text
Branch A
Branch B
   ↓
same DB
```

반대로 컨테이너가 달라도 두 에이전트가 같은 핵심 파일을 바꾸면 병합 시 충돌한다.

```text
Container A → UserService.java 수정
Container B → UserService.java 수정
```

따라서 작업을 병렬화하기 전에 두 질문을 모두 확인한다.

```text
Source 상태가 분리됐는가?
Runtime 상태가 분리됐는가?
```

---

## 8. Artifact 경로도 Task별로 분리한다

여러 실행기가 같은 경로에 결과를 쓰면 검증 근거가 섞일 수 있다.

좋지 않은 구조:

```text
artifacts/result.json
artifacts/build.log
```

권장:

```text
artifacts/AUTH-142/def456/
├─ result.json
├─ junit.xml
└─ build.log
```

다른 작업은 별도 경로를 사용한다.

```text
artifacts/ATTEND-211/xyz789/
```

결과물 내부에도 Git SHA를 기록하면 8장의 검증 근거가 어느 소스 코드 상태의 결과인지 추적하기 쉽다.

---

## 9. 물리적 격리가 논리적 충돌을 없애지는 않는다

완전히 다른 컨테이너에서 작업해도 같은 파일을 바꾸면 통합 시 충돌할 수 있다.

```text
Agent A → UserService 예외 구조 변경
Agent B → UserService 성능 수정
Agent C → UserService logging 변경
```

실행 중에는 서로 방해하지 않아도 결과 통합 시점에는 다음 문제가 생길 수 있다.

```text
Merge Conflict
Semantic Conflict
Regression
```

따라서 병렬화 전에 예상 변경 영역을 본다.

```text
Expected Files
Shared Module
Public API
Common DTO
Central Config
Migration / DB Schema
```

여기서 병렬화의 비용을 계산하지는 않는다. 그 문제는 12장에서 다룬다.

11장의 역할은 **어떤 상태를 반드시 분리해야 하는가**를 정하는 것이다.

---

## 10. Migration과 DB Schema는 별도 경계가 필요하다

DB 구조 변경 작업은 일반 소스 코드 파일보다 순서 의존성이 크다.

예:

```text
Task A
→ V142__add_student_index.sql

Task B
→ V142__add_attendance_status.sql
```

각 브랜치에서는 문제가 없어도 병합 시 번호가 충돌한다.

번호가 달라도 적용 순서가 중요할 수 있다.

```text
V142 → column 생성
V143 → index 생성
```

따라서 마이그레이션 작업에서는 다음을 별도로 본다.

```text
번호 정책
적용 순서
DB Schema dependency
통합 Migration Validation
```

컨테이너를 나눈다고 DB 구조 변경의 논리적 의존성이 사라지는 것은 아니다.

---

## 11. Shared/Common Module은 선행 Task가 될 수 있다

처음에는 독립적으로 보이는 작업이 공통 모듈 때문에 연결될 수 있다.

```text
Task A → attendance
Task B → notification
Task C → student
```

세 작업이 모두 `common` DTO 변경을 요구한다면 실제 의존 관계는 다음과 같다.

```text
Task A ─┐
Task B ─┼→ common DTO
Task C ─┘
```

이 경우 공통 변경을 먼저 분리할 수 있다.

```text
Task 0
common DTO 변경
   ↓
새 Base SHA
   ↓
Task A / B / C
```

이 구조는 12장의 작업 간 의존 관계를 고려한 병렬 그룹으로 이어진다.

---

## 12. Multi-Repository도 필요한 범위만 연결한다

하나의 기능이 여러 저장소에 걸쳐 있을 수 있다.

```text
campus-api
campus-admin
campus-common
```

현재 작업에서 함께 수정될 가능성이 높은 저장소만 작업공간에 제공한다.

무관한 저장소까지 연결하면 다음 비용이 생긴다.

```text
Checkout 증가
검색 범위 증가
Context 증가
잘못된 변경 가능성 증가
```

여러 저장소에 걸친 작업에서는 각 저장소의 기준점도 고정한다.

```text
campus-api    → Base SHA aaa111
campus-common → Base SHA bbb222
```

Git 작업 전달이 여러 저장소로 늘어났을 뿐 원칙은 같다.

---

## 13. campus-platform 격리 예

AUTH-142와 ATTEND-211을 동시에 진행한다고 하자.

```text
AUTH-142
Branch: agent/auth-expired-token-142
Result SHA: def456
Runtime: backend-test-A
Artifact Path: artifacts/AUTH-142/def456/

ATTEND-211
Branch: agent/attendance-retry-211
Result SHA: xyz789
Runtime: backend-test-B
Artifact Path: artifacts/ATTEND-211/xyz789/
```

각 작업은 소스 코드, 실행환경, 결과물을 분리한다.

그러나 두 작업이 같은 `common-auth`를 수정해야 한다면 병렬 실행 가능 여부를 다시 판단한다.

격리는 병렬화를 가능하게 만드는 조건이지, 모든 작업을 병렬로 만들어주는 장치가 아니다.

---

## 14. 11장에서 기억할 경계

클라우드 작업을 격리할 때 다음 세 상태를 확인한다.

```text
Source
→ Branch / Worktree / Clone

Runtime
→ Process / Container / VM / DB / Port

Evidence
→ Task ID / Git SHA별 Artifact Path
```

그리고 논리적 충돌은 별도로 확인한다.

```text
same file
shared module
public API
migration
DB Schema
```

다음 장에서는 이렇게 격리된 작업 중 실제로 무엇을 병렬화할지, 에이전트 수를 늘릴수록 맥락 정보를 중복해서 읽는 비용과 검토·결과 통합 비용이 얼마나 생기는지 살펴본다.
