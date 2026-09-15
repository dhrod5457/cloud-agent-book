# 7장 설계 - Cloud Agent Task Contract: 작은 Task와 작은 Context

## 장의 목표

Cloud Agent에게 자유 형식의 긴 지시를 전달하는 대신, 작업 범위와 검증 방법을 짧고 명확하게 정의하는 `Cloud Agent Task Contract`를 설계한다.

5장에서 `이 Task를 Cloud로 보낼 것인가`를 판단했고, 6장에서 Cloud에 보내기 좋은 실제 작업 유형을 구분했다.

7장에서는 실제로 Cloud Agent에게 Task를 넘길 때 무엇을 전달해야 하는지 정의한다.

핵심 질문:

> Cloud Agent가 Repository 전체를 다시 탐색하지 않고도 필요한 작업을 시작하고, 범위를 벗어나지 않고 수정하며, 완료 여부를 검증하려면 무엇을 알려줘야 하는가?

---

## 핵심 주장

> Task Contract의 목적은 Prompt를 길게 만드는 것이 아니라 Agent가 탐색해야 하는 범위를 줄이는 것이다.

Cloud Agent에게 필요한 정보는 가능한 한 다음 범위로 제한한다.

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

- Base Commit / Branch
- Dependency
- Environment
- Retry / Token Budget
- Related Document

모든 정보를 처음부터 Context에 넣는 것이 아니라, Agent가 필요한 경우 다음 단계의 문서와 Artifact를 조회할 수 있도록 한다.

> Context를 줄이는 것뿐 아니라 Context를 필요할 때 가져오게 만든다.

---

## 독자가 얻는 것

- Cloud Agent용 Task를 작은 실행 단위로 정의할 수 있다.
- Goal과 Scope를 분리할 수 있다.
- Agent가 먼저 읽을 Relevant Files를 제한할 수 있다.
- 변경 금지 영역을 명시해 불필요한 수정 범위를 줄일 수 있다.
- Validation 명령과 Expected Result를 완료 조건으로 사용할 수 있다.
- 자연어 완료 선언 대신 Commit/Test/Artifact 같은 Evidence를 요구할 수 있다.
- AGENTS.md에서 작업별 상세 문서로 이어지는 Progressive Context를 구성할 수 있다.
- Repository 전체 재탐색으로 발생하는 Token 사용을 줄일 수 있다.
- 너무 작은 Task와 너무 큰 Task를 Task Contract 단계에서 다시 확인할 수 있다.

---

# 절 구성

## 7.1 Cloud Task는 Prompt가 아니라 작업 패키지다

좋지 않은 예:

```text
로그인 쪽에 문제가 있는 것 같은데 Repository 전체를 확인해서
관련된 부분을 분석하고 적절하게 수정한 다음 테스트도 해줘.
```

이 지시는 Agent에게 다음을 모두 추론하게 만든다.

- 실제 문제
- 변경 대상
- 관련 파일
- 변경 금지 영역
- 테스트 범위
- 완료 조건

권장 형태:

```text
Task
AuthService expired token 처리 수정

Goal
Expired JWT 요청이 HTTP 401을 반환하게 한다.

Scope
인증 만료 처리 로직과 관련 테스트만 수정한다.

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
- OAuth 전체 구조
- 공통 Exception format

Output
- commit
- changed files
- test result
- short summary
```

Task Contract는 Agent의 사고 과정을 대신 작성하는 문서가 아니다.

Agent가 `어디까지 탐색해야 하는가`를 정해주는 실행 경계다.

---

## 7.2 Goal과 Scope를 분리한다

Goal은 원하는 결과를 설명한다.

```text
Goal
Expired JWT 요청이 HTTP 401을 반환한다.
```

Scope는 변경할 작업 영역을 설명한다.

```text
Scope
AuthService의 token validation 처리와 관련 test만 수정한다.
```

두 항목을 합치면 Agent가 결과는 이해해도 변경 범위를 넓게 해석할 수 있다.

좋지 않은 예:

```text
인증 문제를 해결한다.
```

권장 예:

```text
Goal
Expired JWT → 401

Scope
AuthService token expiration handling
```

핵심:

> Goal은 결과를 제한하고 Scope는 탐색과 변경 범위를 제한한다.

---

## 7.3 Relevant Files는 시작점이지 절대 목록이 아니다

Relevant Files를 제공하면 Agent가 Repository 전체 검색부터 시작하지 않아도 된다.

예:

```text
Relevant Files
- src/main/java/.../AuthService.java
- src/main/java/.../JwtTokenProvider.java
- src/test/java/.../AuthServiceTest.java
```

하지만 예상하지 못한 dependency가 있을 수 있으므로 `이 파일 외에는 절대 읽지 마라`처럼 사용하지 않는다.

