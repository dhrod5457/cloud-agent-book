# 6장. Cloud에 보내기 좋은 개발 작업

5장에서 작업을 `Local / Cloud / Hybrid / Runner-first` 중 어디에 배치할지 판단하는 기준을 만들었다.

이제 그 기준을 실제 개발 작업에 적용한다.

중요한 것은 작업 이름만 보고 클라우드 에이전트를 호출하지 않는 것이다.

같은 `Test 작업`도 실제 작업 흐름에서는 다음처럼 나뉠 수 있다.

```text
Test 실행
→ Cloud Runner

실패 원인 분석
→ Cloud Agent

수정 후 재검증
→ Cloud Runner
```

이 장에서는 작업을 세 범주로 본다.

```text
Deterministic Work
→ Runner / Tool

Reasoning + Code Change
→ Cloud Agent

Internal / Interactive Work
→ Local 또는 Hybrid
```

> 클라우드에 보내기 좋은 작업과 클라우드 에이전트가 직접 해야 하는 작업은 같은 개념이 아니다.

> 실행기가 할 수 있으면 실행기에게 맡긴다.

> 에이전트는 판단과 수정이 필요한 구간에만 사용한다.

10장에서는 이 원칙을 Runner-first 실행 구조로 더 구체화한다. 이번 장에서는 **실제 개발 작업을 누가 실행할지 정리한 목록**을 만든다.

---

## 1. 작업 이름보다 실행 구조를 본다

의존 패키지 갱신을 예로 들어보자.

```text
Version Update
      ↓
Cloud Runner
→ Build / Test
      ↓
PASS ─────────→ Done
FAIL
 ↓
Failure Summary
 ↓
Cloud Agent
 ↓
Compatibility Fix
 ↓
Cloud Runner 재검증
```

처음부터 에이전트가 저장소 전체를 분석할 필요는 없다.

작업을 볼 때 다음 질문을 사용한다.

```text
실행 명령이 명확한가?
판정 기준이 명확한가?
코드 변경이 필요한가?
판단이 필요한가?
내부 시스템 의존성이 있는가?
어떤 Evidence를 남겨야 하는가?
```

이 질문을 빌드, 테스트, E2E, 마이그레이션, 오류 수정 같은 실제 작업에 적용한다.

---

## 2. Build는 Runner 작업이다

Java / Spring Boot 프로젝트의 빌드는 보통 명령과 판정 기준이 명확하다.

```bash
./gradlew clean build
```

기본 흐름:

```text
Git SHA
  ↓
Cloud Runner
  ↓
Build
  ↓
PASS / FAIL
```

PASS라면 에이전트를 호출하지 않는다.

FAIL이고 코드 수정이 필요하면 실패 요약을 에이전트에게 넘긴다.

```text
BUILD: FAIL
AuthService.java:142
cannot find symbol
JwtExpiredException
```

### 기본 Evidence

```text
Git SHA
Build Status
Duration
Artifact Path
Compile Error Summary
```

빌드 방법이 이미 표준화되어 있다면 에이전트에게 `어떻게 빌드해야 할지` 다시 추론시키지 않는다.

---

## 3. Unit Test는 Cloud Compute에 잘 맞는다

단위 테스트는 실행 명령과 성공 조건이 명확하다.

```bash
./gradlew test
```

또는 대상만 좁혀 실행할 수 있다.

```bash
./gradlew test --tests AuthServiceTest
```

기본 구조:

```text
Cloud Runner
→ Unit Test
→ PASS → 종료
→ FAIL → 실패 Test 목록
```

에이전트가 필요하다면 실패한 테스트부터 본다.

```text
AuthServiceTest.expiredToken
expected: 401
actual: 200
```

전체 로그를 처음부터 맥락 정보에 넣는 방법은 피한다. 도구 출력을 줄이는 방법은 8장에서 다룬다.

단위 테스트는 다음 조건에서 클라우드 활용 가치가 커진다.

```text
Test 수가 많음
실행시간이 김
모듈별 분할 가능
Local CPU / RAM 점유가 큼
```

---

## 4. Integration Test는 재현 가능한 환경이 핵심이다

통합 테스트는 단위 테스트보다 실행환경 의존성이 크다.

예:

