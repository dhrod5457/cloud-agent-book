# 2장 설계 - Local Agent와 Cloud Agent

## 장의 목표

Local Agent와 Cloud Agent를 제품 이름이 아니라 **실행 위치와 작업 특성**으로 구분한다.

이 장의 핵심 질문은 하나다.

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

Local과 Cloud를 경쟁 관계로 설명하지 않는다. 하나의 작업도 단계에 따라 Local → Cloud → Local로 이동할 수 있으며, 두 실행 위치를 조합하는 것이 책 전체의 기본 전제다.

---

## 핵심 주장

> Cloud Agent는 Local Agent를 대체하는 것이 아니다.

> Local에서는 설계와 통합을 하고, Cloud에서는 독립적인 작업을 병렬로 처리한다.

> Task는 Local 또는 Cloud 중 하나에 영구적으로 속하는 것이 아니다. 작업 단계에 따라 실행 위치를 이동할 수 있다.

이 문장들은 절대 규칙이 아니라 기본 판단 기준이다.

---

## 독자가 얻는 것

- Local Agent와 Cloud Agent의 차이를 실행 위치 관점에서 설명할 수 있다.
- 내부망, Context 크기, Human Steering, 작업 독립성에 따라 실행 위치를 선택할 수 있다.
- 하나의 Task를 Local → Cloud → Local로 handoff할 수 있다.
- Cloud에 보내면 오히려 비효율적인 작업을 구분할 수 있다.
- 이후 장의 Task Routing 기준을 이해할 수 있다.

---

## 비교 기준

| 관점 | Local Agent | Cloud Agent |
| --- | --- | --- |
| 실행 위치 | 개발자 PC / 내부 서버 | 원격 독립 실행환경 |
| 현재 Workspace | 직접 접근 | Git/Repository 기반으로 시작하는 경우가 많음 |
| 미커밋 상태 | 즉시 사용 가능 | 일반적으로 전달 과정 필요 |
| VPN/내부망 | 접근하기 쉬움 | 제한될 수 있음 |
| 사내 DB/HSM | 접근 가능 | 보통 제한됨 |
| Human Steering | 빠른 대화/수정 반복에 유리 | 명확한 Task 위임에 유리 |
| 장시간 실행 | 개발자 PC를 점유 | 개발자 PC와 분리 가능 |
| 병렬성 | 로컬 CPU/RAM 한계 | 독립 Worker로 확장 가능 |
| 작업 격리 | 직접 구성 필요 | 독립 Workspace/Container 활용 가능 |
| Cold Start | 기존 환경 사용 | 환경 준비 시간이 발생할 수 있음 |

세부 제품 구현에 따라 차이가 있으므로 표를 제품 보장사항처럼 사용하지 않는다.

---

# 절 구성

## 2.1 모델보다 실행 위치가 먼저다

같은 계열의 모델을 사용하더라도 Local과 Cloud의 작업 조건은 다르다.

Local에서는 다음을 바로 사용할 수 있다.

- 현재 IDE/Workspace
- 미커밋 변경
- 로컬 DB
- VPN
- 사내 API
- HSM
- 내부 Jenkins/Nexus

Cloud에서는 대신 다음 장점이 있다.

- 독립 작업공간
- 개발자 PC와 분리된 CPU/RAM
- 장시간 비동기 실행
- 여러 Worker 병렬 실행
- 작업 단위 폐기/재생성

따라서 Local/Cloud 선택을 모델 성능 비교로 축소하지 않는다.

## 2.2 Local Agent가 적합한 작업

기본적으로 다음 특성이 강할수록 Local에 남긴다.

- Architecture 설계
- 요구사항이 아직 불명확함
- 여러 모듈을 동시에 이해해야 함
- 개발자와 질문/수정이 빠르게 반복됨
- 내부망/VPN 의존
- 사내 DB/HSM/내부 API 의존
- 미커밋 로컬 상태가 중요함
- 재현이 어려운 환경 문제
- 최종 통합/Review

예:

```text
신규 인증 Architecture 설계
→ Local

HSM 연동 오류 조사
→ Local

Tibero 운영 환경에서만 발생하는 문제
→ Local
```

## 2.3 Cloud Agent가 적합한 작업

다음 조건을 만족할수록 Cloud에 보내기 쉽다.

- Scope가 명확함
- 완료 조건을 정의할 수 있음
- Git으로 작업 상태를 전달할 수 있음
- 독립적으로 검증 가능함
- 다른 Task와 파일 충돌이 적음
- 사람이 계속 개입할 필요가 없음
- Build/Test처럼 오래 걸리는 실행이 포함됨
- 병렬화 가능한 독립 작업임

대표 작업:

- Unit Test
- Integration Test
- E2E
- Build / Docker Build
- Static Analysis / Lint
- Migration Validation
- 반복 Refactoring
- 작은 Bug Fix
- 독립 Feature
- Documentation
- PR Review
- CI Failure 수정

구체적 작업 분류는 5장과 6장에서 확장한다.

## 2.4 Local → Cloud → Local Handoff

하나의 작업은 실행 위치를 이동할 수 있다.

