# 7장. Cloud Agent Task Contract: 작은 Task와 작은 Context

Cloud Agent에 작업을 맡길 때 가장 흔한 실수 중 하나는 지시를 길게 쓰는 것이다.

Repository 설명, Architecture, Coding Rule, 관련 문서, 과거 장애 이력, 예상 원인까지 한 번에 넣으면 Agent가 더 잘할 것처럼 보인다.

하지만 Cloud Agent가 실제로 필요한 것은 `많은 정보`가 아니라 **작업을 시작할 수 있는 정확한 경계**다.

이 장에서는 그 경계를 `Cloud Agent Task Contract`라고 부른다.

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

필요한 경우 다음을 추가한다.

```text
Base Commit / Branch
Environment
Dependency
Related Document
Retry / Token Budget
```

핵심은 Prompt를 길게 만드는 것이 아니다.

> Task Contract의 목적은 Agent가 탐색해야 하는 범위를 줄이는 것이다.

그리고 한 가지를 더 기억해야 한다.

> 작은 Context는 정보를 없애는 것이 아니라 필요한 정보에 도달하는 경로를 짧게 만드는 것이다.

---

## 1. Cloud Task는 Prompt가 아니라 작업 패키지다

다음과 같은 지시를 생각해 보자.

```text
로그인 쪽에 문제가 있는 것 같은데 Repository 전체를 확인해서
관련된 부분을 분석하고 적절하게 수정한 다음 테스트도 해줘.
```

이 문장은 자연어로는 이해하기 쉽다.

하지만 Agent 입장에서는 결정해야 할 것이 너무 많다.

```text
문제의 정확한 정의는 무엇인가?
어느 모듈을 먼저 봐야 하는가?
관련 파일은 무엇인가?
어디까지 수정해도 되는가?
어떤 테스트를 실행해야 하는가?
무엇을 성공으로 판단해야 하는가?
결과를 어떤 형태로 반환해야 하는가?
```

이 질문을 Agent가 매 Task마다 Repository 전체를 읽으며 해결하면 Context 비용이 커진다.

Cloud에서는 Session이 바뀔 수 있고, Worker가 새로 만들어질 수 있다.

그때마다 같은 탐색을 반복하면 비효율적이다.

반대로 다음처럼 작업 패키지를 만들 수 있다.

```text
Task
AuthService expired token 처리 수정

Goal
Expired JWT 요청이 HTTP 401을 반환한다.

Scope
AuthService의 token expiration 처리와 관련 테스트만 수정한다.

Relevant Files
- AuthService.java
- JwtTokenProvider.java
- AuthServiceTest.java

Validation
./gradlew test --tests AuthServiceTest.expiredToken

Expected Result
expired token → HTTP 401

Forbidden Changes
- DB Schema
- OAuth flow
- 공통 Exception API

Output
- commit SHA
- changed files
- test result
- short summary
```

이 Task는 Agent의 사고 과정을 대신 작성한 것이 아니다.

Agent가 어디에서 시작하고, 어디까지 바꿔도 되며, 무엇으로 완료를 확인할지를 정의한 것이다.

---

## 2. Task와 Goal은 다르다

Task는 무엇을 할지 설명한다.

```text
Task
AuthService expired token 처리 수정
```

Goal은 최종 결과를 설명한다.

```text
Goal
Expired JWT 요청이 HTTP 401을 반환한다.
```

두 항목을 분리하는 이유는 단순하다.

`인증 문제 수정`이라는 Task만 있으면 Agent가 어떤 결과를 만들어야 하는지 넓게 해석할 수 있다.

반대로 Goal만 있으면 결과는 알지만 어떤 코드 범위를 수정할지는 불명확하다.

예를 들어 다음 Goal을 보자.

```text
Expired JWT → HTTP 401
```

이를 구현하는 방법은 여러 가지다.

```text
AuthService 수정
Filter 수정
Exception Handler 수정
공통 Response 구조 변경
JWT 전체 구조 재설계
```

Task Contract는 이 중 허용할 범위를 좁혀야 한다.

---

## 3. Scope는 탐색과 변경 범위를 제한한다

Scope는 `무엇을 고칠 것인가`보다 더 중요하게 사용할 수 있다.

예:

```text
Scope
AuthService token expiration handling과 관련 test만 수정한다.
```

이 한 줄은 Agent에게 다음 의미를 준다.

```text
인증 전체 Architecture를 바꾸지 않는다.
DB까지 내려가지 않는다.
OAuth flow를 재설계하지 않는다.
관련 테스트를 중심으로 확인한다.
```

좋지 않은 Scope는 너무 넓다.

