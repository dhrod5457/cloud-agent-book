# 2장. Local Agent와 Cloud Agent

Cloud Agent를 도입할 때 먼저 결정해야 하는 것은 어떤 제품을 쓸지가 아니다.

더 중요한 질문은 다음이다.

> 이 Task를 어디에서 실행할 것인가?

같은 계열의 Coding Agent라도 개발자 PC에서 실행되는 경우와 별도 Cloud 환경에서 실행되는 경우에는 사용할 수 있는 상태와 자원이 다르다.

Local은 현재 Workspace, 미커밋 파일, VPN, 내부 DB 같은 자원에 가깝다. Cloud는 개발자 PC와 분리된 환경에서 장시간 작업을 실행하거나 여러 독립 Task를 동시에 처리하기 쉽다.

따라서 Local Agent와 Cloud Agent를 경쟁 관계로 볼 필요는 없다.

> Cloud Agent는 Local Agent를 대체하는 것이 아니다.

이 장에서는 두 실행 위치의 차이와 Hybrid 사용의 기본 형태만 정리한다. 실제 Routing Framework는 5장에서 만든다.

---

## 1. 차이는 모델보다 실행 위치에서 시작한다

Local Agent는 개발자의 현재 환경을 직접 사용할 수 있다.

```text
Developer PC
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

반면 Cloud Agent는 별도 실행환경에서 Repository와 Task를 기준으로 시작한다.

```text
Cloud Environment
├─ Repository Checkout
├─ Workspace
├─ CPU / RAM / Disk
├─ Development Tools
└─ Agent
```

다음 두 작업을 비교해 보자.

### 현재 Local 상태가 필요한 작업

```text
미커밋 코드와 Local DB 상태를 같이 보면서
간헐적으로 발생하는 오류를 조사한다.
```

### 독립적으로 전달 가능한 작업

```text
Base SHA: abc123
Task: AuthService expired token 수정
Validation: ./gradlew test --tests AuthServiceTest.expiredToken
```

두 작업의 차이는 모델 성능보다 **현재 상태를 어디까지 전달할 수 있는가**에 가깝다.

---

## 2. Local이 유리한 작업

Local의 가장 큰 장점은 개발자와 현재 작업 상태에 가깝다는 점이다.

다음과 같은 상황을 생각해 보자.

```text
Developer:
인증 모듈을 OAuth 기준으로 바꿀지 기존 JWT 구조를 유지할지 비교해보자.

Agent:
현재 auth와 연동 구조를 보면 두 선택지가 있습니다.

Developer:
DB Schema는 변경하지 않는 방향으로 다시 보자.
```

이 작업은 중간 판단이 계속 바뀐다.

이런 경우에는 비동기 위임보다 짧은 Feedback Loop가 중요하다.

Local에 적합한 대표 조건은 다음과 같다.

- 요구사항이 아직 불명확함
- Architecture 판단 비중이 큼
- 여러 모듈을 넓게 탐색해야 함
- 개발자와 질문/수정이 반복됨
- 현재 미커밋 상태를 사용해야 함
- 로컬 파일이나 디바이스에 의존함
- VPN이나 내부 시스템 접근이 필요함
- 최종 Integration과 Review가 필요함

Local은 단순히 `작은 작업을 하는 곳`이 아니다. 사람의 개입과 현재 환경 의존성이 큰 작업에 가깝다.

---

## 3. Cloud가 유리한 작업

Cloud Agent는 Task가 명확하고 독립적으로 검증할 수 있을수록 사용하기 쉽다.

```text
Task
AuthService expired token 처리 수정

Failure
expected: 401
actual: 200

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

이 Task는 다음 특성을 가진다.

```text
Scope가 작음
완료 조건이 있음
독립 Test 가능
Git 기준점을 만들 수 있음
중간 질문이 적음
```

Cloud가 유리한 대표 조건은 다음과 같다.

- Scope와 완료 조건이 명확함
- 독립적으로 검증 가능함
- 다른 Task와 파일 충돌이 적음
- Git으로 상태를 전달할 수 있음
- 실행시간이 길거나 Local 자원을 많이 사용함
- 사람의 중간 개입이 적음
- 여러 독립 작업으로 분리할 수 있음

