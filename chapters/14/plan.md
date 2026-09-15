# 14장 설계 - Task Queue와 Event-driven Cloud Agent

## 장의 목표

Cloud Agent를 개발자가 터미널에서 직접 시작하는 도구로만 보지 않고 Issue, CI Failure, PR Review, Scheduled Test, Dependency Update 같은 이벤트가 Task를 만들고 필요할 때만 Worker를 실행하는 구조로 설명한다.

핵심 질문:

> 개발자의 PC가 Cloud Agent 실행의 시작점이 아니어도 되게 하려면 어떤 이벤트와 Task Queue가 필요한가?

---

## 핵심 주장

> 이벤트가 없으면 Agent도 실행하지 않는다.

> CI의 정상 경로에는 LLM이 필요하지 않다. 실패 분석과 수정이 필요한 순간에만 Cloud Agent를 호출한다.

> Cloud Agent는 상시 프로세스보다 이벤트에 반응하는 일시적 Remote Worker에 가깝다.

---

## 독자가 얻는 것

- Issue/CI/Review/Schedule을 Cloud Task Source로 사용할 수 있다.
- Task Queue와 Worker Pool을 간단한 실행 모델로 이해할 수 있다.
- CI PASS 경로에서 Agent 호출을 제거할 수 있다.
- CI Failure → Agent Fix → Runner Revalidation → PR 흐름을 설계할 수 있다.
- Review Comment와 Nightly Failure를 동일 패턴으로 처리할 수 있다.
- Dependency Update를 Runner-first로 연결할 수 있다.
- 이벤트 중복과 무한 반복을 막는 기본 조건을 설계할 수 있다.

---

# 절 구성

## 14.1 Cloud Task의 시작점은 사람만이 아니다

Task Source 후보:

- Issue
- Git Push
- CI Failure
- PR Review Comment
- Scheduled Test
- Nightly Build
- Dependency Update
- Security/Static Analysis Failure
- Human Escalation

구조:

```text
Event Source
    ↓
Task Queue
    ↓
Task Classification
    ↓
Runner / Cloud Agent
```

이 장에서는 범용 Queue 시스템 구현보다 개발 Workflow 관점의 Task 생성 규칙에 집중한다.

---

## 14.2 Git Push → CI → Cloud Agent

기본 패턴:

```text
Git Push
   ↓
  CI
   ↓
+--+--+
|     |
PASS FAIL
|     |
Done  Failure Summary
        ↓
     Cloud Agent
        ↓
       Fix
        ↓
      Runner
        ↓
       PASS
        ↓
       PR
```

PASS에서는 Agent가 호출되지 않는다.

---

## 14.3 PR Review Comment → Task

Review Comment가 수정 요청이라면 Cloud Task로 변환할 수 있다.

예:

```text
Reviewer:
"이 예외 처리에 회귀 테스트를 추가해주세요."
      ↓
Task
- PR SHA
- Review Comment
- Relevant Files
- Validation
      ↓
Cloud Agent
```

모든 Comment를 자동 실행하지 않는다.

다음 조건을 확인한다.

- 수정 요청인지
- 권한 있는 Reviewer인지
- Task Scope가 명확한지
- 기존 PR 범위 안인지

---

## 14.4 Nightly Test Failure

```text
Nightly Scheduler
      ↓
Full Test Runner
      ↓
PASS → 종료
FAIL → Result Gateway
      ↓
Cloud Agent 후보
```

적합한 경우:

- 재현 가능한 회귀
- 실패 테스트가 명확함
- 수정 범위가 제한됨

부적합:

- 인프라 장애
- 외부 서비스 outage
- flaky 여부 불명확

먼저 Failure Classification을 사용한다.

---

## 14.5 Dependency Update

```text
Dependency Bot / Schedule
      ↓
Version Update
      ↓
Runner
- Build
- Test
      ↓
PASS → PR
FAIL → Agent
      ↓
Compatibility Fix
      ↓
Runner
```

Agent에게 처음 전달할 정보:

- package name
- old/new version
- failed command
- failed tests
- relevant files

---

## 14.6 Issue → Cloud Task

Issue가 곧바로 Agent Task가 되는 것은 아니다.

좋은 Issue Task:

```text
Goal 명확
Reproduction 있음
Scope 추정 가능
Validation 가능
```

좋지 않은 Issue:

```text
"시스템 전체가 좀 느립니다. 개선해주세요."
```

이 경우 Local 분석/Task 분해가 먼저 필요하다.

---

## 14.7 Task Queue의 최소 정보

```text
Task ID
Source Event
Repository
Base SHA
Priority
Task Type
Environment
Status
Budget
```

예:

```text
id: ci-91
source: ci_failure
repo: campus-platform
sha: abc123
execution: cloud-agent
status: queued
```

이 장에서 복잡한 Queue 제품을 요구하지 않는다.

GitHub Issue/PR, CI metadata, 간단한 상태 저장만으로도 패턴을 설명할 수 있다.

---

## 14.8 Worker Pool

개념:

```text
Task Queue
    ↓
+---------+---------+---------+
|         |         |         |
Worker A Worker B Worker C
```

Worker는 Task가 있을 때 생성하거나 READY pool에서 할당할 수 있다.

9장의 Warm Worker는 짧고 빈번한 Task에 사용할 수 있다.

