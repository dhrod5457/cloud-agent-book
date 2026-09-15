# 13장 설계 - Local → Cloud → Local Handoff

## 장의 목표

Local Agent와 Cloud Agent를 서로 대체 관계가 아니라 하나의 개발 흐름 안에서 단계별로 역할을 나누는 실행 모델로 연결한다.

핵심 질문:

> 하나의 기능 개발이 Local과 Cloud 사이를 어떻게 이동하고, 어느 지점에서 다시 Local로 돌아와야 하는가?

---

## 핵심 주장

> Task는 Local 또는 Cloud 중 하나에 영구적으로 속하는 것이 아니다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

> Local은 설계·탐색·내부망·통합에 강하고, Cloud는 독립 실행·장시간 검증·병렬 처리에 강하다.

> Handoff Boundary는 Git과 Evidence다.

---

## 독자가 얻는 것

- Local→Cloud→Local 기본 Workflow를 설계할 수 있다.
- Architecture/Task Split을 Local에 남기고 실행/검증을 Cloud에 위임할 수 있다.
- 내부망 검증이 필요한 작업을 Hybrid로 구성할 수 있다.
- Git Commit/Branch를 Handoff 입력으로 사용할 수 있다.
- Cloud 결과를 Evidence/PR로 Local에 회수할 수 있다.
- Multi-Repository 작업에서 필요한 Repository만 Cloud Workspace에 연결할 수 있다.
- Local과 Cloud 사이의 책임 경계를 명확히 할 수 있다.

---

# 절 구성

## 13.1 Local과 Cloud는 작업 단계에 따라 바뀐다

예:

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

한 Task 안에서도 실행 위치가 바뀔 수 있다.

---

## 13.2 Local의 역할

Local이 유리한 단계:

- 요구사항 해석
- Architecture 판단
- 넓은 Context 탐색
- 빠른 질문/수정 반복
- 내부망/VPN 접근
- 사내 DB/HSM/Jenkins 확인
- 최종 Review/Integration

Local의 강점은 사람과 Agent의 짧은 Feedback Loop다.

---

## 13.3 Cloud의 역할

Cloud가 유리한 단계:

- 장시간 Build/Test
- 독립 Feature 작업
- 반복 Refactoring
- E2E
- Docker Build
- CI Failure Fix
- 여러 독립 검증의 병렬 실행

Cloud의 강점은 개발자 PC와 분리된 실행환경과 비동기성이다.

---

## 13.4 Handoff 전에 Local에서 준비할 것

Cloud에 넘기기 전 최소 준비:

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

좋지 않은 방식:

```text
로컬에서 하던 것 이어서 해줘.
```

권장:

```text
base_sha=abc123
Task=AuthService expired token fix
Validation=./gradlew test --tests AuthServiceTest
```

---

## 13.5 Git이 Handoff Boundary다

```text
Local
→ Commit / Push
      ↓
Git
      ↓
Cloud Worker
```

Cloud 결과도 Git으로 돌아온다.

```text
Cloud Worker
→ Commit
→ Push
→ PR
      ↓
Local Review
```

11장의 Git isolation을 전체 Workflow에 적용한다.

---

## 13.6 Evidence가 Return Boundary다

Cloud Worker가 반환할 것:

- Commit SHA
- Changed Files
- Build Result
- Test Result
- Artifact
- Screenshot/Video
- PR

Local에서 처음 보는 것은 Agent 설명보다 Evidence다.

```text
Cloud Result
→ Evidence 확인
→ 필요한 Diff 확인
→ Internal Validation
```

---

## 13.7 내부망 검증을 Local에 남긴다

예:

```text
Cloud
→ PostgreSQL/Testcontainers validation
→ Unit/Integration
→ Docker Build
      ↓
Local
→ Tibero 실제 DB
→ HSM
→ Jenkins
→ Internal API
```

Cloud에 없는 자원을 억지로 복제하지 않고 Handoff로 해결한다.

---

## 13.8 Hybrid Task를 단계로 분해한다

예: DB Migration

```text
Local
→ 요구사항/Schema 영향 분석
      ↓
Cloud
→ migration 작성
→ disposable PostgreSQL validation
      ↓
Local
→ Tibero 적용 검증
```

예: 카드/HSM 연동

```text
Local
→ HSM protocol/constraint 확인
      ↓
Cloud
→ pure logic/unit test
      ↓
Local
→ 실제 HSM integration
```

---

## 13.9 Cloud에서 실패하면 바로 Local로 가져오지 않는다

먼저 다음을 시도한다.

