# 2장. Local Agent와 Cloud Agent

Cloud Agent를 도입할 때 가장 먼저 생기는 질문은 어떤 제품이 더 좋은가가 아니다.

더 먼저 결정해야 하는 것은 **이 Task를 어디에서 실행할 것인가**다.

같은 계열의 Coding Agent를 사용하더라도 개발자 PC에서 실행되는 경우와 별도 Cloud 환경에서 실행되는 경우에는 조건이 달라진다.

Local에서는 현재 Workspace, 미커밋 파일, VPN, 내부 DB 같은 자원에 바로 접근할 수 있다. Cloud에서는 개발자 PC와 분리된 환경에서 장시간 작업을 실행하거나 여러 Task를 병렬로 처리하기 쉽다.

따라서 이 책에서는 Local Agent와 Cloud Agent를 경쟁 관계로 설명하지 않는다.

> Cloud Agent는 Local Agent를 대체하는 것이 아니다.

> Local에서는 설계와 통합을 하고, Cloud에서는 독립적인 작업을 병렬로 처리한다.

이 문장은 절대 규칙이 아니다. Task의 특성을 판단하기 위한 기본값이다.

---

## 1. Local과 Cloud의 차이는 모델보다 실행 위치에서 시작한다

개발자가 MacBook에서 Coding Agent를 실행하고 있다고 하자.

Local Agent는 현재 개발환경을 그대로 사용할 수 있다.

```text
Developer Mac
├─ Repository
├─ IDE
├─ Local Agent
├─ Git Working Tree
├─ Docker
├─ Database
├─ VPN
└─ Internal Tools
```

현재 수정 중인 파일도 바로 읽을 수 있다.

```text
modified: AuthService.java
untracked: debug-config.yml
```

로컬에서만 접근 가능한 DB나 내부 API가 있다면 같은 네트워크 조건을 사용할 수도 있다.

Cloud Agent는 다르다.

```text
Cloud Environment
├─ Repository Checkout
├─ Branch
├─ CPU / RAM / Disk
├─ Development Tools
└─ Agent
```

Cloud Agent는 보통 Local Working Directory를 그대로 공유하지 않는다. Git Repository나 별도 Workspace를 기준으로 작업을 시작한다.

이 차이가 Task Routing을 결정한다.

예를 들어 다음 작업은 Local에 가깝다.

```text
현재 미커밋 코드와 로컬 DB 상태를 같이 보면서
간헐적으로 발생하는 오류를 조사한다.
```

반대로 다음 작업은 Cloud에 보내기 쉽다.

```text
Base SHA: abc123
Task: AuthService expired token 수정
Validation: ./gradlew test --tests AuthServiceTest.expiredToken
```

두 작업의 차이는 AI 모델이 아니다.

입력 상태와 실행환경이 얼마나 독립적으로 전달 가능한가의 차이다.

---

## 2. Local Agent가 잘하는 일

Local Agent의 가장 큰 장점은 개발자의 현재 작업 흐름과 가깝다는 점이다.

다음과 같은 상황을 생각해 보자.

```text
Developer:
인증 모듈을 OAuth 기준으로 바꿀지 기존 JWT 구조를 유지할지 먼저 비교해보자.

Agent:
현재 auth 모듈과 gateway 연동을 보면 두 가지 선택지가 있습니다.

Developer:
DB schema는 건드리지 않는 쪽으로 다시 보자.
```

이 작업은 중간 판단이 계속 바뀐다.

개발자는 Agent에게 질문하고, 결과를 보고, 방향을 수정한다.

이런 Task는 Cloud에 던져 두고 비동기로 기다리는 방식보다 Local에서 짧은 feedback loop를 유지하는 편이 자연스럽다.

Local에 적합한 대표 조건은 다음과 같다.

- 요구사항이 아직 불명확함
- Architecture 판단 비중이 큼
- 여러 모듈을 동시에 넓게 봐야 함
- 개발자와 질문/수정이 반복됨
- 현재 미커밋 상태를 사용해야 함
- 로컬 파일이나 디바이스에 의존함
- VPN이나 내부 시스템 접근이 필요함
- 최종 Integration과 Review가 필요함

예를 들어 대학 시스템에서 HSM 연동 오류를 분석한다고 하자.

문제가 실제 HSM 장비, 내부 AP 서버, VPN 구간에 걸쳐 있다면 Cloud 환경에서 Source Code만 받아서는 재현하기 어렵다.

```text
Local / Internal
→ HSM
→ Internal AP
→ Tibero
→ Jenkins
```

이런 경우 Cloud Agent 성능이 좋아도 실행 위치 자체가 맞지 않는다.

