# 7장. Cloud Agent Task Contract: 작은 Task와 작은 Context

Cloud Agent에 작업을 맡길 때 필요한 것은 긴 Prompt가 아니다.

Repository 설명, Architecture, Coding Rule, 과거 장애 이력과 예상 원인을 한 번에 넣으면 정보량은 늘지만 Agent가 실제로 탐색해야 하는 범위도 함께 커진다.

이 장에서는 Cloud Worker가 바로 작업을 시작할 수 있도록 입력 경계를 정리한다. 이를 `Cloud Agent Task Contract`라고 부른다.

기본 형태는 다음과 같다.

```text
Task
Goal
Scope
Relevant Files
Forbidden Changes
Validation
Expected Result
Output / Evidence
```

필요하면 다음 정보를 추가한다.

```text
Base SHA / Branch
Environment
Related Document
Budget
```

핵심은 한 문장으로 정리할 수 있다.

> Task Contract의 목적은 Prompt를 길게 만드는 것이 아니라 Agent가 탐색해야 하는 범위를 줄이는 것이다.

---

## 1. Cloud Task는 Prompt가 아니라 작업 패키지다

다음 지시는 범위가 넓다.

```text
로그인 쪽에 문제가 있는 것 같은데 Repository 전체를 확인해서
관련된 부분을 분석하고 수정한 다음 테스트도 해줘.
```

Agent는 문제 정의, 관련 파일, 변경 범위, 검증 방법, 완료 조건을 모두 다시 추론해야 한다.

같은 작업을 다음처럼 바꿀 수 있다.

```text
Task
AuthService expired token 처리 수정

Goal
Expired JWT 요청이 HTTP 401을 반환한다.

Scope
auth 모듈의 token expiration 처리와 관련 테스트

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Forbidden Changes
- DB Schema
- OAuth flow
- 공통 Exception API

Validation
./gradlew test --tests AuthServiceTest.expiredToken

Expected Result
expired token → HTTP 401

Output / Evidence
- Result SHA
- Changed Files
- Validation Result
```

Task Contract는 Agent의 사고 과정을 대신 작성하는 문서가 아니다.

다음 네 가지를 고정하는 작업 패키지다.

```text
어디서 시작하는가?
어디까지 바꿀 수 있는가?
무엇이 성공인가?
무엇을 반환해야 하는가?
```

---

## 2. Goal과 Scope를 분리한다

Goal은 원하는 결과를 설명한다.

```text
Goal
Expired JWT 요청이 HTTP 401을 반환한다.
```

Scope는 탐색과 변경 범위를 설명한다.

```text
Scope
auth 모듈의 expired token 처리와 해당 regression test
```

Goal만 있으면 Agent가 목표를 달성하기 위해 어디까지 바꿔도 되는지 알기 어렵다.

반대로 Scope만 있으면 어떤 결과를 만들어야 하는지 불분명하다.

> Goal은 결과를 제한하고 Scope는 탐색과 변경 범위를 제한한다.

---

## 3. Relevant Files는 첫 번째 Context다

Agent가 Repository를 처음 열었을 때 관련 파일을 찾는 탐색 자체가 Context를 소비한다.

사람이 이미 시작점을 알고 있다면 그 정보를 전달한다.

```text
Relevant Files
- src/main/java/.../AuthService.java
- src/main/java/.../JwtTokenProvider.java
- src/test/java/.../AuthServiceTest.java
```

이 목록은 whitelist가 아니다.

예상하지 못한 Dependency가 있을 수 있으므로 다음 순서를 기본으로 한다.

```text
Relevant Files
      ↓
Direct Dependency
      ↓
Related Test / Document
      ↓
Wider Module Context
```

즉 Relevant Files는 `Initial Context Boundary`다.

작은 Context의 목적은 정보를 없애는 것이 아니라 필요한 정보까지 도달하는 경로를 짧게 만드는 것이다.

---

## 4. Forbidden Changes는 Diff를 작게 유지한다

Bug 하나를 고치면서 주변 구조까지 함께 정리하면 Review 범위가 빠르게 커진다.

예를 들어 expired token 오류를 수정하다가 다음 변경까지 섞일 수 있다.

```text
Exception hierarchy 정리
JWT 구조 변경
공통 API response 변경
DB Migration 추가
OAuth 설정 변경
```

각 변경이 타당해도 원래 Task와 섞이면 회귀 위험과 Review 비용이 커진다.

따라서 하지 말아야 할 영역을 함께 적는다.

```text
Forbidden Changes
- DB Schema
- OAuth flow
- 공통 Exception API
- public response format
```

필요하면 경로로 제한할 수 있다.

```text
Do Not Change
- db/migration/**
- common/exception/**
- security/oauth/**
```

목적은 Governance가 아니라 Task의 변경 경계를 지키는 것이다.

---

## 5. Validation은 실행 가능한 명령으로 준다

다음 완료 조건은 판정하기 어렵다.

```text
기능이 정상 동작하는지 확인한다.
```

가능하면 Validation을 명령으로 만든다.

```bash
./gradlew test --tests AuthServiceTest.expiredToken
```

필요하면 좁은 검증과 회귀 검증을 나눈다.

```text
Target Validation
./gradlew test --tests AuthServiceTest.expiredToken

Regression Validation
./gradlew test --tests AuthServiceTest
```

Agent가 코드를 읽고 `문제없어 보인다`고 판단하는 것과 실제 Test PASS는 다르다.

> 검증할 수 있는 것은 Agent에게 묻지 말고 실행한다.