```text
Spring Boot
PostgreSQL
Redis
Kafka
Testcontainers
Mock HTTP Server
```

클라우드에서 이 환경을 재현할 수 있다면 클라우드 실행기에 적합하다.

```text
Prepared Environment
        ↓
Integration Runner
        ↓
Testcontainers / DB / Mock
        ↓
Integration Test
```

반대로 실제 내부 자원이 필수라면 Hybrid가 된다.

```text
Cloud
→ 일반 Integration Validation

Local / Internal
→ Tibero / Internal API 최종 검증
```

통합 테스트의 핵심 질문은 `Cloud에서 돌릴 수 있는가`보다 **클라우드에서 필요한 상태를 재현할 수 있는가**다.

---

## 5. E2E와 Browser Test는 Cloud 분리에 적합하다

브라우저 E2E는 실행시간과 자원 점유가 크고 화면 캡처, 영상, 실행 추적 기록 같은 결과물을 남길 수 있다.

```text
Cloud Runner
→ Browser 시작
→ E2E 실행
→ Screenshot / Video / Trace 저장
```

시나리오가 독립적이라면 여러 작업자로 나눌 수 있다.

```text
Worker #1 → auth scenarios
Worker #2 → attendance scenarios
Worker #3 → admin scenarios
```

실패 시 에이전트에게 필요한 것은 전체 실행 추적 기록이 아니라 우선 다음 정보다.

```text
Failed Scenario
Expected / Actual
Screenshot
Console Error Summary
```

### 기본 Evidence

```text
Scenario Count
Failed Scenario List
Screenshot
Video / Trace Reference
Console Error Summary
```

UI 변경은 8장의 `Demos over Diffs`와 연결된다.

---

## 6. Docker Build도 Runner가 먼저다

```bash
docker build -t campus-platform:abc123 .
```

Docker 빌드는 일반적으로 다음 구조로 충분하다.

```text
Git SHA
→ Cloud Runner
→ Docker Build
→ PASS / FAIL
→ Image Digest / Build Log
```

실패했다고 해서 모두 에이전트가 코드를 고쳐야 하는 문제는 아니다.

```text
Dockerfile 문제
Dependency 문제
Registry / Network 문제
Base Image 문제
```

코드나 Dockerfile 수정이 필요한 경우에만 에이전트를 호출한다.

미리 준비한 실행환경과 Docker Layer Cache는 9장에서 다룬다.

---

## 7. Static Analysis와 Lint는 Tool-first다

다음은 기본적으로 일반 도구가 처리한다.

```text
Formatter
Lint
Checkstyle
SpotBugs
Architecture Check
Dependency Check
Security Scan
```

기본 흐름:

```text
Tool
→ PASS / FAIL
```

자동 수정이 가능하면 에이전트보다 도구를 먼저 사용한다.

```text
FAIL
 ↓
Auto-fix 가능?
 ├─ YES → Tool Fix → Recheck
 └─ NO  → Agent 후보
```

> 코드로 판정하고 코드로 수정할 수 있는 규칙은 에이전트보다 도구를 먼저 사용한다.

---

## 8. Migration은 작성과 검증을 분리한다

DB 구조 변경 작업은 한 덩어리의 에이전트 작업으로 보지 않는다.

```text
Migration 작성
→ Developer / Agent

Migration 검증
→ Cloud Runner
```

클라우드에서 일회용 DB를 사용할 수 있다면 다음을 검증할 수 있다.

```text
clean DB 적용
기존 DB Schema에서 upgrade
syntax
순서
기본 Integration Test
```

실제 대상이 내부 Tibero / Oracle이라면 마지막 경계만 로컬에 남긴다.

```text
Cloud
→ 일반 Migration Validation
        ↓
Local / Internal
→ 실제 DB 검증
```

에이전트가 마이그레이션을 작성했다고 에이전트의 자연어 판단으로 검증을 끝내지 않는다.

---

## 9. 반복 Refactoring은 Cloud Agent 후보다

판단과 코드 변경이 필요하지만 범위를 제한할 수 있는 리팩터링은 클라우드 에이전트에 잘 맞는다.

예:

```text
deprecated API 교체
Java 17 → 21 호환 수정
반복 import 변경
동일 패턴 수정
```