```text
Scope
인증 관련 전체 코드
```

좋은 Scope는 수정 가능한 경계를 구체화한다.

```text
Scope
auth 모듈의 expired token 처리 경로와 해당 regression test
```

핵심은 다음이다.

> Goal은 결과를 제한하고 Scope는 탐색과 변경 범위를 제한한다.

---

## 4. Relevant Files는 시작점이다

Cloud Agent가 Repository를 처음 열었을 때 가장 많은 시간을 쓰는 작업 중 하나는 관련 파일을 찾는 일이다.

`grep`, IDE index, symbol search, import 추적을 통해 결국 필요한 파일 몇 개를 찾을 수 있다.

하지만 사람이 이미 관련 파일을 알고 있다면 매번 처음부터 찾을 이유는 없다.

예:

```text
Relevant Files
- src/main/java/.../AuthService.java
- src/main/java/.../JwtTokenProvider.java
- src/test/java/.../AuthServiceTest.java
```

이 목록은 `이 파일만 읽어라`라는 의미가 아니다.

Agent가 첫 번째로 읽어야 할 Context를 지정하는 것이다.

권장 순서는 다음과 같다.

```text
1. Relevant Files 확인
2. 직접 연결된 dependency 확인
3. 필요한 경우 관련 test/document 조회
4. 범위를 넓혀야 하면 이유를 기록
```

예상하지 못한 dependency가 있을 수 있기 때문에 절대적인 파일 whitelist로 만들지는 않는다.

대신 `initial context boundary`로 사용한다.

---

## 5. Forbidden Changes는 Review 범위를 지킨다

Agent가 문제를 해결하면서 주변 코드를 같이 정리하는 경우가 있다.

예를 들어 expired token 오류를 수정하다가 다음까지 바꿀 수 있다.

```text
Exception hierarchy 정리
JWT class 구조 변경
공통 API response 변경
DB column 추가
OAuth 설정 정리
```

각 변경만 보면 좋은 개선일 수도 있다.

그러나 원래 Task와 관계없는 변경이 섞이면 Review가 어려워진다.

회귀 위험도 커진다.

따라서 Task Contract에는 `하지 말아야 할 것`을 포함한다.

```text
Forbidden Changes
- DB Schema
- OAuth flow
- 공통 Exception API
- public response format
```

파일 경로로 지정할 수도 있다.

```text
Do Not Change
- db/migration/**
- common/exception/**
- security/oauth/**
```

이 정보는 Agent를 통제하기 위한 복잡한 Governance가 아니다.

Task의 Diff를 작게 유지하기 위한 개발 경계다.

---

## 6. Validation은 자연어가 아니라 실행 가능한 명령으로 준다

다음 완료 조건은 모호하다.

```text
기능이 정상 동작하는지 확인한다.
```

누가 정상 여부를 판단하는가.

Agent가 코드를 읽고 `문제없어 보인다`고 판단하면 충분한가.

가능하면 Validation을 실행 명령으로 준다.

```bash
./gradlew test --tests AuthServiceTest.expiredToken
```

필요하면 단계적으로 정의한다.

```text
Fast Validation
./gradlew test --tests AuthServiceTest.expiredToken

Regression Validation
./gradlew test --tests AuthServiceTest
```

이렇게 하면 Agent의 자연어 판단과 실제 실행 결과를 분리할 수 있다.

> 검증할 수 있는 것은 Agent에게 묻지 말고 실행한다.

이 원칙은 10장의 Runner-first 구조로 이어진다.

---

## 7. Expected Result는 관찰 가능한 형태로 쓴다

다음 표현은 Goal처럼 보이지만 판정하기 어렵다.

```text
예외 처리를 더 안정적으로 만든다.
```

무엇이 안정적인 상태인지 확인하기 어렵다.

Expected Result는 가능한 한 관찰할 수 있어야 한다.

```text
Expired token
→ HTTP 401

Valid token
→ 기존 성공 동작 유지
```

또는 테스트 기준으로 쓸 수 있다.

```text
AuthServiceTest.expiredToken PASS
AuthServiceTest.validToken PASS
```

API라면 HTTP Status, JSON Field, Header 같은 값으로 정의할 수 있다.

Migration이라면 다음처럼 쓸 수 있다.

```text
clean DB migration PASS
upgrade from previous schema PASS
```

Expected Result가 명확하면 Developer와 Agent가 같은 완료 조건을 공유한다.

---

## 8. Output은 자연어 설명보다 Evidence를 먼저 정의한다

다음 결과는 충분하지 않다.

```text
수정 완료했습니다.
테스트도 정상입니다.
```

Remote Worker의 결과라면 최소한 다음 정도는 확인할 수 있어야 한다.