이 Validation은 10장의 Runner-first 구조에서 그대로 사용한다.

---

## 6. Expected Result와 Evidence를 미리 정한다

Validation 명령만으로는 무엇이 성공인지 충분하지 않을 수 있다.

Expected Result는 관찰 가능한 결과로 작성한다.

```text
Expired token
→ HTTP 401

Valid token
→ 기존 성공 동작 유지
```

그리고 Task가 반환할 결과도 미리 정한다.

```text
Output / Evidence
- Result SHA
- Changed Files
- Validation Result
- Artifact Reference if generated
- Short Summary
```

다음 결과만 받는 것은 부족하다.

```text
수정 완료했습니다.
테스트도 정상입니다.
```

Remote Worker는 자연어 설명보다 검증 가능한 결과를 남겨야 한다.

8장에서는 이 Evidence를 작게 전달하면서 Raw Artifact를 보존하는 방법을 다룬다.

---

## 7. Context는 필요할 때 단계적으로 넓힌다

모든 문서와 Source를 처음부터 Context에 넣지 않는다.

예를 들어 Repository에 다음 안내가 있다고 하자.

```text
AGENTS.md
├─ Backend → docs/backend.md
├─ Authentication → docs/security/auth.md
├─ Database → docs/database.md
├─ Testing → docs/testing.md
└─ Deployment → docs/deployment.md
```

AUTH-142라면 다음 경로부터 시작할 수 있다.

```text
Task Contract
→ AGENTS.md
→ docs/security/auth.md
→ Relevant Files
```

문제 해결에 실패해도 바로 Repository 전체로 확대하지 않는다.

```text
Level 1
Task Contract + Relevant Files
      ↓
Level 2
Direct Dependency + Related Test
      ↓
Level 3
Related Module Documentation
      ↓
Level 4
Specific Failure Artifact / Log
      ↓
Level 5
Wider Module / Repository Context
```

이 흐름을 `Progressive Context`로 사용할 수 있다.

> Context를 줄이는 것뿐 아니라 필요할 때 가져오게 만든다.

---

## 8. Base SHA와 Environment를 명시한다

Cloud Worker가 Remote Repository에서 작업하려면 시작점을 고정해야 한다.

```text
Repository
campus-platform

Base SHA
4f29abc

Task Branch
agent/auth-expired-token-142
```

결과도 같은 작업 상태와 연결한다.

```text
Task ID
→ Base SHA
→ Branch
→ Result SHA
→ Validation Result
→ PR
```

Branch와 Runtime 격리는 11장에서 다룬다.

Environment는 설치 방법보다 이름으로 선택하는 편이 낫다.

```text
Environment
backend-test
```

환경 구성 자체는 9장의 Prepared Environment에서 관리한다.

---

## 9. Budget은 선택 필드다

실패와 수정을 반복할 수 있는 Task에는 중단 조건을 둘 수 있다.

설명용 예:

```yaml
budget:
  max_retry: 2
```

표준값은 아니다. 프로젝트에 따라 다음 기준을 사용할 수 있다.

```text
max retry
max wall-clock time
max token/cost
max changed files
```

목적은 `성공할 때까지 계속해` 같은 무제한 Task를 피하는 것이다.

구체적인 Failure Fingerprint와 중단 판단은 8장과 17장에서 이어서 다룬다.

---

## 10. Task Contract가 너무 커지면 Task를 다시 본다

다음처럼 Contract 자체가 커졌다고 하자.

```text
Relevant Files: 수십 개
Related Modules: 다수
Validation Commands: 다수
Forbidden Changes: 수십 개
```

문서 형식을 더 복잡하게 만들기 전에 Task가 독립 실행하기에 너무 큰지 확인한다.

```text
Task Contract 과대
      ↓
Scope 재검토
      ↓
분리 가능한가?
  ├─ YES → 작은 Task로 분해
  └─ NO  → Local / Hybrid 재검토
```

Task를 작게 만드는 것 자체가 목적은 아니다.

Cloud Worker가 사람의 지속적인 개입 없이 끝낼 수 있는 단위인지 확인하는 것이 목적이다.

---

## 11. AUTH-142 최소 Contract

이 책에서 반복해서 사용할 expired token 예제를 하나의 Task로 정리하면 다음과 같다.

```text
Task ID
AUTH-142

Task
Expired token 처리 수정

Goal
Attendance API에 expired JWT 사용 시 HTTP 401 반환

Base SHA
4f29abc

Scope
auth 모듈의 token expiration handling과 관련 test

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Forbidden Changes
- DB Schema
- OAuth flow
- common exception response format

Environment
backend-test

Validation
./gradlew :auth:test --tests AuthServiceTest.expiredToken

Expected Result
- expired token → 401
- valid token 기존 동작 유지

Output / Evidence
- Result SHA
- Changed Files
- Validation Result
```

이 장 이후에는 같은 배경을 반복하지 않고 `AUTH-142`로 참조한다.

---

## 12. 작은 Input에서 작은 Output으로

Task Contract는 Cloud Agent의 입력 경계를 만든다.

```text
Task Contract
→ Small Input / Context
        ↓
Cloud Runner / Agent
```

하지만 Runner가 만든 대형 로그를 그대로 Agent에게 돌려주면 Context는 다시 커진다.

다음 장에서는 반대 방향을 다룬다.

```text
Cloud Runner / Agent
        ↓
Result Gateway
→ Small Output / Evidence
```

7장은 `어디까지 읽고 바꿀 것인가`를 줄이고, 8장은 `무엇을 먼저 읽을 것인가`를 줄인다.