좋은 작업:

```text
attendance 모듈에서 deprecated API를 교체하고
./gradlew :attendance:test를 통과한다.
```

좋지 않은 작업:

```text
프로젝트 전체 구조를 더 좋게 리팩터링해.
```

클라우드 에이전트 후보의 조건:

```text
변경 패턴 명확
Scope 제한
검증 명령 존재
독립 Branch 가능
Review 가능한 Diff
```

시스템 구조 방향은 로컬에서 정하고 반복 실행 부분만 클라우드로 분리한다.

---

## 10. 작은 Bug Fix는 재현 가능성이 먼저다

클라우드 에이전트에 보내기 좋은 오류는 실패와 완료 조건이 명확하다.

```text
Failure
AuthServiceTest.expiredToken
expected: 401
actual: 200

Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

기본 흐름:

```text
Reproducible Failure
       ↓
Cloud Agent
       ↓
Analyze / Fix
       ↓
Cloud Runner
       ↓
PASS
```

반대로 다음 오류는 로컬에 더 가깝다.

```text
Production에서만 간헐 발생
실제 HSM 의존
내부 DB 상태 의존
재현 절차 불명확
여러 시스템을 동시에 조사해야 함
```

클라우드 에이전트의 모델 성능보다 **재현 가능성**이 먼저다.

---

## 11. Documentation은 사실 기반 갱신부터 보낸다

클라우드에 적합한 문서 작업:

```text
API 변경에 맞춘 README 수정
특정 Module 문서 갱신
Release Note 초안
변경 코드 기반 운영 문서 수정
```

이 작업은 저장소와 커밋을 기준으로 범위를 제한하기 쉽다.

반대로 다음은 로컬 중심이 자연스럽다.

```text
새 Architecture 원칙 정의
조직 전체 개발 정책 결정
요구사항이 정해지지 않은 문서
```

즉 **사실 기반 갱신**과 **방향 결정**을 구분한다.

---

## 12. PR Review는 보조 Worker로 사용한다

클라우드 에이전트는 PR 검토를 보조할 수 있다.

검토 후보:

```text
변경 범위
Test 누락
명확한 오류
규칙 위반
문서 / 코드 불일치
```

권장 순서:

```text
PR
→ Deterministic Checks
→ Cloud Review Agent
→ Findings
→ Human Review
```

클라우드 에이전트의 검토를 최종 승인과 동일하게 취급하지 않는다.

기계적으로 검증할 수 있는 항목은 CI / 도구가 먼저 처리하고, 에이전트는 의미 판단을 보조한다.

---

## 13. Dependency Update는 Runner-first 대표 사례다

```text
Dependency Version 변경
      ↓
Cloud Runner
→ Build / Test
      ↓
PASS → PR
FAIL
 ↓
Failure Summary
 ↓
Cloud Agent Compatibility Fix
 ↓
Cloud Runner 재검증
```

PASS하면 LLM이 필요 없다.

FAIL일 때도 에이전트에게 저장소 전체를 다시 설명하지 않는다.

```text
변경 Dependency
이전 / 신규 Version
실패 Command
실패 Test
관련 파일
```

이 패턴은 이벤트로 시작하는 작업 흐름에도 그대로 연결된다.

---

## 14. CI Failure Fix는 Agent-on-failure에 잘 맞는다

```text
Git Push / PR
     ↓
    CI
     ↓
PASS → Done
FAIL
 ↓
Failure Summary
 ↓
Cloud Agent
 ↓
Fix
 ↓
Cloud Runner
 ↓
PASS
```

이 구조에서 에이전트는 상시 실행되지 않는다.

실패가 발생했고 코드 판단이 필요한 경우에만 호출한다.

CI 실패, 검토 의견, 야간 정기 테스트 같은 이벤트가 실제 작업을 만드는 구조는 14장에서 다룬다.

---

## 15. 하나의 작업 안에서도 실행 주체는 바뀐다

`Unit Test는 Runner`, `Bug Fix는 Agent`처럼 작업 이름에 실행 주체를 영구적으로 붙이지 않는다.

하나의 작업 흐름 안에서도 바뀐다.

```text
Unit Test 실행
→ Cloud Runner

실패 원인 분석
→ Cloud Agent