Cloud의 장점은 단순히 원격에 있다는 데 있지 않다.

독립 실행환경을 이용해 Local 자원을 비우고, 장시간 작업을 위임하며, 서로 독립적인 작업을 동시에 실행할 수 있다는 데 있다.

---

## 4. 하나의 기능도 Local과 Cloud 사이를 이동한다

Task는 Local 또는 Cloud 중 하나에 영구적으로 속하지 않는다.

출결 API 인증 변경을 예로 들어보자.

처음에는 Local에서 요구사항과 영향 범위를 확인한다.

```text
Local
→ 요구사항 분석
→ Architecture 확인
→ 영향 범위 확인
```

변경 기준이 정해지면 독립 검증을 Cloud로 보낼 수 있다.

```text
Cloud
→ Unit Test
→ Integration Test
→ Docker Build
→ Web E2E
```

마지막에 실제 내부 DB나 장비 검증이 필요하면 Local로 돌아온다.

```text
Local
→ Tibero 검증
→ HSM 확인
→ 최종 Review
→ Merge
```

즉 기본 형태는 다음과 같다.

```text
Local
  ↓
Cloud
  ↓
Local
```

13장에서는 이 흐름을 Git과 Evidence를 이용한 Handoff로 구체화한다.

---

## 5. Internal Network는 강한 제약이 된다

기업 환경에서는 다음 자원이 내부망에만 있을 수 있다.

```text
Internal Git
Nexus
Jenkins
Tibero / Oracle
HSM
Internal API
VPN-only Server
```

Cloud 환경이 이 자원에 접근할 수 없다면 Task 전체를 Local에 남기거나, Cloud에서 가능한 부분만 분리해야 한다.

```text
Cloud
→ Unit Test
→ Testcontainers 기반 Integration
→ Docker Build

Local
→ Tibero 실제 검증
→ HSM 최종 검증
```

이런 형태가 Hybrid다.

중요한 것은 모든 내부 환경을 Cloud에 복제하는 것이 아니다.

**재현 가능한 검증과 내부 환경에서만 가능한 검증을 분리하는 것**만으로도 Cloud를 활용할 수 있다.

---

## 6. Human Steering이 많으면 비동기 위임의 이점이 줄어든다

Cloud Agent의 장점 중 하나는 Task를 맡기고 개발자가 다른 일을 할 수 있다는 점이다.

하지만 작업 중간에 사람이 계속 방향을 바꿔야 한다면 이 장점은 작아진다.

예를 들어 UI 구조를 탐색한다고 하자.

```text
A안 구현
→ 사람 확인
→ B안으로 변경
→ 다시 확인
→ 데이터 흐름도 변경
```

이 작업은 짧은 주기로 상호작용이 반복된다.

반대로 다음 Task는 중간 개입이 적다.

```text
Goal
AuthService가 Java 21에서 기존 Test를 모두 통과하도록 deprecated API를 수정한다.

Validation
./gradlew :auth:test
```

따라서 Local/Cloud 판단에서 다음 질문이 중요하다.

> 작업 중간에 사람이 얼마나 자주 개입해야 하는가?

이 책에서는 이를 `Human Steering`으로 부른다.

---

## 7. Context가 너무 크면 먼저 문제를 줄인다

Cloud Agent가 Repository를 읽을 수 있다고 해서 매 Task마다 전체 시스템을 다시 이해시키는 것이 유리한 것은 아니다.

다음 두 범위를 비교해 보자.

```text
작은 Context
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java
```

```text
큰 Context
- Auth
- Student
- Permission
- Payment
- Database Schema
- Common Exception
- External API
- Admin Web
```

전체 시스템 관계를 보며 방향을 정해야 한다면 Local에서 먼저 구조를 좁히는 편이 낫다.

```text
큰 문제
→ Local에서 분석/결정
→ 작은 Task로 분해
→ Cloud 위임
```

7장에서 Cloud Task의 Context 경계를 구체적으로 정의한다.

