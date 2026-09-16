# 17장. Cloud가 항상 정답은 아니다

앞 장까지는 Cloud Agent를 어떻게 활용할 것인가를 다뤘다.

하지만 Cloud Agent를 잘 사용하는 팀은 모든 Task를 Cloud로 보내지 않는다.

Cloud에는 분명한 장점이 있다.

```text
독립 실행환경
장시간 작업의 비동기 위임
병렬 Compute
Local Resource 점유 감소
Git 기반 Handoff
```

그러나 이 장점이 항상 이득으로 이어지는 것은 아니다.

Task가 너무 작거나, Context가 지나치게 크거나, 내부망에 강하게 의존하거나, 사람이 계속 개입해야 한다면 Cloud의 준비 비용이 더 커질 수 있다.

그래서 이 책의 Routing 원칙은 처음부터 다음 문장을 포함한다.

> Cloud에 보낼 수 있는 Task와 Cloud에 보내는 것이 유리한 Task는 다르다.

그리고 하나를 더 추가한다.

> Cloud를 쓰지 않는 결정도 올바른 Routing 결과다.

이 장에서는 어떤 Task를 Local에 남겨야 하는지, Cloud에서 시작한 Task를 언제 Local로 되돌려야 하는지 정리한다.

---

## 1. Cloud 사용률은 목표가 아니다

Cloud Agent를 도입하면 다음 숫자를 보고 싶어질 수 있다.

```text
Cloud Task 수
Agent Session 수
자동 생성 PR 수
병렬 Worker 수
```

이 숫자가 증가하면 자동화가 잘되고 있는 것처럼 보일 수 있다.

그러나 책에서 보는 목표는 다르다.

```text
개발 Lead Time
Developer Blocking Time
검증 품질
Review Cost
Retry Cost
Compute / LLM 비용
```

예를 들어 Agent가 하루에 PR 30개를 만들었는데 Reviewer가 처리하지 못하고 Queue에 쌓인다면 Cloud 사용량은 늘었지만 개발 흐름은 빨라지지 않았다.

또 30초면 끝나는 수정에 Worker를 준비하고 Repository를 checkout하고 Branch를 만들고 PR을 생성하는 데 몇 분이 걸린다면 Cloud가 작업보다 더 비싼 절차가 된다.

따라서 목표를 다음처럼 잡지 않는다.

```text
Cloud Agent 사용률 최대화
```

대신 다음을 본다.

```text
이 Task를 Cloud로 보냈을 때
전체 Workflow 비용이 실제로 줄어드는가?
```

Cloud Agent는 수단이다.

---

## 2. 너무 작은 Task는 Local이 더 단순하다

다음 작업을 생각해 보자.

```text
README 문구 한 줄 수정
오타 수정
설정값 한 줄 변경
검증 10초
```

작업 자체는 매우 짧다.

그런데 Cloud로 보내면 다음 과정이 붙을 수 있다.

```text
Worker Start
→ Repository Checkout
→ Context Load
→ Branch
→ Change
→ Validation
→ Commit
→ Push
→ PR
```

작업보다 전달 비용이 커진다.

이 경우 Local에서 바로 수정하고 검증하는 편이 낫다.

```text
Local
→ 수정
→ 검증
→ Commit
```

여기서 `몇 분 이하의 작업은 Local` 같은 절대 기준을 만들지는 않는다.

프로젝트마다 Cloud Cold Start, Repository 크기, CI 속도, Review 방식이 다르기 때문이다.

대신 실제 값을 측정한다.

```text
Cloud Setup Time
Task Execution Time
Review Time
```

그리고 다음을 비교한다.

```text
Cloud Overhead > Task Work
```

이라면 Local을 우선한다.

9장에서 Prepared Environment와 Cache를 만들었더라도 모든 작은 Task가 Cloud에 적합해지는 것은 아니다.

---

## 3. 큰 Context가 필요한 Task는 Local이 유리할 수 있다