---

## 14.9 Event → Runner → Agent 순서를 유지한다

이벤트가 발생했다고 무조건 Agent를 깨우지 않는다.

예:

```text
Dependency Update Event
→ Runner
→ PASS
→ 종료
```

다음처럼 실패와 판단이 필요한 경우에만 Agent를 호출한다.

```text
Runner FAIL
→ Result Gateway
→ Code Reasoning Required
→ Agent
```

---

## 14.10 이벤트 중복을 제어한다

같은 실패가 여러 이벤트를 만들 수 있다.

예:

```text
CI retry
PR synchronize
review update
```

같은 SHA와 같은 Failure Fingerprint라면 중복 Task를 막을 수 있다.

Task Dedup Key 후보:

```text
repo + sha + event_type + failure_fingerprint
```

범용 분산 Queue 설계까지 확장하지 않는다.

---

## 14.11 Agent가 자기 자신을 무한 호출하지 않게 한다

예:

```text
Agent Push
→ CI FAIL
→ Agent Fix
→ Push
→ CI FAIL
→ Agent Fix
→ ...
```

제어:

- max retry
- max token/cost
- same fingerprint stop
- changed failure 확인
- Human Escalation

8장의 Budget/Fingerprint를 Event-driven Loop에 적용한다.

---

## 14.12 Draft PR을 기본 결과로 사용할 수 있다

자동 수정 결과를 바로 Merge하지 않는다.

```text
Event
→ Agent Fix
→ Runner PASS
→ Draft PR
→ Human Review
```

상황에 따라 기존 PR에 Commit을 Push할 수도 있다.

핵심은 Merge 권한보다 `검증 가능한 결과를 Review 가능한 형태로 반환`하는 것이다.

---

## 14.13 PR Follow-up Session

하나의 PR은 여러 이벤트를 거칠 수 있다.

```text
PR 생성
→ CI
→ Review Comment
→ 수정
→ CI 재실행
```

각 이벤트마다 Repository 전체를 다시 이해시키지 않도록 다음을 유지할 수 있다.

- PR
- Branch
- Base/Current SHA
- Task Summary
- Relevant files
- previous Evidence

장기 Agent Memory Architecture로 확장하지 않고 PR Task Context 재사용 관점으로만 다룬다.

---

## 14.14 Event-driven Agent의 비용 효과

비교:

```text
Always-on Agent
→ idle 시간에도 session 유지
```

```text
Event-driven
→ 이벤트 발생
→ 필요한 Task만 실행
→ 종료
```

비용 관점:

- Agent invocation 감소
- Idle session 감소
- 성공 경로 LLM 제거
- Developer monitoring 감소

---

## 14.15 campus-platform 이벤트 예제

### CI Failure

```text
PR #142
→ runner-integration FAIL
→ AuthServiceTest failure
→ Cloud Agent
→ Fix
→ runner-integration PASS
→ PR update
```

### Nightly

```text
02:00 Full E2E
→ 3 FAIL
→ 1 infra / 2 reproducible
→ reproducible 2건만 Agent Task 생성
```

### Dependency Update

```text
Spring dependency update
→ build/test
→ compile failure
→ Agent compatibility fix
```

---

# 좋은 사례와 나쁜 사례

## 모든 Event에서 Agent 호출

좋지 않은 방식:

```text
Push → Agent
CI → Agent
PASS → Agent
```

권장:

```text
Event → deterministic path → 필요 시 Agent
```

## 같은 실패 무한 반복

좋지 않은 방식:

```text
same fingerprint
→ 계속 Agent 재호출
```

권장:

```text
Budget/Fingerprint Check
→ Escalation
```

## 자동 Merge까지 한 번에

기본 권장:

```text
Agent → Evidence → PR → Review
```

---

# 필요한 그림

1. Event Source → Task Queue → Worker
2. CI PASS/FAIL Agent Activation
3. Review Comment Follow-up
4. Nightly Test Failure
5. Retry/Fingerprint Loop Guard

---

# Phase 6 구현 후보

```text
tasks/events/
scripts/create-task-from-ci.py
scripts/deduplicate-task.py
```

Task 예:

```json
{
  "source": "ci_failure",
  "sha": "abc123",
  "failure": "AuthServiceTest.expiredToken"
}
```

---

# 필요한 조사

- GitHub/Copilot cloud agent scheduled/event-driven 현재 기능
- Claude Code Web PR auto-fix/review comment 현재 기능
- GitHub Actions event/CI metadata

현재 제품 기능은 research에서 기준일과 출처를 관리한다.

---

# 앞뒤 장 연결

13장:
사람이 Local↔Cloud로 Task를 Handoff

14장:
이벤트도 동일한 Cloud Task를 생성

15장:
두 방식을 campus-platform 운영 모델에 통합

---

# 의도적으로 다루지 않을 내용

- 범용 Message Queue 설계
- Kafka/RabbitMQ 입문
- Agent Platform Scheduler
- 자동 Merge Governance

---

# 장의 결론 메시지

> 이벤트가 없으면 Agent도 실행하지 않는다.

> 정상 CI 경로는 Runner가 처리하고 실패와 판단이 필요한 순간에만 Cloud Agent를 호출한다.

> Event-driven Cloud Agent는 개발자의 PC가 실행 시작점일 필요를 줄인다.