---

## 8. Agent 실행시간과 개발자 대기시간은 다르다

Cloud Task가 오래 걸린다고 개발자가 같은 시간 동안 멈춰 있어야 하는 것은 아니다.

다음은 설명용 예다.

```text
10:00 Cloud Task 위임
10:01 개발자는 다음 작업 시작
10:40 Cloud Task 완료
11:20 개발자가 결과 Review
```

이 경우 Cloud Task의 실행시간과 개발자의 실제 대기시간은 다르다.

```text
Agent Execution Time
= Worker가 Task를 수행한 시간

Developer Blocking Time
= 개발자가 해당 Task 때문에 실제로 멈춘 시간
```

Cloud 사용의 가치를 평가할 때는 `Agent가 몇 분 걸렸는가`만 보지 않는다.

Local CPU / RAM 점유를 얼마나 줄였는지, 개발자가 다른 작업을 계속할 수 있었는지도 함께 본다. 이 차이는 4장에서 더 자세히 다룬다.

---

## 9. Cloud가 항상 더 빠른 것은 아니다

Cloud에서는 Worker 준비, Repository Checkout, 환경 준비, 결과 회수 같은 비용이 생길 수 있다.

예를 들어 문구 한 줄을 수정하고 짧은 검증만 하면 되는 Task라면 Cloud 준비 비용이 작업 자체보다 클 수 있다.

반대로 장시간 Integration Test나 E2E처럼 Local 자원을 오래 점유하는 작업은 Cloud 분리의 이점이 커질 수 있다.

따라서 다음 두 질문은 다르다.

```text
Cloud에서 실행할 수 있는가?

Cloud로 보내는 것이 유리한가?
```

첫 번째가 가능하더라도 두 번째의 답이 `아니오`일 수 있다.

5장에서는 이 판단을 Local / Cloud / Hybrid / Runner-first Routing Framework로 정리한다.

---

## 10. campus-platform에서 실행 위치를 나누면

`campus-platform`에 다음 작업이 있다고 가정하자.

| Task | 기본 위치 | 이유 |
| --- | --- | --- |
| 신규 인증 구조 설계 | Local | 넓은 Context와 Human Steering 필요 |
| expired token 수정 | Cloud Agent 후보 | 작은 Scope, 독립 검증 가능 |
| 전체 Unit Test | Cloud Runner | 결정론적 검증, Compute 중심 |
| Web E2E | Cloud Runner | Browser / 장시간 검증 |
| Tibero 실제 Migration 검증 | Local / Hybrid | 내부 DB 의존 |
| HSM 오류 분석 | Local | 내부 장비 / 네트워크 의존 |
| Docker Build | Cloud Runner | 독립 실행 가능 |

이 표는 고정 규칙이 아니다.

프로젝트의 네트워크와 실행환경, 보안 정책, Task 크기에 따라 결과가 달라질 수 있다.

중요한 것은 제품 이름이 아니라 Task 특성을 실행 위치에 연결하는 것이다.

---

## 11. 이 장에서 기억할 질문

Local과 Cloud를 비교할 때 다음 항목을 먼저 본다.

```text
현재 Local 상태가 필요한가?
Internal Network가 필요한가?
Human Steering이 많은가?
Context가 큰가?
독립적으로 검증 가능한가?
장시간 Compute가 필요한가?
Git으로 기준점을 전달할 수 있는가?
Cloud 준비 비용보다 Task 가치가 큰가?
```

이 장에서는 판단 재료만 만들었다.

5장에서는 이 질문을 실제 Routing 순서로 정리하고, 17장에서는 반대로 Cloud Task를 중단하거나 Local로 Fallback해야 하는 조건을 다룬다.

다음 장에서는 Cloud 환경에서 사용되는 CPU, RAM, Disk와 LLM Token을 분리해 본다.

---

## 참고자료

제품별 기능은 변경될 수 있으므로 본문에서는 실행 위치에 따른 공통 차이만 사용한다. 제품 사례의 세부 사실은 `research/chapter-02-local-cloud-official-sources.md`에서 기준일과 출처를 관리한다.