---

## 3. Cloud Agent가 잘하는 일

Cloud Agent는 Task가 명확하고 독립적으로 검증할 수 있을수록 유리하다.

예를 들어 다음 Task를 보자.

```text
Task
AuthService expired token 처리 수정

Failure
expected: 401
actual: 200

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

이 Task는 다음 특징을 가진다.

```text
Scope가 명확함
완료 조건이 있음
독립 테스트 가능
Git으로 전달 가능
중간 질문이 적음
```

Cloud Agent는 이런 작업을 별도 Workspace에서 수행하기 쉽다.

또 장시간 실행 작업을 개발자 PC에서 분리할 수 있다.

```text
Cloud Worker #1
→ 전체 Unit Test

Cloud Worker #2
→ Integration Test

Cloud Worker #3
→ Web E2E

Cloud Worker #4
→ Docker Build
```

이 작업이 실행되는 동안 개발자는 다음 기능을 계속 개발할 수 있다.

Cloud가 유리한 대표 조건은 다음과 같다.

- Scope가 명확함
- 완료 조건을 정의할 수 있음
- 독립적으로 검증 가능함
- 다른 Task와 파일 충돌이 적음
- Git으로 상태를 전달할 수 있음
- 실행시간이 김
- CPU/RAM을 많이 사용함
- 사람의 중간 개입이 적음
- 여러 독립 작업으로 분리할 수 있음

이 조건은 5장의 Task Routing에서 더 구체적으로 사용한다.

---

## 4. Local과 Cloud를 하나만 선택할 필요는 없다

하나의 기능이 Local 또는 Cloud에 영구적으로 속할 필요는 없다.

예를 들어 출결 API 인증 변경을 진행한다고 하자.

처음에는 Local에서 요구사항을 확인한다.

```text
Local
→ 요구사항 분석
→ Architecture 확인
→ 영향 범위 확인
```

변경 방향이 정해지면 Cloud에 검증을 위임한다.

```text
Cloud
→ Unit Test
→ Integration Test
→ Docker Build
→ Web E2E
```

마지막으로 실제 내부 DB나 HSM 검증이 필요하면 다시 Local로 돌아온다.

```text
Local
→ Tibero 검증
→ HSM 확인
→ 최종 Review
→ Merge
```

전체 흐름은 다음과 같다.

```text
Local
  ↓
Cloud
  ↓
Local
```

이 구조를 이 책에서는 `Local → Cloud → Local Handoff`의 기본 형태로 사용한다.

핵심은 어느 도구를 계속 사용할지 고르는 것이 아니다.

> 작업 단계에 맞게 실행 위치를 바꾸는 것이다.

---

## 5. Internal Network는 강한 Routing 조건이다

Cloud Agent를 사용할 때 가장 먼저 확인해야 할 조건 중 하나는 네트워크다.

기업 환경에서는 다음 자원이 내부망에만 있을 수 있다.

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

Cloud 환경이 이 자원에 접근할 수 없다면 두 가지 선택이 있다.

첫 번째는 Task 전체를 Local에 남기는 것이다.

```text
Local Agent
→ Code
→ Internal DB
→ HSM
→ Verification
```

두 번째는 Cloud에서 가능한 부분만 분리하는 것이다.

```text
Cloud
→ Unit Test
→ PostgreSQL/Testcontainers Migration Validation
→ Docker Build

Local
→ Tibero 실제 적용 검증
→ HSM 최종 검증
```

후자의 방식이 Hybrid다.

중요한 것은 Cloud 환경을 억지로 내부망에 연결하는 것이 항상 정답은 아니라는 점이다.

Cloud에서 재현 가능한 검증과 내부 환경에서만 가능한 검증을 나누는 것만으로도 많은 작업을 분산할 수 있다.

---

## 6. Human Steering이 많으면 Local이 유리하다

Cloud Agent의 장점 중 하나는 Task를 위임하고 다른 일을 할 수 있다는 점이다.

그러나 작업 중간에 사람이 계속 판단해야 한다면 이 장점은 줄어든다.

예를 들어 UI를 새로 설계한다고 하자.

```text
Agent:
A안으로 구현했습니다.

Developer:
메뉴 구조가 너무 복잡하다. B안으로 바꿔보자.

Agent:
B안으로 바꿨습니다.

Developer:
데이터 흐름도 같이 바꿔야겠다.
```

이런 작업은 매우 짧은 주기로 상호작용이 반복된다.

Cloud에 위임할 수도 있지만 개발자가 계속 결과를 확인하고 다시 지시한다면 비동기 위임의 효과가 작다.

반대로 다음 Task는 중간 개입이 거의 필요 없다.

```text
Goal
모든 AuthService 테스트가 Java 21에서 통과하도록 deprecated API를 수정한다.

