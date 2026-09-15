# 11장 설계 - Git, Branch, Worktree, Container로 작업 격리하기

## 장의 목표

Git을 단순한 형상관리 도구가 아니라 Local과 Cloud 사이의 작업 전달 경계이자 여러 Cloud Worker가 서로 간섭하지 않도록 하는 격리 수단으로 설명한다.

핵심 질문:

> 여러 Cloud Agent가 동시에 작업해도 서로의 파일, Git 상태, 테스트 결과를 오염시키지 않게 하려면 어떤 단위로 격리해야 하는가?

---

## 핵심 주장

> Cloud Agent 시대에는 Git이 개발자와 Remote Worker 사이의 작업 전달 프로토콜 역할까지 수행한다.

> Cloud Task 하나에는 가능한 한 하나의 명확한 Base SHA와 하나의 독립 Branch를 연결한다.

> 작업 공간을 분리해도 논리적 충돌까지 사라지는 것은 아니다. 같은 파일과 같은 Schema를 동시에 바꾸는 Task는 처음부터 병렬화하지 않는 편이 낫다.

---

## 독자가 얻는 것

- Git을 Local↔Cloud Handoff Boundary로 사용할 수 있다.
- Task ID, Session ID, Branch, Commit SHA, Test Result, PR을 하나의 작업 상태로 연결할 수 있다.
- Local의 Git Worktree와 Cloud의 Independent Clone/Container 차이를 설명할 수 있다.
- Task Branch 기준으로 여러 Worker를 격리할 수 있다.
- 동일 Working Directory 공유로 생기는 충돌을 피할 수 있다.
- Shared File, Migration, Schema 같은 논리적 충돌을 사전에 식별할 수 있다.
- Cloud 결과를 Commit/PR 단위로 회수할 수 있다.

---

# 절 구성

## 11.1 Git은 Cloud Handoff Boundary다

기본 흐름:

```text
Local Developer
→ 수정 / Test
→ Commit
→ Push
      ↓
Git Repository
      ↓
Cloud Worker
→ Branch
→ Work / Test
→ Commit
→ Push
→ PR
```

Cloud Worker가 Local IDE의 임시 상태를 직접 공유하는 것이 아니라, 명시적인 Git 상태를 기준으로 작업하도록 한다.

좋은 Task Input:

```text
repository: campus-platform
base_sha: abc123
branch: agent/task-142
```

좋지 않은 Input:

```text
내 Mac에서 지금 수정 중인 상태 기준으로 이어서 해줘.
```

Cloud에 보내려면 전달 가능한 상태로 먼저 정리한다.

---

## 11.2 Base SHA를 고정한다

Task 시작점이 불명확하면 결과의 재현성과 Review가 어려워진다.

권장:

```text
Task #142
base_sha: abc123
branch: agent/task-142
```

검증 결과도 같은 SHA 계열과 연결한다.

```text
Task Input SHA
→ Agent Commit
→ Runner Verification SHA
→ PR
```

서로 다른 SHA에서 생성된 테스트 결과를 같은 Evidence로 섞지 않는다.

---

## 11.3 Branch per Task

예:

```text
Task #142
→ agent/task-142

Task #143
→ agent/task-143
```

장점:

- 변경 범위 추적
- Commit 단위 Review
- 독립 Retry
- 작업 취소/폐기 용이
- PR 연결 용이

Branch 이름 자체보다 `Task와 Branch의 1:1 대응이 명확하다`는 점이 중요하다.

---

## 11.4 Task State를 Git 상태와 연결한다

권장 상태 후보:

```text
Task ID
Cloud Session ID
Base SHA
Branch
Current Commit SHA
Status
Test Result
Artifact Path
PR
```

예:

```text
Task: task-142
Session: cloud-142
Base: abc123
Branch: agent/task-142
Commit: def456
Status: verifying
Test: PASS
PR: #142
```

이 정보가 있으면 여러 Cloud Worker가 동시에 실행돼도 현재 상태를 추적하기 쉽다.

---

## 11.5 Local Worktree

Local에서 여러 Task를 동시에 유지해야 할 때 Git Worktree를 사용할 수 있다.

```text
repo/
worktrees/
├─ task-a/
├─ task-b/
└─ task-c/
```

장점:

- 같은 Repository object database 재사용
- 작업 디렉터리 분리
- Branch 분리

주의:

- shared external runtime
- 같은 DB/port
- 공통 cache
- 임시 파일 위치

Worktree만 분리했다고 Runtime까지 격리되는 것은 아니다.

---

## 11.6 Cloud에서는 Independent Clone/Container가 단순하다

Cloud Worker는 다음처럼 독립 공간을 갖는 편이 이해하기 쉽다.

```text
Worker A
- clone A
- branch A
- container A
- artifacts A

Worker B
- clone B
- branch B
- container B
- artifacts B
```

격리 대상:

- working directory
- git index
- temp directory
- test output
- process
- port
- disposable DB
- artifact path

---

## 11.7 Branch와 Container는 다른 문제를 해결한다

```text
Branch
→ Source 변경 격리

Container/VM
→ Runtime/Process/Filesystem 격리
```

둘 중 하나만으로 모든 충돌을 막을 수 없다.

예:

```text
Branch A / Branch B
but
same local DB
```

이면 테스트 데이터 충돌이 생길 수 있다.

Cloud Task는 Source와 Runtime 두 경계를 모두 고려한다.

---

## 11.8 Artifact 경로도 Task별로 분리한다

좋지 않은 구조:

```text
artifacts/result.json
```

여러 Worker가 덮어쓸 수 있다.

권장:

```text
artifacts/task-142/
artifacts/task-143/
```