권장 정책:

```text
1. Relevant Files부터 읽는다.
2. 필요한 경우 직접 연결된 dependency를 조회한다.
3. 범위를 확대해야 하면 이유를 기록한다.
4. 변경은 Scope와 Forbidden Changes 안에서만 수행한다.
```

즉 Relevant Files의 역할은 `initial context boundary`다.

목표는 정확한 세 파일만 강제하는 것이 아니라 불필요한 Repository 탐색을 줄이는 것이다.

---

## 7.4 Forbidden Changes로 변경 경계를 명확히 한다

Cloud Agent는 문제를 해결하면서 예상보다 넓은 개선을 시도할 수 있다.

예:

```text
expired token 처리 수정
→ Exception hierarchy 정리
→ JWT 구조 리팩터링
→ 공통 API 응답 형식 변경
```

Task 자체는 성공하더라도 Review 범위와 회귀 위험이 커진다.

따라서 금지 영역을 명시한다.

```text
Forbidden Changes
- DB Schema
- OAuth flow
- 공통 Exception API
- public API response format
```

또는 파일 단위로 제한할 수 있다.

```text
Do Not Change
- db/migration/**
- common/exception/**
- security/oauth/**
```

핵심:

> Cloud Agent에게 무엇을 할지뿐 아니라 무엇을 하지 않을지도 알려준다.

---

## 7.5 Validation은 설명이 아니라 실행 명령으로 준다

좋지 않은 완료 조건:

```text
기능이 정상 동작하는지 확인한다.
```

권장 방식:

```text
Validation
./gradlew test --tests AuthServiceTest.expiredToken
```

필요하면 단계적으로 정의한다.

```text
Fast Validation
./gradlew test --tests AuthServiceTest.expiredToken

Regression Validation
./gradlew test --tests AuthServiceTest
```

Cloud Agent는 수정 후 자연어로 `문제가 해결됐다`고 판단하는 것이 아니라 정해진 명령으로 확인한다.

10장에서 Runner-first 구조를 상세히 다루지만, Task Contract에는 최소한 어떤 명령이 성공해야 하는지 명시한다.

핵심:

> 검증할 수 있는 것은 Agent에게 묻지 말고 실행한다.

---

## 7.6 Expected Result를 기계적으로 확인 가능한 형태로 쓴다

Expected Result는 가능한 한 관찰 가능한 결과로 작성한다.

좋지 않은 예:

```text
예외 처리를 더 안정적으로 만든다.
```

권장 예:

```text
Expired token
→ HTTP 401

Valid token
→ 기존 성공 경로 유지
```

테스트 기준으로 표현할 수도 있다.

```text
AuthServiceTest.expiredToken PASS
AuthServiceTest.validToken PASS
```

Expected Result가 명확하면 Agent의 수정 범위를 좁힐 뿐 아니라 Reviewer도 결과를 빠르게 확인할 수 있다.

---

## 7.7 Output은 설명이 아니라 Evidence 중심으로 정의한다

Cloud Task가 끝났을 때 다음 응답만 받는 것은 부족하다.

```text
수정 완료했습니다.
```

권장 Output:

```text
Output
- commit SHA
- changed files
- validation result
- artifact reference if generated
- short summary
```

예:

```text
Result
commit: abc123
changed:
- AuthService.java
- AuthServiceTest.java

validation:
./gradlew test --tests AuthServiceTest
PASS 24/24
```

UI Task라면 다음이 추가될 수 있다.

- screenshot
- browser video
- E2E result

8장에서 Evidence와 Result Gateway를 상세히 다룬다.

---

## 7.8 Progressive Context: 처음부터 모든 문서를 넣지 않는다

나쁜 구조:

```text
architecture.md
security.md
database.md
api.md
testing.md
deployment.md

→ 전부 Context에 포함
```

권장 구조:

```text
AGENTS.md
   ↓
Task Type 확인
   ↓
관련 문서만 조회
   ↓
관련 소스
   ↓
관련 테스트
```

예:

```text
AGENTS.md
├─ Backend → docs/backend.md
├─ Authentication → docs/security/auth.md
├─ Database → docs/database.md
├─ Testing → docs/testing.md
└─ Deployment → docs/deployment.md
```

JWT 오류 Task라면:

```text
Task Contract
→ AGENTS.md
→ docs/security/auth.md
→ AuthService.java
→ JwtTokenProvider.java
→ AuthServiceTest.java
```

DB나 Deployment 문서는 필요하지 않으면 읽지 않는다.

핵심:

> 작은 Context는 정보를 없애는 것이 아니라 필요한 정보에 도달하는 경로를 짧게 만드는 것이다.

---

## 7.9 Context 확대는 단계적으로 한다

