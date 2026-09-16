# 11장. Git, Branch, Worktree, Container로 작업 격리하기

Cloud Agent를 여러 개 사용하려면 먼저 작업공간을 나눠야 한다.

Agent 수만 늘리고 같은 Working Directory, 같은 Git index, 같은 임시 파일, 같은 DB를 공유하면 병렬화가 아니라 충돌을 만든다.

Cloud Agent 시대의 격리는 크게 두 층으로 나눠 볼 수 있다.

```text
Source Isolation
→ Git Branch / Worktree / Independent Clone

Runtime Isolation
→ Container / VM / Process / Port / DB / Artifact Path
```

이 둘은 같은 문제가 아니다.

Branch를 나눴다고 Runtime이 자동으로 분리되는 것도 아니고, Container를 나눴다고 같은 파일을 동시에 수정하는 논리적 충돌까지 없어지는 것도 아니다.

이 장에서는 Git을 단순 형상관리 도구가 아니라 Local과 Cloud 사이의 Handoff Boundary이자 Remote Worker의 작업 상태를 고정하는 기준점으로 본다.

> Cloud Agent 시대에는 Git이 개발자와 Remote Worker 사이의 작업 전달 프로토콜 역할까지 수행한다.

---

## 1. Cloud Task는 명시적인 Git 상태에서 시작한다

Local 개발자는 현재 IDE 안에 많은 상태를 가지고 있다.

```text
modified files
untracked files
local config
IDE state
local DB
running process
```

하지만 Cloud Worker는 이 상태를 자동으로 알지 못한다.

다음과 같은 지시는 Handoff 기준이 불명확하다.

```text
내 Mac에서 지금 하던 상태 기준으로 이어서 수정해줘.
```

Cloud Worker가 재현 가능한 Task를 받으려면 시작점을 명시해야 한다.

```text
Repository
campus-platform

Base SHA
abc123

Task Branch
agent/auth-expired-token-142
```

기본 Handoff는 다음처럼 된다.

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
→ Checkout
→ Task Branch
→ Work
→ Test
→ Commit
→ Push
→ PR
```

여기서 Git은 Source Code 저장소 이상의 역할을 한다.

다음 질문에 답하는 기준점이 된다.

```text
어떤 코드에서 시작했는가?
어떤 변경이 추가됐는가?
어떤 Commit을 테스트했는가?
어떤 결과가 어느 코드와 연결되는가?
```

---

## 2. Base SHA를 고정한다

Cloud Task를 만들 때 Branch 이름만 적는 것보다 Base SHA를 함께 기록하는 편이 낫다.

예:

```text
Task: AUTH-142
Base SHA: abc123
Branch: agent/auth-expired-token-142
```

이유는 Branch가 움직일 수 있기 때문이다.

Task가 시작한 뒤 main에 새로운 Commit이 들어올 수 있다.

```text
Task 시작
main = abc123

10분 뒤
main = xyz999
```

Cloud Worker가 어느 시점의 코드에서 시작했는지 알 수 있어야 결과를 재현할 수 있다.

검증 결과도 Commit과 연결한다.

```text
Base SHA
abc123
   ↓
Agent Commit
def456
   ↓
Runner Verification
def456
   ↓
PR
```

좋지 않은 결과:

```text
Agent가 def456을 만들었음
하지만 Test Result가 abc123에서 실행됨
```

이 결과는 수정 코드가 실제로 검증됐다는 Evidence가 아니다.

> Source 상태와 Evidence는 같은 Commit 기준으로 연결한다.

---

## 3. Branch per Task

작은 Cloud Task는 가능한 한 독립 Branch와 연결한다.

```text
AUTH-142
→ agent/auth-expired-token-142