Cloud Agent는 Repository를 읽을 수 있다.

하지만 Repository를 읽을 수 있다는 것과 매 Task마다 Repository 전체를 다시 이해시키는 것이 좋은 것은 다르다.

다음 작업을 보자.

```text
인증
학생
권한
결제
공통 Exception
DB Schema
Admin Web
외부 API
```

이 모든 영역의 관계를 함께 보고 새 Architecture를 결정해야 한다고 하자.

이 Task는 몇 개의 관련 파일만 읽고 끝나는 실행 Task가 아니다.

필요한 것은 다음에 가깝다.

```text
전체 구조 탐색
선택지 비교
영향 분석
사람과 반복 대화
방향 결정
```

이런 작업은 Local에서 Developer와 Agent가 짧은 Feedback Loop로 진행하는 편이 자연스럽다.

```text
Developer + Local Agent
→ 구조 탐색
→ 질문
→ 선택지 비교
→ 결정
```

그다음 결정된 내용을 작은 Cloud Task로 나눈다.

```text
Architecture Decision
        ↓
Task A
Task B
Task C
        ↓
Cloud
```

7장의 Task Contract가 비정상적으로 커지는 것도 신호다.

예를 들어 다음처럼 된다면:

```text
Relevant Files: 80개
Related Modules: 12개
Validation Commands: 15개
```

Task Contract를 더 길게 쓰기 전에 Task 자체가 Cloud용으로 너무 큰지 확인해야 한다.

---

## 4. Human Steering이 많으면 비동기 위임의 가치가 줄어든다

Cloud의 중요한 장점은 Task를 넘기고 개발자가 다른 일을 할 수 있다는 점이다.

그러나 다음처럼 작업 중간마다 사람이 판단해야 한다면 어떨까.

```text
Agent
→ UI A안 구현

Developer
→ 메뉴가 복잡하다. B안으로 변경

Agent
→ B안 구현

Developer
→ 데이터 배치를 다시 바꾸자

Agent
→ 수정

Developer
→ 모바일 화면도 같이 조정
```

이 작업은 위임이라기보다 계속 이어지는 협업 세션에 가깝다.

개발자가 Cloud 결과를 기다리고 다시 지시하고 다시 기다린다면 비동기성의 이점이 줄어든다.

판단 질문은 단순하다.

> 작업 중간에 개발자가 몇 번이나 개입해야 하는가?

개입 빈도가 높다면 Local을 우선한다.

반대로 다음 Task는 Cloud에 적합하다.

```text
attendance 모듈 deprecated API 6건을 교체하고
./gradlew :attendance:test를 통과한다.
```

Scope와 Validation이 명확하고 중간 질문이 적다.

Cloud의 비동기성은 사람이 없어도 상당 구간을 독립적으로 진행할 수 있는 Task에서 가치가 크다.

---

## 5. Internal Network는 강한 Hard Constraint다

기업 프로젝트에서는 다음 자원이 내부망에만 있을 수 있다.

```text
Tibero / Oracle
HSM
Internal Jenkins
Internal API
VPN-only Server
Internal Git
Nexus
사내 Redis / Kafka
```

Cloud Worker가 이 자원에 접근할 수 없다면 LLM 성능과 상관없이 Task를 끝낼 수 없다.

예를 들어 HSM 연동 오류를 분석한다고 하자.

```text
Cloud Agent
→ Source 분석 가능
→ Unit Test 가능
→ 실제 HSM 접근 불가
```

최종 원인을 실제 장비에서 확인해야 한다면 Local이 필요하다.

이 경우 두 가지 선택이 있다.

첫 번째:

```text
Task 전체 Local
```

두 번째:

```text
Cloud
→ Pure Logic Test
→ Mock/Contract Test
→ Build

Local
→ 실제 HSM Integration
```

두 번째가 Hybrid다.

Cloud에 접근할 수 없는 내부 자원을 억지로 복제하는 것이 항상 정답은 아니다.