```text
Runner FAIL
→ Result Gateway
→ 작은 Context
→ Cloud Agent Fix
→ Runner 재검증
```

Local로 되돌릴 조건:

- 내부망 필요
- Context가 예상보다 커짐
- 재현 실패
- 동일 Failure 반복
- Budget 소진
- Architecture 판단 필요

---

## 13.10 Local Fallback은 실패가 아니다

Cloud에서 적합하지 않은 Task를 Local로 돌리는 것은 정상 Routing이다.

```text
Cloud Attempt
→ Internal dependency 발견
→ Local Fallback
```

중요한 것은 `끝까지 Cloud에서 해결하는 것`이 아니라 전체 Workflow 비용을 줄이는 것이다.

---

## 13.11 Multi-Repository Handoff

예:

```text
campus-api
campus-admin
campus-common
```

학생 프로필 변경이 API/Common/UI에 걸치면 필요한 Repository만 Cloud Workspace에 포함한다.

```text
Task Scope
→ campus-api
→ campus-common
→ campus-admin
```

무관한 Repository는 제외한다.

---

## 13.12 Multi-Repo에서 Base Version을 고정한다

각 Repository의 기준점을 기록한다.

```text
campus-api: abc123
campus-common: def456
campus-admin: ghi789
```

결과도 각 Commit으로 반환한다.

Cross-repo 변경의 Review 순서와 의존성은 Local Integration 단계에서 확인한다.

---

## 13.13 Handoff 상태 모델

예:

```text
LOCAL_ANALYSIS
READY_FOR_CLOUD
CLOUD_RUNNING
CLOUD_VERIFYING
CLOUD_DONE
LOCAL_VALIDATION
READY_TO_MERGE
```

복잡한 Workflow Engine 구현이 목적은 아니다.

사람과 Agent가 현재 Task가 어디에 있는지 구분할 최소 상태만 정의한다.

---

## 13.14 Developer Blocking Time을 줄이는 Handoff

좋은 흐름:

```text
Developer
→ Cloud Task 위임
→ 다음 Feature 작업
→ 결과 도착 후 Review
```

나쁜 흐름:

```text
Developer
→ Cloud Task 위임
→ 계속 진행상황 확인
→ 질문 대기
→ 사실상 동기 작업
```

Cloud의 비동기성을 실제로 활용한다.

---

## 13.15 campus-platform Hybrid Workflow

```text
Local
→ Attendance API 설계
→ Task Contract 작성
→ Commit / Push
      ↓
Cloud #1
→ Unit Test

Cloud #2
→ Integration Test

Cloud #3
→ Docker Build

Cloud #4
→ Admin E2E
      ↓
Evidence / PR
      ↓
Local
→ Tibero 최종 검증
→ 내부 Jenkins
→ Review
→ Merge
```

---

# 좋은 사례와 나쁜 사례

## 모든 것을 Cloud에 복제

좋지 않은 방식:

```text
Tibero/HSM/Jenkins 내부환경까지 Cloud에 억지로 복제
```

권장:

```text
Cloud 가능한 검증
→ Local 내부 검증
```

## Cloud 결과를 자연어로만 회수

좋지 않은 방식:

```text
"완료했습니다"
```

권장:

```text
Commit + Test + Artifact + PR
```

## Local/Cloud 고정 역할

좋지 않은 생각:

```text
이 기능은 무조건 Local
이 기능은 무조건 Cloud
```

권장:

```text
Task 단계별 Routing
```

---

# 필요한 그림

1. Local → Cloud → Local 기본 흐름
2. Git Handoff / Evidence Return Boundary
3. Hybrid Internal Validation
4. Multi-Repo Handoff
5. Local Fallback Decision

---

# Phase 6 구현 후보

Task 상태 예:

```yaml
status: READY_FOR_CLOUD
base_sha: abc123
cloud_branch: agent/task-142
local_validation:
  - tibero
  - hsm
```

---

# 앞뒤 장 연결

12장:
병렬 Worker의 실행과 통합 비용

13장:
병렬 결과를 Local 개발 흐름과 연결

14장:
사람이 직접 Handoff하지 않아도 CI/Issue/Review 이벤트가 Task를 생성

---

# 의도적으로 다루지 않을 내용

- VPN 구축 방법
- 사내 네트워크 입문
- HSM 제품 사용법
- 범용 Workflow Engine

---

# 장의 결론 메시지

> Task는 Local 또는 Cloud에 영구적으로 속하지 않는다.

> Git으로 작업을 넘기고 Evidence로 결과를 돌려받는다.

> Cloud에서 할 수 없는 마지막 검증은 Local로 Handoff하면 된다.
