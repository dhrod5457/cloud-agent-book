# 5장 설계 - Task Routing: Local인가 Cloud인가

## 장의 목표

Cloud Agent 활용에서 가장 중요한 실전 판단은 `Cloud를 쓸 수 있는가`가 아니라 `이 Task를 Cloud로 보내는 것이 실제로 유리한가`이다.

이 장에서는 작업 특성을 기준으로 다음 실행 경로를 선택하는 판단 체계를 만든다.

```text
Task
  ↓
Task Classification
  ↓
+---------+---------+---------+
|         |         |         |
Local    Cloud    Hybrid    Runner-first
```

4장에서 Cloud의 독립 실행환경과 비동기/병렬 실행 가치를 설명했다면, 5장에서는 그 가치를 어떤 Task에 적용해야 하는지 결정한다.

핵심 질문:

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

---

## 핵심 주장

> Cloud Agent를 잘 사용하는 핵심은 Agent 수를 늘리는 것이 아니라 어떤 작업을 Cloud로 보낼지 결정하는 것이다.

그리고 다음 원칙을 사용한다.

> Task는 Local 또는 Cloud 중 하나에 영구적으로 속하지 않는다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

예:

```text
Local
→ 요구사항 분석
→ Architecture 결정
→ 핵심 코드 작성

Cloud
→ 전체 테스트
→ Integration / E2E
→ Docker Build
→ CI 실패 수정

Local
→ 내부망 검증
→ 최종 Review
→ Merge
```

---

## 독자가 얻는 것

- Task의 특성을 보고 Local / Cloud / Hybrid를 선택할 수 있다.
- Human Steering이 많이 필요한 작업을 Cloud에 보내지 않을 수 있다.
- 내부망, 큰 Context, 파일 충돌, 재현 불가 문제를 판단할 수 있다.
- 장시간 Build/Test처럼 Cloud에 유리한 Task를 식별할 수 있다.
- 너무 작은 Task와 너무 큰 Task의 Cloud 비용을 비교할 수 있다.
- Cloud Agent가 아니라 일반 Runner로 충분한 Task를 구분할 수 있다.
- 하나의 Task를 단계별로 Local과 Cloud 사이에서 handoff할 수 있다.

---

# 5.1 Task Routing은 모델 선택이 아니라 실행 위치 선택이다

같은 모델을 사용해도 실행 위치에 따라 가능한 작업이 다르다.

Routing 판단에서 먼저 보는 항목:

- Repository 상태
- Internal Network 필요 여부
- Scope 명확성
- Context 크기
- Human Steering 빈도
- 독립 검증 가능 여부
- 실행시간
- File Conflict 가능성
- Git을 통한 결과 회수 가능 여부

판단 예:

```text
Architecture 방향 탐색
→ Local

전체 Unit Test
→ Cloud Runner

작은 독립 Bug Fix
→ Cloud Agent

Tibero/HSM 포함 문제
→ Local 또는 Hybrid
```

---

# 5.2 Local에 적합한 Task

다음 조건이 많을수록 Local에 남기는 편이 유리하다.

- 요구사항이 아직 불명확함
- 개발자의 질문과 수정이 빠르게 반복됨
- 여러 모듈을 동시에 넓게 이해해야 함
- 로컬 미커밋 상태에 의존함
- VPN / 내부망 / 사내 DB / HSM 접근 필요
- 재현 절차 자체가 불명확함
- Architecture 판단 비중이 높음
- 수정 범위가 너무 작아 Cloud 시작 비용이 더 큼

예:

```text
새 인증 구조를 어떻게 설계할지 논의
→ Local

간헐적으로 발생하는 내부 HSM 오류 분석
→ Local

한 줄 설정 수정
→ Local
```

Local의 핵심 장점은 `개발자와 Agent 사이의 짧은 feedback loop`다.

---

# 5.3 Cloud에 적합한 Task

다음 조건이 많을수록 Cloud에 보내기 좋다.

