# 17장. Cloud가 항상 정답은 아니다

5장에서는 작업을 시작할 때 `Local / Cloud / Hybrid / Runner-first` 중 어디에 둘지 판단했다.

17장에서는 반대 방향을 본다.

> 클라우드에서 시작한 작업을 언제 중단하고, 언제 로컬이나 Hybrid로 되돌려야 하는가?

클라우드 에이전트의 장점은 분명하다.

```text
독립 실행환경
장시간 작업의 비동기 위임
Parallel Compute
Local Resource Occupancy 감소
Git 기반 Handoff
```

그러나 이 장점이 항상 이득으로 이어지는 것은 아니다.

> 클라우드에 보낼 수 있는 작업과 클라우드에 보내는 것이 유리한 작업은 다르다.

> 클라우드를 쓰지 않는 결정도 올바른 실행 위치 결정 결과다.

이 장의 핵심은 클라우드 사용률을 높이는 것이 아니라 **클라우드의 이점이 사라지는 신호를 빨리 찾는 것**이다.

---

## 1. Cloud 사용량은 성공 지표가 아니다

다음 숫자가 늘었다고 해서 개발 작업 흐름이 좋아졌다고 볼 수는 없다.

```text
Cloud Task 수
Agent Session 수
자동 생성 PR 수
병렬 Worker 수
```

대신 다음을 본다.

```text
Lead Time
Developer Blocking Time
검증 품질
Review Cost
Retry Cost
Compute / LLM / Human Cost
```

에이전트가 많은 PR을 만들었지만 검토 대기열이 쌓였다면 병목은 그대로다.

작은 수정 하나를 위해 작업자 준비, Checkout, 브랜치, PR 과정을 거친다면 클라우드 준비와 회수 비용이 작업 자체보다 클 수 있다.

목표는 다음이 아니다.

```text
Cloud Agent 사용률 최대화
```

목표는 다음에 가깝다.

```text
이 Task를 Cloud로 보냈을 때
전체 Workflow 비용이 실제로 줄어드는가?
```

---

## 2. Task보다 전달 비용이 크면 Local이 낫다

다음 작업을 생각해 보자.

```text
README 오타 수정
설정값 한 줄 변경
짧은 코드 수정
```

클라우드에는 고정 비용이 있다.

```text
Worker Start
Repository Checkout
Environment Ready
Context Load
Commit / Push / PR
Review
```

개념적으로 다음 관계라면 로컬을 우선한다.

```text
Cloud Overhead > Task Work
```

`몇 분 이하이면 Local` 같은 절대 기준은 두지 않는다. 프로젝트마다 Cold Start, CI, 검토 시간이 다르기 때문이다.

9장의 미리 준비한 실행환경으로 시작 비용을 줄일 수 있어도 모든 작은 작업이 클라우드에 적합해지는 것은 아니다.

---

## 3. 큰 Context와 잦은 Human Steering은 Cloud 이점을 줄인다

다음처럼 여러 영역을 동시에 이해해야 하는 작업이 있다고 하자.

```text
Auth
Student
Permission
DB Schema
Common Exception
Admin Web
External API
```

이 작업이 시스템 구조 결정이나 요구사항 정리까지 포함한다면 독립적인 실행 작업이라기보다 탐색과 의사결정에 가깝다.

```text
Developer + Local Agent
→ 구조 탐색
→ 선택지 비교
→ 결정
→ 작은 Task로 분해
```

클라우드 작업 명세가 계속 커지는 것도 신호다.

```text
Relevant Files 증가
관련 Module 증가
Validation 증가
Forbidden Changes 증가
```

또 작업 중 사람이 계속 방향을 바꿔야 한다면 비동기 위임의 가치가 줄어든다.

```text
Cloud 결과 확인
→ 방향 수정
→ 다시 실행
→ 다시 확인
```

이 흐름이 반복된다면 로컬의 짧은 피드백 주기가 더 적합할 수 있다.

---

## 4. Internal Network와 정책은 Hard Constraint다

일부 조건은 단순한 비용 비교가 아니라 실행 위치를 제한한다.

예:

```text
Tibero / Oracle
HSM
Internal Jenkins
VPN-only API
Internal Git / Nexus
사내 Redis / Kafka
```

클라우드에서 이 자원에 접근할 수 없고 작업 완료에 반드시 필요하다면 로컬 또는 Hybrid가 된다.

```text
Cloud
→ Pure Logic / Mock / 일반 검증
      ↓
Local
→ 실제 내부 자원 검증
```

조직 정책도 같은 종류의 제약이다.

```text
외부 SaaS에 Source 제공 금지
특정 Repository 반출 금지
운영 데이터 반출 금지
Secret 제공 제한
계약상 제3자 처리 제한
```

이 경우 제품 기능과 관계없이 클라우드 실행이 불가능할 수 있다.

책에서는 법률이나 보안 정책을 해석하지 않는다. 실행 위치 결정의 Hard Constraint로만 다룬다.

---

## 5. Cloud에서 재현되지 않는 문제를 계속 Retry하지 않는다

다음 문제는 클라우드에서 재현하기 어려울 수 있다.

```text
특정 Device 의존
HSM Firmware 의존
Internal Network Latency
Production-only Race Condition
운영 데이터 상태 의존
Local-only File / Process State
```

좋지 않은 흐름:

```text
재현 안 됨
→ Context 확대
→ Agent Retry
→ 재현 안 됨
→ 다시 Retry
```

먼저 다음을 확인한다.

```text
같은 입력을 만들 수 있는가?
같은 Runtime 조건을 만들 수 있는가?
같은 외부 의존성에 접근할 수 있는가?
```

아니라면 클라우드에서 얻은 검증 근거를 보존하고 로컬로 이동한다.

```text
Cloud
→ cannot reproduce
→ Evidence 저장
→ Local Fallback
```

운영 중 발생한 장애도 마찬가지다. `가끔 느리다`처럼 원인이 불분명한 문제는 먼저 운영 환경이나 로컬 환경에서 범위를 좁힌다. 그런 다음 재현할 수 있는 작은 오류로 정리됐을 때 클라우드 작업으로 바꾸는 편이 낫다.

---

## 6. Local Working State가 기준점이라면 Handoff가 먼저다

클라우드 작업자는 다음 상태를 자동으로 공유하지 않는다.

```text
uncommitted source
untracked config
IDE-only setting
local DB state
임시 patch
running process state
```

선택지는 두 가지다.

```text
명시적인 Git 상태로 정리
→ Cloud Handoff
```

또는:

```text
현재 Task는 Local에서 계속
```

클라우드 사용을 위해 중간 실험 상태를 억지로 정리하는 비용이 더 크다면 로컬을 선택할 수 있다.

Git은 작업을 넘기는 경계이지만 모든 작업이 즉시 작업 전달 가능한 것은 아니다.

---

## 7. 격리할 수 있어도 통합 비용이 크면 병렬화하지 않는다

여러 클라우드 작업자가 다른 브랜치와 컨테이너를 사용해도 논리적 충돌은 남는다.

```text
Agent A → UserService.java
Agent B → UserService.java
Agent C → UserService.java
```

또는:

```text
Agent A / B
→ 같은 DB Schema / Migration Sequence 수정
```

실행 중에는 독립적이어도 결과 통합에서 비용이 발생한다.

```text
Merge Conflict
Semantic Conflict
Regression
Review 증가
```

이 경우 선택은 다음과 같다.

```text
순차화
Task 재분해
공통 변경 선행
```

11~12장의 원칙을 역으로 적용한다.

> 병렬화의 대상은 에이전트가 아니라 독립 작업이다.

---

## 8. Cloud Task 중단 기준을 명시한다

클라우드 작업은 시작했다고 끝까지 클라우드에서 해결할 필요가 없다.

다음 신호가 나타나면 중단이나 재분류를 검토한다.

```text
동일 Failure Fingerprint 반복
Retry / Cost Budget 소진
Scope 예상보다 크게 증가
Internal Dependency 발견
필요 Repository 계속 증가
요구사항 불명확 발견
Human Steering 반복
Cloud 재현 불가
```