코드 수정
→ Cloud Agent

재검증
→ Cloud Runner
```

마이그레이션도 마찬가지다.

```text
Migration 작성
→ Developer / Agent

Disposable DB Validation
→ Cloud Runner

Tibero 실제 검증
→ Local / Internal
```

실행 주체는 작업의 **현재 단계**에 따라 선택한다.

---

## 16. 작업별 기본 Catalog

| 작업 | 기본 실행 주체 | 에이전트 호출 조건 | 대표 검증 근거 |
| --- | --- | --- | --- |
| 빌드 | 클라우드 실행기 | 코드 / 빌드 수정 필요 | 상태, 결과물, 오류 요약 |
| 단위 테스트 | 클라우드 실행기 | 실패 원인 분석 / 수정 | 실패한 테스트, JUnit 결과 |
| 통합 테스트 | 클라우드 실행기 | 재현할 수 있는 코드 문제로 인한 실패 | 테스트 결과, 로그 / 결과물 |
| E2E | 클라우드 실행기 | UI / 코드 분석 필요 | 화면 캡처, 영상 / 실행 추적 기록 |
| Docker 빌드 | 클라우드 실행기 | Dockerfile / 코드 수정 필요 | 이미지 식별값(Image Digest), 빌드 결과 |
| Lint / Static Analysis | 도구 / 실행기 | 자동 수정 불가 | 위반 사항 요약 |
| 마이그레이션 검증 | 클라우드 실행기 | 마이그레이션 수정 필요 | 적용 결과, 테스트 결과 |
| 반복 리팩터링 | 클라우드 에이전트 | 처음부터 판단 / 수정 필요 | 커밋, 테스트 결과 |
| 작은 오류 수정 | 클라우드 에이전트 | 재현 가능해야 함 | 커밋, 대상 테스트 |
| 문서 작성 | 클라우드 에이전트 후보 | 사실 기반 범위 명확 | 변경 파일, 검토 |
| PR 검토 | 클라우드 에이전트 보조 | 의미 검토 필요 | 검토 결과 |
| 의존 패키지 갱신 | Runner-first | FAIL일 때 | 빌드 / 테스트 결과 |
| CI 실패 수정 | Agent-on-failure | 코드 문제로 실패했을 때 | 수정 커밋, 재검증 |

이 표는 기본값이다.

내부망, 사람의 중간 판단과 방향 조정, 맥락 정보 크기 같은 조건은 5장의 실행 위치 결정 기준을 다시 적용한다.

---

## 17. campus-platform Task Catalog

프로젝트에서 반복되는 작업은 이름과 실행 방식을 미리 정해둘 수 있다.

```text
RUN-BUILD
→ ./gradlew build

RUN-UNIT
→ ./gradlew test

RUN-INTEGRATION
→ integrationTest + Testcontainers

RUN-E2E
→ npx playwright test

RUN-DOCKER
→ docker build

FIX-BUG
→ Failure Summary + Relevant Files + Target Validation

REFACTOR-MODULE
→ One Module + Module Test
```

마이그레이션은 다음처럼 나눈다.

```text
Cloud
→ Disposable DB Validation

Local / Internal
→ Tibero 실제 검증
```

이런 목록이 있으면 매번 에이전트에게 `무엇을 어떻게 실행할지` 처음부터 설명할 필요가 줄어든다.

15장에서는 이 목록을 `campus-platform` 전체 운영 모델에 배치한다.

---

## 18. 이 장에서 기억할 실행 순서

실제 작업을 받으면 다음처럼 생각한다.

```text
1. Cloud로 보낼 가치가 있는가?
2. Tool / Runner만으로 처리 가능한가?
3. 실패 또는 코드 판단이 발생했는가?
4. Agent가 필요한가?
5. Agent 수정 후 Cloud Runner로 재검증했는가?
6. Evidence를 남겼는가?
```

한 줄로 줄이면 다음과 같다.

```text
Cloud Runner
→ 필요한 순간에 Cloud Agent
→ 다시 Cloud Runner
```

다음 장에서는 에이전트에게 실제 수정 작업을 넘길 때 저장소 전체를 다시 탐색하지 않도록 `Task Contract`와 적은 양의 맥락 정보를 구성한다.