- Scope가 명확함
- 완료 조건을 기계적으로 정의 가능
- 중간 Human Steering이 적음
- 독립적인 Branch/Workspace에서 작업 가능
- Git을 통해 입력과 결과를 전달 가능
- Build/Test 실행시간이 김
- CPU/RAM을 많이 사용함
- 다른 작업과 파일 충돌이 적음
- 실패 시 로그/Artifact로 결과를 확인 가능

대표 작업:

- Unit Test 추가
- Integration Test
- E2E
- Docker Build
- Migration Validation
- Static Analysis / Lint
- 반복 Refactoring
- 작은 Bug Fix
- Dependency Update
- Documentation
- CI Failure Fix

---

# 5.4 Hybrid가 필요한 Task

일부 작업은 Local 또는 Cloud 하나만으로 끝나지 않는다.

예:

```text
Local
→ 요구사항 분석
→ 내부 시스템 제약 확인
→ Task 분리
      ↓
Cloud
→ 코드 수정
→ Unit / Integration Test
→ Docker Build
      ↓
Local
→ Tibero / HSM / Jenkins 최종 검증
```

Hybrid를 선택하는 대표 조건:

- 코드 변경 자체는 Cloud에서 가능
- 최종 검증은 내부망에서만 가능
- Cloud에서 대부분의 자동 테스트 수행 가능
- 마지막 단계에서만 Local 시스템 의존

13장에서 전체 Handoff Workflow를 상세히 설명한다.

---

# 5.5 먼저 Runner로 끝낼 수 있는지 확인한다

모든 Cloud Task가 LLM Agent를 필요로 하는 것은 아니다.

```text
Task
  ↓
Deterministic?
  ├─ YES → Runner
  └─ NO  → Agent 후보
```

예:

```text
./gradlew test
→ Runner

Docker image build
→ Runner

Migration validation
→ Runner

실패 원인 분석 후 코드 수정
→ Agent
```

즉 Routing은 다음 순서로 보는 것이 좋다.

```text
1. Runner로 해결 가능한가?
2. Cloud에 독립 실행 가능한가?
3. Human Steering이 필요한가?
4. Internal Network가 필요한가?
```

10장에서 Runner-first를 상세히 다룬다.

---

# 5.6 Routing Decision Matrix

책에서 사용할 기본 판단표:

| 조건 | Local | Cloud | Hybrid |
| --- | --- | --- | --- |
| Scope 명확 | 가능 | 유리 | 유리 |
| 요구사항 불명확 | 유리 | 불리 | 불리 |
| Human Steering 잦음 | 유리 | 불리 | 가능 |
| 큰 Context 필요 | 유리 | 불리 | 가능 |
| 장시간 Build/Test | 가능 | 유리 | 유리 |
| CPU/RAM 많이 사용 | 가능 | 유리 | 유리 |
| 내부망/VPN 필요 | 유리 | 불리 | 유리 |
| Git으로 결과 회수 가능 | 가능 | 유리 | 유리 |
| 독립 검증 가능 | 가능 | 유리 | 유리 |
| 동일 파일 충돌 가능성 큼 | 유리 | 불리 | 불리 |
| 작은 독립 Bug Fix | 가능 | 유리 | 가능 |
| Architecture 판단 | 유리 | 불리 | 가능 |

이 표는 절대 규칙이 아니다.

프로젝트별 실행시간, Cloud start overhead, 보안 정책, Repository 규모에 따라 가중치를 조정한다.

---

# 5.7 Task 크기가 너무 작아도 비효율적이다

Cloud에는 고정 overhead가 있다.

- Session/Worker 시작
- Repository checkout
- Branch 준비
- Environment 준비
- Context loading
- 결과 Commit/PR

따라서 다음과 같은 Task는 Local이 더 빠를 수 있다.

```text
변경 파일 1개
수정 1줄
검증 명령 10초
```

Cloud에 보내면 작업 자체보다 준비 비용이 더 클 수 있다.

개념:

```text
너무 작은 Task
→ Cloud Overhead > Task Work
```

절대적인 `몇 분 이하` 같은 규칙은 두지 않는다.

실제 프로젝트에서 측정한다.

---

# 5.8 Task가 너무 커도 비효율적이다