예를 들어 처음에는 제한된 파일 변경으로 예상했지만 실제로 여러 모듈과 DB 구조까지 영향을 준다면 다음처럼 전환한다.

```text
Cloud Task Stop
→ 현재 Evidence 반환
→ Local Architecture Review
→ Task 재분해
```

이때 작업을 중단했다고 해서 실패한 것은 아니다. 처음 실행 위치를 정할 때의 가정이 잘못됐음을 확인하고 빠르게 바로잡은 것이다.

---

## 9. Local Fallback도 Handoff다

로컬로 돌아갈 때 다음 한 줄만 남기면 안 된다.

```text
Cloud에서 해결하지 못함
```

로컬에서 그대로 이어갈 수 있는 반환 묶음을 만든다.

```text
Task ID
Base SHA
Result SHA
Changed Files
Failed Command
Failure Summary
Artifacts
Failure Fingerprint
Attempted Fixes
Retry / Budget 상태
Fallback Reason
```

예:

```text
Task: HSM-37
Status: LOCAL_FALLBACK
Base SHA: abc123
Result SHA: def456
Reason: actual HSM required

Cloud Validation Result
Unit: PASS
Mock Contract: PASS

Artifact Path
artifacts/HSM-37/def456/

Remaining
Actual HSM session validation
```

클라우드에서 한 작업을 버리지 않고 남은 경계부터 로컬에서 이어간다.

> 로컬 복귀는 실패가 아니라 실행 위치 결정의 일부다.

---

## 10. campus-platform에서 역방향 Routing을 적용하면

기능 이름보다 실행 조건을 본다.

| 상황 | 기본 판단 |
| --- | --- |
| HSM 실제 장비 장애 분석 | 로컬 |
| Tibero에서만 재현되는 SQL 동작 | 로컬 |
| 새 인증 시스템 구조 설계 | 로컬 |
| 재현 가능한 AuthService 오류 | 클라우드 에이전트 후보 |
| Unit / Integration / E2E / Docker | 클라우드 실행기 |
| 마이그레이션 일반 검증 | 클라우드 실행기 |
| Tibero 실제 적용 | 로컬 |
| HSM Pure Logic / Mock 테스트 | 클라우드 실행기 |
| 실제 HSM Session | 로컬 |

같은 기능 안에서도 단계별 실행 위치는 달라질 수 있다.

```text
Architecture
→ Local

Implementation
→ Cloud Agent 후보

Build / Test
→ Cloud Runner

Internal Boundary
→ Local
```

---

## 11. Cloud 보내기 전과 실행 중에 묻는 질문은 다르다

5장에서는 작업 시작 전에 실행 위치를 판단했다.

```text
Scope가 명확한가?
독립 검증 가능한가?
Internal Network가 필요한가?
Git Handoff가 가능한가?
```

17장에서는 실행 중 조건이 바뀌었는지 본다.

```text
예상보다 Scope가 커졌는가?
Cloud에서 재현 가능한가?
같은 실패가 반복되는가?
새 Hard Constraint가 발견됐는가?
사람 개입이 계속 필요한가?
Fan-in 비용이 예상보다 큰가?
```

초기 실행 위치 결정과 실행 중 실행 위치 재결정을 구분하면 클라우드 작업을 억지로 끝까지 유지하지 않아도 된다.

---

## 12. Cloud를 쓰지 않는 능력도 Cloud 활용 능력이다

클라우드 에이전트를 잘 사용한다는 말을 `많이 사용한다`로 이해하면 과도한 자동화로 이어질 수 있다.

이 책에서 말하는 활용 능력은 다음에 가깝다.

```text
Cloud가 이득인 Task를 찾는다.
Runner로 끝날 작업은 Runner에 둔다.
Local이 유리한 작업은 Local에 둔다.
Cloud 이점이 사라지면 빠르게 Fallback한다.
```

> 클라우드 에이전트를 잘 사용하는 능력에는 클라우드를 쓰지 않을 때를 아는 것도 포함된다.

다음 장에서는 이 작업 흐름이 안정화된 뒤 반복 결정을 어디까지 Harness와 Orchestration으로 자동화할 수 있는지 살펴본다.