Cloud Agent가 문제를 해결하지 못했다고 바로 Repository 전체를 읽게 하지 않는다.

Context 확대 순서를 정의할 수 있다.

```text
Level 1
Task Contract + Relevant Files
      ↓
Level 2
직접 dependency + 관련 test
      ↓
Level 3
관련 module documentation
      ↓
Level 4
Failure artifact / specific log
      ↓
Level 5
넓은 module/repository 탐색
```

이 방식은 Token 절약뿐 아니라 Agent가 문제와 관련 없는 영역을 수정할 가능성도 줄인다.

8장의 Result Gateway 역시 같은 원칙을 사용한다.

```text
Summary
→ Failure Detail
→ Specific Log
→ Raw Artifact
```

---

## 7.10 Base Commit과 Branch를 입력에 포함한다

Cloud Agent는 Remote Repository 기준으로 작업하므로 시작점을 명확하게 한다.

예:

```text
Repository
dhrod5457/campus-platform

Base
main@4f29abc

Task Branch
agent/auth-expired-token-142
```

이 정보가 없으면 작업 중 branch가 이동하거나 다른 변경과 섞였을 때 어떤 코드 기준으로 작업했는지 추적하기 어렵다.

Task 상태를 다음과 같이 연결할 수 있다.

```text
Task ID
→ Session ID
→ Branch
→ Base SHA
→ Result Commit
→ Test Result
→ PR
```

Branch와 Worktree의 상세 전략은 11장에서 다룬다.

---

## 7.11 Environment 정보는 필요한 만큼만 준다

모든 Task에 전체 인프라 설명을 포함할 필요는 없다.

예:

```text
Environment
backend-test

Requires
- Java 21
- Gradle
- PostgreSQL Testcontainers
```

UI Task라면:

```text
Environment
frontend-e2e

Requires
- Node
- Playwright
- Chrome
```

Prepared Cloud Environment의 상세 설계는 9장에서 다룬다.

Task Contract에는 `어떤 실행환경을 사용해야 하는가`만 명확하면 된다.

---

## 7.12 Budget은 선택 필드로 둔다

Cloud Agent가 판단과 수정을 반복하는 작업이라면 Budget을 지정할 수 있다.

예:

```text
Budget
max_retry: 2
max_turns: 8
```

제품별 Token accounting이 다를 수 있으므로 본문에서는 절대 수치를 표준값으로 제시하지 않는다.

Budget 후보:

- max retry
- max turns
- max wall-clock time
- max token/cost
- max changed files

핵심 목적은 무한 Retry를 막는 것이다.

동일 실패가 반복되면 Context를 무작정 늘리지 않고 8장의 Failure Fingerprint와 14장의 Event/Retry 흐름으로 연결한다.

---

## 7.13 Task Contract가 너무 크면 Task가 큰지 의심한다

Task Contract가 다음처럼 길어질 수 있다.

```text
Relevant Files: 80개
Related Modules: 12개
Validation Commands: 15개
Forbidden Changes: 수십 개
```

이 경우 문서를 더 잘 쓰는 것보다 Task 자체가 너무 큰지 먼저 확인한다.

판단:

```text
Task Contract가 비정상적으로 커짐
        ↓
Task Scope 재검토
        ↓
분리 가능한가?
  ├─ YES → 작은 Task로 분해
  └─ NO  → Local/Hybrid 재검토
```

5장의 Task Routing과 연결되는 지점이다.

핵심:

> Context를 줄이기 어려운 Task는 Cloud에 적합하지 않을 수도 있다.

---

## 7.14 campus-platform 예제 - AuthService expired token

실전 예제 Task Contract:

```text
Task ID
AUTH-142

Task
AuthService expired token 처리 수정

Goal
Expired JWT 요청이 HTTP 401을 반환한다.

Base
main@4f29abc

Scope
인증 token expiration 처리와 관련 test만 수정한다.

Relevant Files
- src/main/java/.../AuthService.java
- src/main/java/.../JwtTokenProvider.java
- src/test/java/.../AuthServiceTest.java

Related Doc
docs/security/auth.md

Failure
AuthServiceTest.expiredToken
expected: 401
actual: 200

Validation
./gradlew test --tests AuthServiceTest.expiredToken
./gradlew test --tests AuthServiceTest

Expected Result
- expired token → 401
- valid token 기존 동작 유지

Forbidden Changes
- DB Schema
- OAuth 전체 구조
- common exception response format

Environment
backend-test

Output
- result commit SHA
- changed files
- validation result
- short summary

Budget
- max_retry: 2
```

Agent의 첫 Context는 위 Contract와 Relevant Files로 제한한다.

문제가 해결되지 않을 때만 직접 dependency, 관련 문서, 실패 로그 순서로 Context를 확장한다.

---

# 좋은 사례와 나쁜 사례