13~16장에서 본 것처럼 Cloud에서 일반 검증을 최대한 끝내고 마지막 경계만 Local에 남길 수 있다.

---

## 6. Repository나 데이터를 Cloud에 제공할 수 없는 경우

기술적으로 Cloud Agent가 가능해도 조직 정책 때문에 사용할 수 없는 경우가 있다.

예:

```text
외부 SaaS에 Source 제공 금지
특정 Repository 외부 반출 금지
운영 데이터 반출 금지
Secret 제공 제한
계약상 제3자 처리 금지
```

이 조건은 Routing의 Hard Constraint다.

다른 조건을 점수로 계산할 필요도 없다.

```text
Policy prohibits cloud execution
→ Local
```

책에서는 법률이나 보안 정책을 해석하지 않는다.

중요한 것은 Cloud 여부를 결정할 때 모델 기능만 보는 것이 아니라 `이 Repository와 이 데이터가 Cloud 실행환경으로 이동할 수 있는가`를 확인하는 것이다.

Task 일부만 분리할 수 있다면 다음처럼 구성할 수 있다.

```text
민감한 부분
→ Local

공개 가능한 Test Harness / Synthetic Data 기반 검증
→ Cloud
```

하지만 허용 여부는 조직 정책이 우선한다.

---

## 7. Cloud에서 재현할 수 없는 문제를 계속 Retry하지 않는다

다음 문제는 Cloud에서 재현하기 어려울 수 있다.

```text
특정 Mac에서만 발생
특정 USB/Device 의존
HSM Firmware 의존
내부 Network Latency
Production-only Race Condition
운영 데이터 상태 의존
```

좋지 않은 흐름:

```text
Cloud에서 재현 안 됨
→ Context 확대
→ 다시 Agent
→ 다시 실행
→ 재현 안 됨
→ 다시 Retry
```

Compute와 Token만 계속 소비된다.

먼저 재현 가능성을 확인한다.

```text
같은 입력을 만들 수 있는가?
같은 Runtime 조건을 만들 수 있는가?
같은 외부 의존성에 접근할 수 있는가?
```

아니라면 Local Fallback한다.

```text
Cloud
→ cannot reproduce
→ Evidence 저장
→ Local Fallback
```

Cloud에서 얻은 정보는 버리지 않는다.

다음과 같이 넘긴다.

```text
실행한 명령
현재 Commit
실패/재현 결과
수집한 Artifact
시도한 수정
```

Local에서는 이 상태에서 이어간다.

---

## 8. 미커밋 Local State는 Cloud Handoff를 어렵게 만든다

개발자의 현재 Workspace에는 Git에 없는 상태가 있을 수 있다.

```text
uncommitted source
untracked file
IDE-only config
local DB state
local environment variable
임시 patch
```

Cloud Worker는 이 상태를 자동으로 공유하지 않는다.

예를 들어 Developer가 다음처럼 요청한다고 하자.

```text
내 Mac에서 지금 수정한 상태 이어서 고쳐줘.
```

Remote Worker에게는 기준점이 없다.

선택은 두 가지다.

```text
Commit / Push 가능한 상태로 정리
→ Cloud Handoff
```

또는:

```text
현재 작업은 Local에서 계속
```

Cloud 사용을 위해 억지로 중간 상태를 Commit하는 것이 더 번거롭다면 Local에 남겨도 된다.

Git은 Handoff Boundary이지만 모든 작업이 즉시 Handoff 가능한 것은 아니다.

---

## 9. 환경 준비 비용이 Task보다 크면 Cloud가 불리하다

일부 Task는 특수한 환경을 요구한다.

예:

```text
대형 Simulator
특수 SDK
수십 GB Dataset
대형 Docker Image
희귀한 Native Dependency
```

한 번의 짧은 검증을 위해 이 환경을 매번 준비해야 한다면 Cloud Cold Start가 커진다.

다음처럼 비교한다.

```text
Environment Setup: 12분
Task Execution: 2분
```