Validation
./gradlew :auth:test
```

이런 작업은 Cloud에 더 잘 맞는다.

따라서 Local/Cloud 판단에서 다음 질문을 사용한다.

> 사람이 작업 중간에 몇 번이나 개입해야 하는가?

---

## 7. Context가 크면 Local이 유리할 수 있다

Cloud Agent는 Repository를 읽을 수 있지만 Repository 전체를 매 Task마다 다시 이해시키는 것은 비용이 크다.

다음 Task를 비교해 보자.

### 작은 Context

```text
AuthService.java
JwtTokenProvider.java
AuthServiceTest.java
```

### 큰 Context

```text
Auth
Student
Permission
Payment
Database Schema
Common Exception
External API
Admin Web
```

작은 Context에서 수정할 수 있는 Task는 Cloud에 위임하기 쉽다.

반대로 전체 시스템의 관계를 같이 보고 방향을 정해야 한다면 개발자와 Local Agent가 먼저 구조를 좁히는 편이 낫다.

이후 Cloud에 넘길 때는 큰 문제를 작은 Task로 나눈다.

```text
큰 Architecture 문제
→ Local에서 결정
→ 작은 실행 Task로 분해
→ Cloud로 위임
```

따라서 Cloud Agent를 잘 쓰는 능력에는 Task를 작게 만드는 능력이 포함된다.

---

## 8. Cloud Agent의 실행시간과 개발자의 대기시간은 다르다

Cloud Agent가 40분 동안 작업했다고 가정하자.

이 숫자만 보면 느려 보일 수 있다.

그러나 실제 Workflow는 다음과 같을 수 있다.

```text
10:00
Cloud Agent에 테스트 작성 위임

10:01
Developer는 다음 Feature 작업 시작

10:40
Cloud Task 완료

11:20
Developer가 결과 Review
```

Cloud Task는 40분 걸렸다.

하지만 개발자가 40분 동안 기다린 것은 아니다.

따라서 다음 두 값을 구분해야 한다.

```text
Agent Execution Time
= Agent 또는 Worker가 Task를 수행한 시간

Developer Blocking Time
= 개발자가 해당 Task 때문에 실제로 멈춰 있던 시간
```

Cloud Agent의 가치는 Agent 자체를 몇 초 더 빠르게 만드는 것에만 있지 않다.

개발자의 Blocking Time을 줄이고 Local 컴퓨팅 자원을 비워 두는 것도 가치다.

이 주제는 4장에서 장시간 작업과 병렬성 관점으로 다시 다룬다.

---

## 9. Cloud가 항상 더 빠른 것은 아니다

Cloud는 Worker를 만들고 Repository를 checkout하고 환경을 준비해야 할 수 있다.

예를 들어 변경이 다음과 같다고 하자.

```text
README 문구 한 줄 수정
검증 5초
```

이 Task를 Cloud로 보내면 작업 자체보다 준비 과정이 더 길 수 있다.

```text
Worker Start
→ Checkout
→ Context Load
→ Branch
→ Commit
→ PR
```

반면 다음 작업은 Cloud의 준비 비용을 상쇄하기 쉽다.

```text
Integration Test 25분
E2E 18분
Docker Build 12분
```

따라서 중요한 질문은 다음이다.

> Cloud에서 실행할 수 있는가?

보다:

> Cloud로 보내는 것이 실제로 유리한가?

이 판단을 5장에서 Routing Framework로 만든다.

---

## 10. 같은 작업도 단계에 따라 실행 주체가 달라진다

Dependency Update를 예로 들어보자.

처음부터 Cloud Agent가 Repository를 분석할 필요는 없다.

```text
Dependency Update
      ↓
Cloud Runner
→ Build / Test
      ↓
PASS
→ 종료
```

실패한 경우에만 Agent가 필요할 수 있다.

```text
FAIL
→ 실패 정보 추출
→ Cloud Agent
→ Compatibility Fix
→ Runner 재검증
```

즉 선택지는 Local과 Cloud 두 개만 있는 것이 아니다.

Cloud 안에서도 다음을 나눈다.

```text
Cloud Runner
→ 결정론적 실행

Cloud Agent
→ 판단과 코드 수정
```

이 구분은 6장과 10장에서 상세히 다룬다.

---

## 11. Git은 Local과 Cloud를 연결한다

Cloud Agent가 Local Working Directory를 직접 공유하지 않는다면 작업 상태를 넘길 방법이 필요하다.

기본 Handoff는 다음과 같다.

```text
Local
→ Commit
→ Push
      ↓