ATTEND-211
→ agent/attendance-retry-211
```

이 구조의 장점은 단순하다.

```text
변경 범위를 추적하기 쉬움
Task 취소가 쉬움
Retry가 쉬움
PR 연결이 쉬움
다른 Task와 Git index가 섞이지 않음
```

Branch 이름 규칙 자체가 핵심은 아니다.

중요한 것은 다음 대응관계가 명확한 것이다.

```text
Task
↕
Branch
↕
Commit
↕
Verification
↕
PR
```

Cloud Worker가 무엇을 했는지 대화 기록만 뒤지지 않아도 Git 상태로 추적할 수 있어야 한다.

---

## 4. Task State를 Git과 연결한다

Cloud Task 상태를 다음 정도로 관리할 수 있다.

```text
Task ID
Session ID
Base SHA
Branch
Current Commit SHA
Status
Test Result
Artifact Path
PR
```

예:

```yaml
task_id: AUTH-142
session_id: cloud-142
base_sha: abc123
branch: agent/auth-expired-token-142
current_sha: def456
status: verifying
test: PASS
artifact: artifacts/AUTH-142/
pr: 142
```

이런 상태 정보는 여러 Worker가 동시에 실행될 때 특히 중요하다.

다음 상황을 생각해 보자.

```text
Worker A → attendance
Worker B → notification
Worker C → admin UI
Worker D → Docker verification
```

대화창 네 개만 보고 상태를 관리하면 어떤 결과가 어떤 Commit과 연결되는지 헷갈리기 쉽다.

Task ID와 Git SHA를 기준으로 보면 결과를 기계적으로 연결할 수 있다.

---

## 5. Local에서는 Worktree로 작업공간을 분리할 수 있다

Local에서 여러 Branch를 동시에 열어야 한다면 Git Worktree를 사용할 수 있다.

```text
repo/
worktrees/
├─ auth-task/
├─ attendance-task/
└─ notification-task/
```

각 Worktree는 별도 Working Directory를 가진다.

```text
worktree/auth-task
→ branch agent/auth

worktree/attendance-task
→ branch agent/attendance
```

장점은 같은 Repository object database를 재사용하면서 작업 디렉터리와 Branch를 나눌 수 있다는 점이다.

하지만 Worktree만 만들었다고 모든 실행환경이 격리되는 것은 아니다.

예를 들어 두 Worktree가 동시에 다음을 사용할 수 있다.

```text
localhost:8080
localhost PostgreSQL
/tmp/campus-test
same Docker network
same artifact folder
```

이 경우 Source는 분리되어 있지만 Runtime은 충돌할 수 있다.

따라서 Worktree는 Source 격리 수단으로 이해한다.

---

## 6. Cloud에서는 Independent Clone과 Container가 단순하다

Cloud Worker는 보통 작업별로 독립된 Workspace를 주는 편이 관리하기 쉽다.

개념적으로 다음과 같다.

```text
Worker A
├─ clone A
├─ branch A
├─ container A
├─ temp A
└─ artifacts A

Worker B
├─ clone B
├─ branch B
├─ container B
├─ temp B
└─ artifacts B
```

격리 대상은 Source만이 아니다.

```text
working directory
git index
process
temp directory
port
disposable DB
test output
artifact path
```

예를 들어 Integration Test가 각각 PostgreSQL을 띄운다면 Worker별 DB 상태가 분리되어야 한다.

```text
Worker A
→ postgres-A

Worker B
→ postgres-B
```

같은 DB를 공유하면 한 Task가 만든 데이터가 다른 Task의 테스트 결과에 영향을 줄 수 있다.

---

## 7. Branch와 Container는 다른 문제를 해결한다

다음 구분을 기억하는 것이 좋다.

```text
Branch
→ Source 변경 격리

Container / VM
→ Runtime, Process, Filesystem 격리
```

예를 들어 Branch가 나뉘어 있어도 같은 개발 DB를 사용할 수 있다.

```text
Branch A
Branch B
   ↓
same DB
```

Task A가 테스트 데이터를 삭제하면 Task B가 실패할 수 있다.

반대로 Container를 나눴어도 두 Agent가 같은 핵심 파일을 수정하면 통합 시 충돌한다.

```text
Container A → UserService.java 수정
Container B → UserService.java 수정
```

실행 중에는 문제가 없다.

그러나 Merge 시점에는 충돌한다.

따라서 Cloud Task 격리는 다음 두 질문을 모두 본다.

```text
Source는 분리됐는가?
Runtime은 분리됐는가?
```

---

## 8. Artifact 경로도 Task별로 분리한다

여러 Runner가 같은 경로에 결과를 쓰면 마지막 실행 결과가 앞의 결과를 덮어쓸 수 있다.

좋지 않은 구조:

```text
artifacts/result.json
artifacts/build.log
```

권장:

```text
artifacts/AUTH-142/
├─ result.json
├─ junit.xml
└─ build.log