```text
Local
요구사항 분석
→ Architecture 결정
→ 핵심 코드 작성
        |
        v
Cloud
전체 테스트
→ Integration Test
→ E2E
→ Docker Build
→ 독립 Refactoring
→ CI 문제 수정
        |
        v
Local
최종 Review
→ 내부망 검증
→ 통합
→ Merge
```

핵심:

> Task의 단계마다 가장 적합한 실행 위치를 선택한다.

이 흐름은 13장에서 실제 Hybrid Workflow로 확장한다.

## 2.5 Git은 Handoff의 전제가 된다

Cloud Worker가 Remote Repository의 clean state에서 시작하는 경우 Local의 미커밋 상태는 자동으로 전달되지 않는다.

따라서 일반적인 handoff는 다음 형태가 된다.

```text
Local
→ Test
→ Commit
→ Push

Git Repository

Cloud Worker
→ Checkout / Branch
→ Work
→ Test
→ Commit
→ Push / PR
```

여기서는 개념만 소개한다.

Branch, Worktree, Session과 Git의 관계는 11장에서 상세히 다룬다.

## 2.6 내부망은 중요한 Routing 조건이다

`campus-platform` 예:

### Local 중심

- Tibero 실제 연동
- HSM
- VPN 내부 학사 API
- 내부 Jenkins
- 사내 Nexus 문제

### Cloud 중심

- PostgreSQL/Testcontainers 기반 통합 테스트
- Unit Test
- Web E2E
- Docker Build
- 정적 분석
- 독립 코드 수정

Cloud에서 내부 시스템을 억지로 노출하는 대신 테스트 가능한 대체 경로를 만드는 것이 더 나을 수 있다.

## 2.7 Human Steering 비용을 본다

Cloud Agent가 독립 실행할 수 있으려면 Task가 충분히 명확해야 한다.

좋지 않은 Cloud Task:

```text
프로젝트 전체 구조를 좀 더 좋게 개선해.
```

```text
왜 시스템이 가끔 느린지 알아봐.
```

이런 작업은 탐색 중 사람의 질문/판단이 반복되므로 Local 작업에 더 적합할 수 있다.

반대로 다음은 Cloud에 보내기 쉽다.

```text
AuthService.expiredToken 테스트 실패를 수정한다.
관련 테스트를 통과한다.
DB Schema는 변경하지 않는다.
```

Task Contract 상세는 7장에서 다룬다.

## 2.8 Cloud의 시간은 개발자의 대기시간과 다르다

Cloud Agent의 장점 중 하나는 비동기성이다.

예:

```text
10:00 Cloud에 테스트 보강 작업 위임
10:01 개발자는 다음 Feature 작업
10:40 Cloud 작업 완료
11:20 개발자가 결과 Review
```

Cloud 작업이 40분 걸렸다고 개발자가 40분 동안 Blocked된 것은 아니다.

구분할 지표:

- Agent Execution Time
- Developer Blocking Time

이 개념은 4장에서 장시간 작업과 병렬성의 가치로 확장한다.

---

# 좋은 사례 / 나쁜 사례

## 사례 A - 내부 HSM 오류

나쁜 선택:

```text
Cloud Agent
→ HSM 접근 시도
→ Network/Auth 실패
→ 환경 문제 분석
→ Retry
```

권장:

```text
Local Agent
→ HSM 실제 환경에서 조사
```

## 사례 B - 전체 Unit Test

비효율:

```text
Local Agent
→ 개발자 MacBook CPU 점유
→ 전체 테스트 종료까지 다른 작업 영향
```

Cloud 활용:

```text
Cloud Worker
→ 전체 Unit Test

Local Developer
→ 다음 작업 계속
```

## 사례 C - Architecture 변경

나쁜 선택:

```text
Cloud Agent
→ 큰 Repository 전체 탐색
→ 여러 모듈 재설계
→ 반복 질문 불가
```

권장:

```text
Local Developer + Local Agent
→ 설계/경계 결정
→ 독립된 후속 Task만 Cloud로 분리
```

---

# 필요한 구조/그림

1. Local vs Cloud 비교표
2. Task 특성 → Local/Cloud Routing
3. Local → Cloud → Local Handoff
4. Internal Network Boundary
5. Agent Execution Time vs Developer Blocking Time

---

# 필요한 공식 자료 조사

제품 사례는 다음을 검증하는 데만 사용한다.

- Cloud Repository checkout 방식
- 비동기 Task 실행
- Branch/PR 반환
- Local/Cloud 접근 범위 차이

제품별 네트워크, 보안, Session 제한은 일반 원칙과 분리한다.

---

# 앞 장과 뒤 장의 연결

1장에서는 Cloud Agent를 Remote Worker로 정의했다.

3장에서는 Remote Worker 안의 CPU/RAM/Disk와 LLM Token을 분리한다.

5장에서는 이 장의 판단 기준을 실제 Task Routing 표로 구체화한다.

13장에서는 Local/Cloud handoff를 하나의 실제 Workflow로 완성한다.

---

# 본문에서 의도적으로 다루지 않을 내용

- Token/Compute 상세 계산
- Result Gateway 상세
- Prebuilt Environment/Cache
- Branch/Worktree 전략 상세
- PM Agent/Orchestration 일반론
- Agent Memory
- Agent Platform Architecture