반복적으로 사용하는 환경이라면 9장의 Prepared Environment로 비용을 줄일 수 있다.

```text
한 번 준비
→ 여러 Task에서 재사용
```

하지만 한 번만 사용하는 특수 환경이라면 Local에 이미 준비된 환경을 쓰는 편이 낫다.

즉 환경을 Cloud에 만들 수 있는가보다 **그 환경을 만드는 비용을 반복해서 회수할 수 있는가**를 본다.

---

## 10. 같은 파일을 여러 Agent가 수정하는 병렬화는 피한다

Cloud Worker를 격리하면 동시에 여러 Branch에서 수정할 수 있다.

하지만 격리는 논리적 충돌까지 없애지 않는다.

다음 구조를 보자.

```text
Agent A → UserService.java
Agent B → UserService.java
Agent C → UserService.java
```

각 Agent는 독립 Container와 Branch에서 성공할 수 있다.

문제는 Fan-in이다.

```text
PR A
PR B
PR C
  ↓
Merge Conflict
Semantic Conflict
Regression
```

DB Migration도 마찬가지다.

```text
Agent A → migration sequence 변경
Agent B → 같은 schema 변경
```

Cloud에서 병렬 실행됐다는 사실은 Integration이 쉬워졌다는 의미가 아니다.

이 경우 다음 중 하나를 선택한다.

```text
순차화
Task 재분해
공통 변경 먼저 처리
```

11~12장의 원칙을 다시 적용한다.

> 병렬화의 대상은 Agent가 아니라 독립 Task다.

---

## 11. 요구사항이 정해지지 않았다면 실행보다 결정이 먼저다

다음 요청을 보자.

```text
신규 학생 인증 UX를 가장 좋은 방식으로 만들어줘.
```

아직 다음이 정해지지 않았다.

```text
어떤 인증 방식을 사용할지
모바일/웹 흐름을 어떻게 나눌지
실패 UX를 어떻게 할지
기존 시스템을 얼마나 유지할지
```

이 상태에서 Cloud Agent를 실행하면 Agent가 구현과 동시에 요구사항을 추론하게 된다.

결과가 나오더라도 Developer가 방향을 바꾸면 작업 대부분을 다시 해야 할 수 있다.

권장 흐름:

```text
Local
→ Requirement Clarification
→ Option 비교
→ Decision
→ Task Split
      ↓
Cloud
→ 독립 구현 / 검증
```

Cloud는 결정된 Task를 실행하는 데 강하다.

결정 자체가 계속 바뀌는 상황에서는 Local Feedback Loop가 더 중요하다.

---

## 12. Architecture 변경은 기본적으로 Local 중심이다

다음 작업을 생각해 보자.

```text
모듈 Boundary 재설계
인증 구조 교체
DB Architecture 변경
Cross-system Transaction 재설계
공통 API Error Model 변경
```

이 작업들은 여러 모듈에 동시에 영향을 준다.

결정 과정에는 다음이 필요하다.

```text
현재 구조 이해
제약 확인
선택지 비교
Trade-off 논의
조직 규칙 확인
Migration 계획
```

Cloud Agent가 조사나 후보 생성을 도울 수는 있다.

그러나 이를 독립 비동기 Task처럼 통째로 넘기는 것은 적합하지 않을 수 있다.

권장 구조:

```text
Local
→ Architecture Decision
→ Migration Plan
→ 작은 실행 Task 정의
      ↓
Cloud
→ 반복 변경
→ Build/Test
→ Validation
```

설계와 실행을 분리한다.

---

## 13. 불확실한 Production Incident는 먼저 범위를 좁힌다

운영에서 다음 이슈가 발생했다고 하자.

```text
가끔 응답이 느리다.
```

이 상태로 Cloud Agent에게 다음처럼 맡기는 것은 위험하다.

```text
성능 문제를 찾아서 고쳐줘.
```

재현 조건도 없고 병목 위치도 모른다.

먼저 운영/Local 환경에서 조사한다.