```text
Commit
abc123

Changed Files
- AuthService.java
- AuthServiceTest.java

Validation
./gradlew test --tests AuthServiceTest
PASS 24/24
```

UI Task라면 다음이 추가될 수 있다.

```text
Screenshot
Browser Video
E2E Result
```

Docker Build라면 Image Digest가 Evidence가 될 수 있다.

```text
Output / Evidence
- commit SHA
- changed files
- validation result
- artifact reference
- short summary
```

8장에서는 이 Output을 어떻게 작게 만들면서 원본 Artifact를 보존할지 다룬다.

---

## 9. 모든 Context를 처음부터 넣지 않는다

Agent에게 필요한 문서를 모두 Prompt에 붙이는 방식은 쉽게 커진다.

예:

```text
architecture.md
security.md
database.md
api.md
testing.md
deployment.md
```

JWT 오류 하나를 수정하는데 이 문서를 전부 읽힐 필요는 없을 수 있다.

대신 Context를 탐색하는 경로를 만든다.

```text
AGENTS.md
   ↓
Task Type 확인
   ↓
관련 문서
   ↓
관련 Source
   ↓
관련 Test
```

예를 들어 Repository에 다음 안내가 있다고 하자.

```text
AGENTS.md
├─ Backend → docs/backend.md
├─ Authentication → docs/security/auth.md
├─ Database → docs/database.md
├─ Testing → docs/testing.md
└─ Deployment → docs/deployment.md
```

Expired JWT Task라면 다음만 따라가면 된다.

```text
Task Contract
→ AGENTS.md
→ docs/security/auth.md
→ AuthService.java
→ JwtTokenProvider.java
→ AuthServiceTest.java
```

Database와 Deployment 문서는 필요할 때만 읽는다.

이 구조를 `Progressive Context`라고 볼 수 있다.

> Context를 줄이는 것뿐 아니라 Context를 필요할 때 가져오게 만든다.

---

## 10. Context 확대도 단계가 있어야 한다

첫 번째 수정에 실패했다고 바로 Repository 전체 탐색으로 넘어갈 필요는 없다.

Context 확장 단계를 미리 정할 수 있다.

```text
Level 1
Task Contract + Relevant Files
      ↓
Level 2
직접 dependency + related test
      ↓
Level 3
관련 module documentation
      ↓
Level 4
specific failure artifact / log
      ↓
Level 5
넓은 module/repository 탐색
```

이 흐름은 Token 절약만을 위한 것이 아니다.

문제와 관련없는 파일을 수정하는 범위 확장도 줄인다.

Context가 넓어질수록 Agent가 선택할 수 있는 변경 경로도 많아진다.

따라서 작은 문제는 작은 Context로 시작하는 편이 Review에도 유리하다.

---

## 11. Base Commit과 Branch를 명시한다

Cloud Worker가 Remote Repository에서 작업하려면 시작점을 고정하는 것이 좋다.

예:

```text
Repository
campus-platform

Base SHA
4f29abc

Task Branch
agent/auth-expired-token-142
```

Task 상태를 다음처럼 연결할 수 있다.

```text
Task ID
→ Session ID
→ Base SHA
→ Branch
→ Result Commit
→ Test Result
→ PR
```

이 정보가 있으면 Review할 때도 어떤 코드에서 작업을 시작했는지 확인하기 쉽다.

Branch와 Worktree의 상세 격리는 11장에서 다룬다.

---

## 12. Environment는 이름으로 선택하게 만든다

매 Task마다 JDK, Node, Docker 설치 방법을 설명할 필요는 없다.

대신 준비된 Environment를 선택할 수 있다.

```text
Environment
backend-test
```

이 Environment가 다음을 포함한다고 하자.

```text
Java 21
Gradle
Docker
PostgreSQL Testcontainers
```

Frontend Task는 다른 환경을 사용한다.

```text
Environment
frontend-e2e
```

```text
Node
Playwright
Chrome
```

Task Contract에는 `어떤 환경을 사용할지`만 적고, 환경 구성 자체는 9장의 Prepared Environment에서 관리한다.

---

## 13. Budget은 선택 필드다

Cloud Agent가 실패와 수정을 반복하면 작업이 예상보다 커질 수 있다.

이를 무한히 계속하지 않도록 Budget을 둘 수 있다.

```text
Budget
max_retry: 2
max_turns: 8
```

다른 후보도 있다.

```text
max wall-clock time
max token/cost
max changed files
```

제품마다 Token과 과금 방식이 다르기 때문에 이 책에서는 절대 수치를 표준값으로 정하지 않는다.