artifacts/ATTEND-211/
├─ result.json
├─ junit.xml
└─ build.log
```

또는 Run ID와 SHA를 사용할 수 있다.

```text
artifacts/AUTH-142/def456/run-03/
```

Artifact 내부에도 Git SHA를 기록한다.

이렇게 해야 8장에서 만든 Evidence가 어느 Source 상태의 결과인지 추적할 수 있다.

---

## 9. 물리적 격리가 논리적 충돌을 해결하지는 않는다

여러 Cloud Agent가 완전히 다른 Container에서 작업한다고 하자.

```text
Agent A → UserService 예외 구조 변경
Agent B → UserService 성능 개선
Agent C → UserService logging 변경
```

각 환경에서는 아무 충돌이 없다.

하지만 세 결과를 main에 합치는 순간 문제가 생긴다.

```text
PR A
PR B
PR C
  ↓
Merge Conflict
또는
Semantic Conflict
```

Git이 자동 Merge에 성공해도 의미적 충돌이 남을 수 있다.

예를 들어 A가 메서드 구조를 바꿨는데 B가 이전 구조를 전제로 최적화했다면 둘을 합친 코드가 의도대로 동작하지 않을 수 있다.

따라서 병렬화 전에 예상 변경 범위를 확인한다.

```text
Expected Files
Common DTO
Shared Module
Central Config
Migration
Schema
Public API
```

같은 핵심 영역을 건드리는 Task는 순차 처리하거나 먼저 Dependency를 정리하는 편이 낫다.

---

## 10. Migration과 Schema는 특별 취급한다

DB Migration은 일반 Source 파일보다 병렬화가 어렵다.

예:

```text
Agent A
→ V142__add_student_index.sql

Agent B
→ V142__add_attendance_status.sql
```

각 Branch에서는 문제가 없어 보일 수 있다.

하지만 Merge하면 Migration 번호가 충돌한다.

번호가 달라도 적용 순서 의존성이 생길 수 있다.

```text
V142 → column 생성
V143 → index 생성
```

두 Task의 순서가 바뀌면 실패할 수 있다.

Migration Task에서는 다음을 별도로 본다.

```text
번호 정책
적용 순서
Schema dependency
Rollback/upgrade validation
```

권장 흐름:

```text
Migration Task 동시성 제한
→ Merge 순서 확정
→ 통합 Migration Validation
```

Container를 나눴다고 Schema 변경의 통합 문제가 사라지는 것은 아니다.

---

## 11. Shared/Common Module은 병렬화 병목이 된다

`campus-platform`에 다음 Module이 있다고 하자.

```text
attendance
notification
student
common
```

처음에는 세 Task가 독립적으로 보일 수 있다.

```text
Task A → attendance
Task B → notification
Task C → student
```

그런데 세 Task 모두 `common`의 DTO를 바꿔야 한다면 실제로는 독립적이지 않다.

```text
Task A ─┐
Task B ─┼→ common DTO
Task C ─┘
```

이 경우 먼저 공통 변경을 별도 선행 Task로 분리하는 방법이 있다.

```text
Task 0
common DTO 변경
   ↓
새 Base SHA
   ↓
Task A / B / C 병렬 실행
```

병렬화 전에 `공통 영역을 수정하는가`를 확인해야 하는 이유다.

---

## 12. Multi-Repository도 최소 범위로 연결한다

하나의 기능이 여러 Repository에 걸쳐 있을 수 있다.

예:

```text
campus-api
campus-admin
campus-common
```

API와 Common DTO를 동시에 바꾸는 Task라면 두 Repository가 필요할 수 있다.

하지만 현재 Task와 관계없는 Repository까지 모두 연결하면 비용이 생긴다.

```text
checkout 증가
Context 증가
검색 범위 증가
잘못된 변경 가능성 증가
```

따라서 원칙은 단순하다.

> 현재 Task에서 함께 수정될 가능성이 높은 Repository만 Workspace에 제공한다.

예:

```text
Task
학생 프로필 API + DTO 변경