Git Repository
      ↓
Cloud Worker
→ Branch
→ Work
→ Test
→ Commit
→ PR
```

이 구조의 장점은 Task의 기준점을 명시할 수 있다는 점이다.

```text
Task ID: task-142
Base SHA: abc123
Branch: agent/task-142
```

Cloud 결과도 Git과 연결한다.

```text
Result Commit: def456
Test: PASS
PR: #142
```

따라서 Git은 단순한 Source Version Control을 넘어 Local과 Remote Worker 사이의 Handoff Boundary 역할을 한다.

11장에서 Branch, Worktree, Container 격리와 함께 상세히 다룬다.

---

## 12. campus-platform에서 Local과 Cloud를 나누면

`campus-platform`에 다음 작업이 있다고 가정하자.

```text
A. 신규 인증 구조 설계
B. AuthService expired token 수정
C. 전체 Unit Test
D. Web E2E
E. Tibero 실제 Migration 검증
F. HSM 연동 오류 분석
G. Docker Build
```

기본 분류는 다음처럼 할 수 있다.

| Task | 기본 위치 | 이유 |
| --- | --- | --- |
| 신규 인증 구조 설계 | Local | 넓은 Context와 Human Steering 필요 |
| expired token 수정 | Cloud Agent 후보 | 작은 Scope, 독립 검증 가능 |
| 전체 Unit Test | Cloud Runner | 결정론적, Compute 중심 |
| Web E2E | Cloud Runner | Browser/장시간 검증 |
| Tibero 실제 Migration 검증 | Local/Hybrid | 내부 DB 의존 |
| HSM 오류 분석 | Local | 내부 장비/네트워크 의존 |
| Docker Build | Cloud Runner | 결정론적 Build |

이 표는 제품 기능표가 아니다.

Task의 특성을 실행 위치에 연결한 것이다.

프로젝트 환경이 달라지면 결과도 달라질 수 있다.

예를 들어 Cloud에서 내부 DB 접근이 가능하도록 별도 인프라를 제공한다면 Migration 검증 위치도 달라질 수 있다.

따라서 책 전체에서는 `무조건 Local`, `무조건 Cloud`보다 **판단 기준**을 우선한다.

---

## 13. Local + Cloud의 기본 Workflow

이 장의 내용을 하나의 흐름으로 정리하면 다음과 같다.

```text
Developer / Local Agent
        ↓
Requirement / Architecture
        ↓
Task Split
        ↓
이 Task를 어디서 실행할 것인가?
        ↓
+--------------+----------------+
|              |                |
Local       Cloud Runner    Cloud Agent
|              |                |
Interactive   Build/Test      Analyze/Fix
Internal      Validation      Independent Task
|              |                |
+--------------+----------------+
        ↓
Evidence / Git
        ↓
Local Review / Integration
```

Cloud Agent는 이 흐름의 일부다.

Local Agent를 제거하는 것이 목적이 아니다.

Runner를 Agent로 바꾸는 것도 목적이 아니다.

각 단계에서 가장 적합한 실행 위치와 실행 주체를 고르는 것이 목적이다.

---

## 14. 이 장에서 기억할 질문

Local과 Cloud 사이에서 고민할 때 다음 순서로 질문한다.

```text
현재 Local 상태가 필요한가?
내부망이 필요한가?
Human Steering이 많이 필요한가?
Context가 큰가?
독립적으로 검증 가능한가?
장시간 Compute 작업인가?
Git으로 입력과 결과를 넘길 수 있는가?
```

Cloud로 보내기 좋은 신호가 많아도 Hard Constraint 하나가 실행 위치를 바꿀 수 있다.

예를 들어 HSM 접근이 반드시 필요한 Task는 다른 조건이 좋아도 Local에 남을 수 있다.

반대로 Local에서 가능한 Task라도 장시간 Test나 E2E를 Cloud로 분리하면 개발자의 Blocking Time을 줄일 수 있다.

다음 장에서는 Cloud Session 안에서 사용되는 CPU, RAM, Disk와 LLM Token을 분리해 본다.

---

## 참고자료

제품별 기능은 변경될 수 있으므로 본문에서는 실행 위치에 따른 공통 차이만 사용했다. 확인 기준일은 2026-09-16이다.

- GitHub Docs, `About GitHub Copilot cloud agent`
- GitHub Docs, `About cloud and local sandboxes for GitHub Copilot`
- GitHub Docs, `About the GitHub Copilot app`
- OpenAI, `Codex is now generally available`, 2025-10-06

세부 조사 메모는 `research/chapter-02-local-cloud-official-sources.md`에서 관리한다.