중요한 것은 `성공할 때까지 계속해` 같은 무제한 Task를 피하는 것이다.

같은 실패가 반복되면 8장의 Failure Fingerprint로 중단 여부를 판단한다.

---

## 14. Task Contract가 너무 크면 Task 자체를 의심한다

Task Contract가 다음처럼 커졌다고 하자.

```text
Relevant Files: 80개
Related Modules: 12개
Validation Commands: 15개
Forbidden Changes: 수십 개
```

이 문제는 Prompt 형식을 더 정교하게 만든다고 해결되지 않을 수 있다.

Task 자체가 너무 큰 것이다.

다음 순서로 다시 본다.

```text
Task Contract가 비정상적으로 큼
        ↓
Task Scope 재검토
        ↓
분리 가능한가?
  ├─ YES → 작은 Task로 분해
  └─ NO  → Local / Hybrid 재검토
```

작은 Task가 항상 정답은 아니다.

하지만 Cloud Worker가 독립적으로 끝낼 수 있는 단위인지 확인해야 한다.

---

## 15. campus-platform Task Contract 예제

`campus-platform`의 expired token Bug를 하나의 Cloud Task로 만들면 다음처럼 정리할 수 있다.

```text
Task ID
AUTH-142

Task
Expired token 처리 수정

Goal
Attendance API에 expired JWT 사용 시 HTTP 401 반환

Repository
campus-platform

Base SHA
4f29abc

Task Branch
agent/auth-expired-token-142

Scope
auth 모듈의 token expiration handling과 관련 test

Relevant Files
- auth/.../AuthService.java
- auth/.../JwtTokenProvider.java
- auth/.../AuthServiceTest.java

Forbidden Changes
- DB Schema
- OAuth flow
- common exception response format

Environment
backend-test

Validation
./gradlew :auth:test --tests AuthServiceTest.expiredToken
./gradlew :auth:test

Expected Result
- expired token → 401
- valid token 기존 동작 유지

Output / Evidence
- result commit SHA
- changed files
- test result
- short summary

Budget
- max retry: 2
```

Agent는 먼저 Relevant Files에서 시작한다.

문제를 해결하지 못하면 direct dependency와 관련 문서를 단계적으로 추가한다.

전체 Repository를 처음부터 읽는 것은 마지막 선택에 가깝다.

---

## 16. Task Contract는 팀의 반복 작업 자산이 된다

Task Contract를 매번 자유 형식 Prompt로 작성할 필요는 없다.

자주 반복되는 작업은 Template으로 만들 수 있다.

예:

```text
bug-fix-task.md
refactor-task.md
ci-failure-task.md
dependency-update-task.md
migration-task.md
```

Bug Fix Template:

```text
Task
Failure
Goal
Scope
Relevant Files
Validation
Forbidden Changes
Output
```

Dependency Update Template:

```text
Dependency
Old Version
New Version
Validation
Expected Compatibility
Output
```

Template의 목적은 문서를 늘리는 것이 아니다.

Developer가 Cloud Worker에게 전달할 최소 정보를 빠뜨리지 않게 하는 것이다.

---

## 17. 작은 Input과 작은 Output을 연결한다

7장의 Task Contract는 Cloud Agent의 **입력**을 줄이는 방법이다.

하지만 입력만 작게 만들어서는 충분하지 않다.

Cloud Runner가 100MB 로그를 만든 뒤 전부 Agent에게 전달한다면 Context는 다시 커진다.

따라서 다음 구조로 이어진다.

```text
Task Contract
→ Small Input / Context
        ↓
Cloud Runner / Agent
        ↓
Result Gateway
→ Small Output / Evidence
```

다음 장에서는 Build/Test/E2E의 대형 결과를 저장하고, Agent에게는 필요한 실패와 Evidence만 전달하는 방법을 다룬다.

---

## 이 장에서 기억할 것

Cloud Agent Task를 만들 때 다음 질문을 확인한다.

```text
무엇을 해야 하는가?
어떤 결과가 필요한가?
어디까지 수정 가능한가?
어디서 읽기 시작해야 하는가?
무엇을 변경하면 안 되는가?
어떤 명령으로 검증하는가?
무엇을 Evidence로 반환하는가?
어느 Git 상태에서 시작하는가?
어떤 Environment를 사용하는가?
언제 Retry를 멈추는가?
```

모든 항목을 항상 채울 필요는 없다.

Task에 필요한 최소 경계를 제공하면 된다.

핵심은 다시 다음 문장으로 돌아온다.

> Task Contract의 목적은 Prompt를 길게 만드는 것이 아니라 Agent가 탐색해야 하는 범위를 줄이는 것이다.