또는 SHA/Run ID를 함께 사용한다.

Artifact에는 Git SHA를 기록해 결과와 Source를 연결한다.

---

## 11.9 같은 파일을 여러 Agent가 수정하면 격리만으로 해결되지 않는다

예:

```text
Agent A → UserService.java 구조 변경
Agent B → UserService.java 오류 처리 변경
Agent C → UserService.java 성능 개선
```

모두 별도 Branch에 있어도 Merge Conflict와 의미적 충돌이 커진다.

따라서 병렬화 전에 변경 Scope를 확인한다.

- 예상 파일
- 공통 DTO
- common module
- central configuration
- migration/schema

같은 핵심 파일을 공유하면 순차 처리 또는 Task 재분해를 우선한다.

---

## 11.10 Migration과 Schema는 특별 취급한다

Migration은 파일 번호 충돌뿐 아니라 적용 순서 의존성이 있다.

예:

```text
Agent A → V142__add_student.sql
Agent B → V142__add_attendance.sql
```

Branch에서는 문제없어 보여도 통합 시 충돌할 수 있다.

권장:

- migration task는 동시성 제한
- 중앙 번호 정책 사용
- merge 전 통합 migration validation

DB Schema 공동 변경은 12장의 Dependency-aware fan-out과 연결한다.

---

## 11.11 Shared/Common Module은 병렬화 병목이 될 수 있다

예:

```text
attendance
notification
student
```

세 Task가 모두 `common`을 수정한다면 실제로는 독립 Task가 아니다.

따라서 `공통 모듈을 수정해야 하는지`를 Task Routing 시 확인한다.

가능하면 각 기능 변경을 module-local하게 유지한다.

---

## 11.12 Multi-Repository는 필요한 Repository만 연결한다

Multi-Repo 예:

```text
campus-api
campus-admin
campus-common
```

Task가 API와 Common DTO를 함께 바꿔야 한다면 두 Repository가 필요할 수 있다.

그러나 현재 Task와 무관한 Repository까지 모두 연결하면:

- checkout 증가
- Context 증가
- 탐색 범위 증가
- 잘못된 변경 가능성 증가

원칙:

> 현재 Task에서 함께 수정될 가능성이 높은 Repository만 제공한다.

13장에서 Handoff 관점으로 다시 사용한다.

---

## 11.13 Commit Policy

Cloud Agent 결과는 가능하면 Review 가능한 Commit으로 남긴다.

권장:

- Task 범위 밖 변경 최소화
- generated file 여부 명시
- test 결과와 Commit SHA 연결
- unrelated formatting 금지

좋지 않은 결과:

```text
Bug Fix + 전체 formatting + dependency update
```

권장:

```text
Bug Fix Commit
→ Verification
```

---

## 11.14 Merge는 Cloud Worker의 기본 책임으로 두지 않는다

기본 흐름:

```text
Cloud Worker
→ Commit
→ Push
→ PR
→ CI/Evidence
→ Review
→ Merge
```

자동 Merge는 별도 정책과 권한이 필요한 문제다.

현재 책에서는 기본값을 `PR 반환`으로 둔다.

---

## 11.15 campus-platform 예제

```text
Task A
attendance test
branch: agent/attendance-test
container: worker-a

Task B
notification retry
branch: agent/notification-retry
container: worker-b

Task C
student profile UI
branch: agent/student-ui
container: worker-c
```

각 작업이 서로 다른 module/file scope라면 병렬 진행한다.

반대로 세 Task가 모두 `common-student.dto`를 바꿔야 한다면 dependency를 먼저 정리한다.

---

# 좋은 사례와 나쁜 사례

## 같은 Working Directory 공유

좋지 않은 방식:

```text
Agent A + Agent B
→ same directory
→ same git index
```

권장:

```text
Task A → branch/workspace A
Task B → branch/workspace B
```

## Branch만 분리

좋지 않은 방식:

```text
branch 분리
but same DB / same temp path
```

권장:

```text
Source + Runtime + Artifact 경로 분리
```

## 무관한 Repository 전부 제공

좋지 않은 방식:

```text
Task 하나
→ 조직 전체 Repository 연결
```

권장:

```text
필요 Repository만 연결
```

---

# 필요한 그림

1. Local → Git → Cloud Handoff
2. Task/Session/Branch/SHA/PR 상태 모델
3. Branch vs Runtime Isolation
4. Parallel Worker Workspace
5. Migration Conflict 사례

---

# Phase 6 구현 후보

```text
tasks/task-142.yaml
scripts/create-task-branch.sh
scripts/verify-base-sha.sh
artifacts/<task-id>/
```

Task 상태 예:

```yaml
task_id: task-142
base_sha: abc123
branch: agent/task-142
status: verifying
```

---

# 앞뒤 장 연결

10장:
Runner/Agent 실행 경로 정의

11장:
각 Task 실행공간과 Git 상태 격리

12장:
여러 격리 Task를 실제로 fan-out하고 비용을 관리

13장:
Git을 Local↔Cloud 전체 Handoff 흐름으로 확장

---

# 의도적으로 다루지 않을 내용

- Git 입문
- 고급 Git 내부 구조
- 범용 Merge Queue 제품 비교
- 자동 Merge Governance

---

# 장의 결론 메시지

> Cloud Agent 시대에는 Git이 개발자와 Remote Worker 사이의 작업 전달 프로토콜 역할까지 수행한다.

> Branch는 Source를 격리하고 Container는 Runtime을 격리한다.

> 같은 파일과 같은 Schema를 수정하는 Task는 격리보다 먼저 병렬화 여부를 다시 판단한다.