```text
Metric
Log
Trace
DB 상태
Network 상태
재현 조건
```

그 결과 다음처럼 구체화될 수 있다.

```text
AttendanceService.findStudent
특정 Query에서 5초 이상
재현 Test 존재
```

이제 Cloud Agent Task로 만들 수 있다.

```text
Goal
Query latency regression 원인 수정

Validation
performance regression test
```

즉 Production Incident 전체가 Cloud Task가 아니라, 조사 후 만들어진 **재현 가능한 작은 문제**가 Cloud Task가 된다.

---

## 14. 비용이 작업 가치보다 클 수 있다

Cloud Agent의 비용은 Token만이 아니다.

```text
Compute
LLM
Environment Start
Artifact Storage
Review
Merge
Human Attention
```

작은 문서 오탈자를 수정하는데 Agent 다섯 개를 Best-of-N으로 실행한다고 하자.

기술적으로는 가능할 수 있다.

하지만 검증 가치보다 비용이 크다.

12장에서 Best-of-N의 기본값을 N=1로 둔 이유다.

Task 위험도와 가치에 맞게 실행 수준을 선택한다.

예:

```text
오탈자
→ Local

재현 가능한 작은 Bug
→ Cloud Agent 1개

실패 비용이 크고 자동 검증 가능한 어려운 Bug
→ 제한적 Best-of-N 검토
```

많이 실행할 수 있다는 사실이 많이 실행해야 한다는 의미는 아니다.

---

## 15. Cloud Task는 중간에 중단할 수 있어야 한다

Cloud Task가 시작됐다고 끝까지 Cloud에서 해결할 필요는 없다.

다음 신호가 나오면 중단 또는 재분류를 검토한다.

```text
같은 Failure Fingerprint 반복
Retry Budget 소진
Scope 예상보다 크게 증가
Internal Dependency 발견
필요 Repository가 계속 증가
요구사항 불명확 발견
Human Steering 반복
Cloud에서 재현 불가
```

예를 들어 처음에는 세 파일 수정으로 예상했다.

```text
AuthService.java
JwtTokenProvider.java
AuthServiceTest.java
```

그런데 분석 중 다음이 드러났다.

```text
auth
student
attendance
common
admin
DB schema
```

이 시점에서 Agent에게 Repository 전체를 더 읽히는 것이 항상 좋은 대응은 아니다.

```text
Cloud Task Stop
→ Evidence 반환
→ Local에서 재분해
```

Task 중단은 실패가 아니다.

잘못된 Routing을 빨리 수정한 것이다.

---

## 16. Local Fallback에도 Handoff 정보가 필요하다

Cloud에서 Local로 되돌릴 때 다음 한 줄만 남기면 안 된다.

```text
Cloud에서 해결 못했습니다.
```

Cloud에서 이미 수행한 작업을 Local에서 재사용할 수 있어야 한다.

권장 Return Package:

```text
Current Commit
Changed Files
Failed Command
Failure Summary
Artifacts
Failure Fingerprint
Attempted Fixes
Budget / Retry Count
```

예:

```text
Task: HSM-37
Current SHA: def456
Status: LOCAL_FALLBACK
Reason: actual HSM required

Cloud Verification:
Unit 88/88 PASS
Mock Contract PASS

Remaining:
actual HSM session validation
```

Local Developer는 처음부터 다시 분석하지 않고 남은 경계부터 확인할 수 있다.

Cloud 실패도 Evidence를 남겨야 하는 이유다.

---

## 17. Cloud에 불리한 신호를 빠르게 찾는다

Routing을 반대로 보면 Cloud에 불리한 대표 신호를 다음처럼 정리할 수 있다.

```text
Internal Network Required
Large Cross-module Context
Frequent Human Steering
Non-reproducible Failure
Tiny Task
High Shared-file Conflict
Policy Restriction
Large One-off Environment Setup
Local Uncommitted State Dependency
```

이 중 일부는 단순 감점 요소이고 일부는 Hard Constraint다.