반대로 Task가 지나치게 크면 다음 비용이 증가한다.

- Repository 탐색
- Context
- Token
- 변경 파일 수
- Retry 범위
- Test 범위
- Review 부담
- Merge Risk

좋지 않은 예:

```text
프로젝트 전체 구조를 개선해.
```

권장 예:

```text
AuthService의 expired token 처리 수정
```

또는:

```text
attendance 모듈의 Java 21 호환성 수정
```

개념적으로는 다음 영역을 찾는다.

```text
작은 Task
→ overhead 과다

적정 Task
→ 독립 실행 / 검증 가능

큰 Task
→ Context / Retry / Review 비용 과다
```

7장에서 Task Contract로 구체화한다.

---

# 5.9 Human Steering 빈도를 중요한 기준으로 본다

Cloud Agent는 비동기 위임에 유리하지만 작업 중 사람의 판단을 계속 요구하면 장점이 줄어든다.

예:

### Local에 적합

```text
Developer:
이 화면은 A와 B 중 어느 UX가 맞는지 먼저 비교해보자.

Agent:
A안을 구현해볼까요?

Developer:
아니, 먼저 데이터 흐름부터 바꿔보자.
```

짧은 상호작용이 반복되는 작업이다.

### Cloud에 적합

```text
Goal:
AuthService expired token → 401

Validation:
./gradlew test --tests AuthServiceTest.expiredToken
```

완료 조건이 고정되어 있어 중간 질문이 적다.

핵심 기준:

> 사람이 작업 중간에 몇 번이나 개입해야 하는가?

---

# 5.10 Context 크기를 Routing 기준으로 사용한다

Cloud Agent에 보내기 전에 Agent가 이해해야 하는 범위를 본다.

작은 Context:

```text
AuthService.java
JwtTokenProvider.java
AuthServiceTest.java
```

→ Cloud에 적합

큰 Context:

```text
인증
학생
권한
결제
공통 예외
DB schema
외부 연동
```

전체 구조를 함께 판단해야 한다면 Local에서 개발자와 탐색하는 편이 나을 수 있다.

Cloud로 보낼 경우 큰 Task를 작은 Task로 먼저 분해할 수 있는지도 판단한다.

---

# 5.11 File Conflict와 Dependency를 본다

독립 Branch를 사용한다고 논리적 충돌까지 없어지는 것은 아니다.

좋은 Cloud Task:

```text
Agent A → attendance test
Agent B → notification test
```

충돌 가능성이 낮다.

좋지 않은 Cloud 병렬 Task:

```text
Agent A → UserService 구조 변경
Agent B → UserService 구조 변경
Agent C → UserService 테스트 구조 변경
```

같은 파일과 설계 결정을 공유한다.

Routing 시 확인:

- 변경 예상 파일
- shared/common module
- migration/schema
- 동일 DTO/API
- 순서 의존성

병렬화 상세는 11~12장에서 다룬다.

---

# 5.12 Git으로 결과를 돌려받을 수 있는가

Cloud Task는 Remote Worker 작업이므로 입력과 결과가 명확한 경계를 가져야 한다.

권장 구조:

```text
Input
- Git SHA
- Branch
- Task

Cloud Work
- Change
- Test

Output
- Commit
- Test Result
- Artifact
- PR
```

로컬의 미커밋 파일, 임시 DB 상태, IDE 내부 상태처럼 Git으로 전달하기 어려운 입력에 강하게 의존하면 Cloud 적합도가 낮아진다.

11장에서는 Git을 Local과 Cloud 사이의 Handoff Boundary로 상세히 설명한다.

---

# 5.13 Routing Score는 참고용으로만 사용한다

팀에서는 단순 체크리스트나 점수를 사용할 수 있다.

예:

```text
Scope 명확성           +2
독립 검증 가능          +2
장시간 실행             +1
CPU/RAM 사용 큼         +1
Human Steering 많음     -2
Internal Network 필요   -3
큰 Context 필요         -2
File Conflict 높음      -2
```

점수가 높으면 Cloud 후보로 볼 수 있다.

하지만 점수 하나로 자동 결정하지 않는다.