## 사례 A - Repository 전체 탐색

좋지 않은 방식:

```text
Repository 전체를 분석해서 인증 오류를 해결해.
```

권장 방식:

```text
Task Contract
+ 관련 파일 3개
+ 단일 실패 테스트
+ 검증 명령
```

## 사례 B - 완료 조건

좋지 않은 방식:

```text
적절하게 고쳐줘.
```

권장 방식:

```text
expired token → 401
AuthServiceTest PASS
```

## 사례 C - 변경 범위

좋지 않은 방식:

```text
필요한 부분은 알아서 리팩터링해도 됨
```

권장 방식:

```text
Scope
Auth expiration handling

Forbidden
DB / OAuth architecture / common exception API
```

## 사례 D - Context

좋지 않은 방식:

```text
모든 architecture/security/database/testing 문서를 시작 Context에 포함
```

권장 방식:

```text
AGENTS.md
→ auth 문서
→ 관련 파일
→ 필요 시 추가 조회
```

---

# 필요한 구조/그림

## 그림 1 - Task Contract 구조

```text
Task
├─ Goal
├─ Scope
├─ Relevant Files
├─ Forbidden Changes
├─ Validation
├─ Expected Result
└─ Output / Evidence
```

## 그림 2 - Progressive Context

```text
Task Contract
      ↓
Relevant Files
      ↓
Direct Dependency
      ↓
Related Documentation
      ↓
Failure Detail
      ↓
Wider Repository Context
```

## 그림 3 - Git 기준점

```text
Base SHA
   ↓
Task Branch
   ↓
Cloud Agent
   ↓
Result Commit
   ↓
Validation
   ↓
PR
```

---

# 필요한 코드/설정 예제

본문 작성 시 다음 예제를 사용할 수 있다.

- Markdown 기반 Task Contract
- YAML 형태의 선택적 Task Contract 예
- `AuthServiceTest.expiredToken` 실행 명령
- AGENTS.md의 문서 routing 예
- Task ID / Branch / Commit / Result 연결 예

YAML schema나 orchestration protocol 자체를 표준으로 제안하지 않는다.

목적은 형식보다 Task/Context 범위를 줄이는 데 있다.

---

# 필요한 공식 자료 조사

본문 작성 시 필요한 경우 다음을 확인한다.

- Claude Code, Codex, GitHub Copilot cloud agent의 task/repository context 전달 방식
- 제품별 repository/branch 기준 작업 방식
- 제품별 instruction file 지원 방식

제품별 Prompt format을 비교하는 장으로 만들지 않는다.

현재 제품 기능은 일반 원칙의 사례로만 사용한다.

---

# 앞 장과 뒤 장의 연결

## 앞 장 - 6장

6장에서 Cloud에 보내기 좋은 작업을 Build/Test/Refactoring/Bug Fix 등으로 분류했다.

7장은 그중 Agent 판단이 필요한 작업을 실제로 어떻게 작은 Task로 전달하는지 정의한다.

## 다음 장 - 8장

8장에서는 Task를 실행한 뒤 만들어지는 대량 Tool Output을 어떻게 줄이고, 원본 Artifact를 보존하면서 Agent에는 필요한 Evidence만 전달할지 설명한다.

연결:

```text
7장
작은 Input / Context
      ↓
Cloud Agent / Runner
      ↓
8장
작은 Output / Evidence
```

즉 7장과 8장은 Cloud Agent의 입력과 출력 양쪽에서 Token 사용량을 줄이는 구조다.

---

# 본문에서 의도적으로 다루지 않을 내용

- Prompt Engineering 일반론
- Chain-of-Thought 작성법
- Agent Memory Architecture
- Agent Platform Task Queue 상세 구현
- 특정 제품의 Task UI 사용법
- Workflow engine 설계
- YAML/JSON schema 표준화
- 전체 Repository instruction architecture 일반론

필요한 내용은 Cloud Agent의 작은 Task와 작은 Context라는 목적에 직접 연결되는 범위에서만 다룬다.

---

# 이 장의 체크포인트

독자는 7장을 읽은 뒤 다음 질문에 답할 수 있어야 한다.

- Task의 Goal과 Scope가 분리되어 있는가?
- Agent가 먼저 읽을 Relevant Files를 제한했는가?
- 변경하면 안 되는 영역을 명시했는가?
- 완료 여부를 확인할 Validation 명령이 있는가?
- Expected Result가 관찰 가능한 형태인가?
- Output에 Commit/Test/Artifact Evidence를 요구하는가?
- 필요한 문서만 Progressive Context로 조회할 수 있는가?
- Task Contract가 너무 커서 Task 분리가 필요한 상태는 아닌가?
- Cloud Agent가 Repository 전체를 다시 탐색하지 않아도 되는가?