Repos
- campus-api
- campus-common
```

Admin UI가 이번 Task와 무관하다면 `campus-admin`은 넣지 않는다.

Multi-Repository Handoff는 13장에서 다시 연결한다.

---

## 13. Cloud Agent의 Commit은 Review 가능한 단위여야 한다

Cloud Agent가 Task 하나를 끝냈는데 다음이 한 Commit에 섞여 있다고 하자.

```text
Bug Fix
전체 formatting
dependency update
README 수정
불필요한 import 정리
```

기능은 맞더라도 Review 비용이 커진다.

Cloud Task의 결과는 가능하면 Task 범위와 맞춰 둔다.

```text
AUTH-142
→ expired token fix
→ related test
```

Commit과 Evidence를 연결한다.

```text
Commit: def456
Changed Files: 3
Validation: AuthServiceTest PASS
```

Unrelated formatting이나 광범위한 cleanup은 별도 Task로 분리한다.

작은 Task와 작은 Diff는 Cloud 결과를 Review하기 쉽게 만든다.

---

## 14. 기본 결과는 Merge가 아니라 PR이다

Cloud Worker가 코드를 수정했다고 바로 main에 합치는 것을 기본값으로 두지 않는다.

권장 흐름:

```text
Cloud Worker
→ Commit
→ Push
→ PR
→ CI / Evidence
→ Review
→ Merge
```

자동 Merge는 별도의 권한과 정책 문제다.

현재 책의 기본 모델은 Cloud Worker가 **검증 가능한 변경을 반환**하고 최종 Integration은 Local/Review 단계에서 처리하는 것이다.

이 구조가 13장의 Local → Cloud → Local Handoff와 연결된다.

---

## 15. campus-platform 격리 예제

세 작업이 있다고 하자.

```text
Task A
attendance test 보강

Task B
notification retry bug 수정

Task C
student profile UI 수정
```

작업 공간:

```text
Task A
branch: agent/attendance-test
workspace: worker-a
artifacts: artifacts/task-a/

Task B
branch: agent/notification-retry
workspace: worker-b
artifacts: artifacts/task-b/

Task C
branch: agent/student-ui
workspace: worker-c
artifacts: artifacts/task-c/
```

각 Task가 다른 Module과 파일을 사용한다면 병렬로 진행하기 쉽다.

하지만 작업 중 세 Task 모두 `common-student.dto` 변경이 필요하다는 사실이 드러났다고 하자.

그 순간 병렬 전략을 다시 본다.

```text
공통 DTO 변경을 선행 Task로 분리
→ 새 Base SHA 생성
→ 나머지 Task 재실행
```

Cloud Task는 실행 중에도 재분류될 수 있다.

---

## 16. 격리 체크리스트

Cloud Task를 여러 개 실행하기 전에 다음을 확인한다.

```text
Base SHA가 명확한가?
Task별 Branch가 있는가?
Working Directory가 분리됐는가?
Runtime Process가 분리됐는가?
Port가 충돌하지 않는가?
DB/Test Data가 격리됐는가?
Artifact Path가 분리됐는가?
공통 파일/DTO/Schema를 동시에 수정하지 않는가?
```

모든 항목을 Container 하나로 해결하려 하지 않는다.

Source, Runtime, Data, Artifact, Integration을 각각 본다.

---

## 17. 이 장에서 기억할 것

Cloud Agent 여러 개를 띄우는 것은 어렵지 않을 수 있다.

문제는 각 Worker가 **어떤 상태에서 시작했고, 무엇을 바꿨고, 어떤 결과를 만들었는지 서로 섞이지 않게 하는 것**이다.

기본 구조는 다음과 같다.

```text
Task
→ Base SHA
→ Branch
→ Independent Workspace
→ Isolated Runtime
→ Commit
→ Evidence
→ PR
```

그리고 한 가지 제한이 남는다.

작업공간을 완전히 나눠도 같은 파일과 같은 Schema를 수정하면 통합 비용은 그대로 발생한다.

> Branch는 Source를 격리하고 Container는 Runtime을 격리한다.

> 같은 파일과 같은 Schema를 수정하는 Task는 격리보다 먼저 병렬화 여부를 다시 판단한다.

다음 장에서는 격리된 Task를 몇 개까지 동시에 실행할 것인지, Agent 수가 늘어날수록 Context·Merge·Review 비용이 어떻게 증가하는지 살펴본다.