예를 들어 `Internal Network 필수` 하나만으로 Cloud 실행이 불가능할 수도 있다.

따라서 Score보다 다음을 우선한다.

- Hard Constraint
- Task Independence
- Verification 가능성
- Handoff 가능성

---

# 5.14 campus-platform Routing 예제

| Task | 판단 | 이유 |
| --- | --- | --- |
| 전체 Unit Test | Cloud Runner | 장시간, 독립 검증 가능 |
| AuthService expired token bug | Cloud Agent | Scope 작고 단일 테스트 가능 |
| 신규 인증 Architecture 설계 | Local | 큰 Context, Human Steering 필요 |
| Tibero 실제 DB 검증 | Local | 내부망/DB 접근 필요 |
| HSM AP 통합 검증 | Local | 내부 장비 접근 필요 |
| Web E2E | Cloud Runner | Browser/CPU 작업, 독립 검증 가능 |
| Docker Build | Cloud Runner | deterministic compute |
| Dependency Update | Cloud → 필요 시 Agent | 먼저 Runner 검증, 실패 시 분석 |
| DB migration 작성 + 실제 Tibero 검증 | Hybrid | 작성/기초 테스트는 Cloud, 실제 검증은 Local |
| 한 줄 문구 수정 | Local | Cloud overhead가 더 큼 |

이 예제를 통해 `Cloud에 보낼 수 있다`와 `Cloud에 보내는 것이 유리하다`를 구분한다.

---

# 좋은 사례와 나쁜 사례

## 사례 A - 무조건 Cloud

좋지 않은 방식:

```text
모든 Issue
→ Cloud Agent
```

권장 방식:

```text
Issue
→ Task Classification
→ Local / Runner / Cloud Agent / Hybrid
```

## 사례 B - 큰 Task

좋지 않은 방식:

```text
프로젝트 인증 전체를 개선해.
```

권장 방식:

```text
Architecture 판단 → Local

Task 1: expired token fix → Cloud
Task 2: refresh token test → Cloud
Task 3: 전체 regression → Runner
```

## 사례 C - 내부망 Task

좋지 않은 방식:

```text
Cloud Agent
→ 접근 불가능한 HSM 오류를 계속 추측
→ Retry
→ Token 소비
```

권장 방식:

```text
Local Agent
→ HSM 실제 상태 확인
→ 원인 범위 축소
→ Cloud에 보낼 수 있는 코드 수정만 분리
```

---

# 다른 장과의 연결

## 4장

Cloud의 독립 실행환경과 시간/병렬성의 가치를 설명했다.

5장은 어떤 Task가 그 이점을 얻는지 판단한다.

## 6장

5장에서 만든 판단 기준을 실제 작업 유형별로 적용한다.

## 7장

Cloud로 보내기로 결정한 Task를 작은 Task Contract로 만드는 방법을 설명한다.

## 10장

LLM이 필요 없는 Task를 Runner로 보내는 전략을 구체화한다.

## 13장

하나의 작업이 Local과 Cloud 사이를 이동하는 전체 Handoff Workflow를 설명한다.

## 17장

Cloud에 보내지 말아야 할 상황을 더 명시적으로 정리한다.

---

# 본문에서 의도적으로 다루지 않을 내용

- 특정 제품의 자동 Task Routing 기능
- 특정 Model 성능 비교
- Agent 조직 구조
- 일반적인 Scheduler 구현
- Agent Platform Control Plane
- 복잡한 비용 최적화 수학 모델

이 장의 목적은 개발자가 실제 Task를 보고 실행 위치를 판단하는 기준을 만드는 것이다.

---

# 장의 결론 메시지

이 장은 다음 세 문장으로 정리한다.

> Cloud Agent를 잘 사용하는 핵심은 Agent 수를 늘리는 것이 아니라 어떤 작업을 Cloud로 보낼지 결정하는 것이다.

> Task는 Local 또는 Cloud 중 하나에 영구적으로 속하지 않는다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

> Cloud에 보낼 수 있는 Task와 Cloud에 보내는 것이 유리한 Task는 다르다.