예:

```text
작은 Task
→ 상황에 따라 Cloud 가능

HSM 실제 장비 필수 + Cloud 접근 불가
→ Local Hard Constraint
```

따라서 점수만으로 자동 결정하지 않는다.

Hard Constraint가 있으면 다른 장점이 많아도 Local 또는 Hybrid를 선택한다.

---

## 18. campus-platform에서 다시 분류해 보자

### Local에 남길 작업

```text
HSM 실제 장비 장애 분석
Tibero에서만 재현되는 SQL 동작
새 인증 Architecture 설계
현재 Local Working Tree에만 있는 긴급 수정
운영 설정 한 줄 즉시 변경
```

### Cloud에 보내기 좋은 작업

```text
AuthService 재현 가능한 Bug
전체 Unit Test
Integration Test
E2E
Docker Build
독립 Module Refactoring
```

### Hybrid

```text
Migration 작성/일반 검증
→ Cloud

Tibero 실제 적용
→ Local
```

또는:

```text
HSM 연동 Pure Logic
→ Cloud Unit Test

실제 HSM Session
→ Local
```

중요한 것은 기능 이름 자체가 아니라 **단계별 실행 조건**이다.

---

## 19. Cloud 보내기 전 마지막 질문

실제 Task를 넘기기 전에 다음을 확인한다.

```text
Scope가 명확한가?
독립적으로 검증 가능한가?
Git으로 입력을 전달 가능한가?
Internal Network가 필요한가?
Context가 과도하게 큰가?
Human Steering이 잦은가?
Cloud에서 재현 가능한가?
Shared File / Schema 충돌이 큰가?
Task보다 Cloud Overhead가 큰가?
정책상 Cloud 실행이 허용되는가?
```

이 질문에 대한 답이 매번 같을 필요는 없다.

같은 기능도 단계에 따라 달라진다.

```text
Architecture
→ Local

Implementation
→ Cloud Agent 후보

Build/Test
→ Cloud Runner

Internal DB
→ Local
```

이것이 Task Routing의 최종 형태다.

---

## 20. Cloud를 쓰지 않는 능력도 Cloud 활용 능력이다

Cloud Agent를 잘 사용한다는 말을 `많이 사용한다`로 이해하면 쉽게 과도한 자동화로 간다.

이 책에서 말하는 활용 능력은 다음에 가깝다.

```text
Cloud가 이득인 Task를 찾는다.
Cloud가 필요 없는 Task는 Runner나 Local에 남긴다.
Cloud에서 해결하기 어려워지면 빨리 Fallback한다.
```

Cloud를 사용하는 결정과 사용하지 않는 결정은 모두 Routing이다.

> Cloud Agent를 잘 사용하는 능력에는 Cloud를 쓰지 않을 때를 아는 것도 포함된다.

---

## 장을 마치며

Cloud Agent는 Local Agent의 상위 호환이 아니다.

실행 위치가 다르고 강점이 다르다.

Cloud는 독립 실행환경, 장시간 작업, 병렬 Compute, 비동기 Handoff에서 큰 가치를 만든다.

반대로 다음 상황에서는 Local이 더 나을 수 있다.

```text
작업이 너무 작음
Context가 너무 큼
사람 개입이 잦음
내부망이 필수
재현 불가
정책상 Cloud 불가
동일 파일 충돌이 큼
```

그리고 Cloud에서 시작한 Task도 언제든 Local로 되돌릴 수 있다.

```text
Cloud
→ Evidence
→ Local Fallback
```

Fallback은 실패가 아니다.

처음의 Routing 가정을 실행 중에 다시 평가한 결과다.

다음 장에서는 이 책에서 만든 Cloud Agent Workflow가 안정화된 뒤 무엇을 더 자동화할 수 있는지 살펴본다.

하지만 방향은 바뀌지 않는다.

먼저 Cloud Agent를 잘 사용하는 Workflow를 만들고, 그 다음에 자동화를 확장한다.
