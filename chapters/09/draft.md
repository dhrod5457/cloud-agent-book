# 9장. Prepared Cloud Environment, Cache, Snapshot

클라우드 에이전트가 작업을 받아도 바로 코드를 수정할 수 있는 것은 아니다. 실제 작업에 들어가기 전에 실행환경을 준비해야 한다.

```text
Worker 생성
→ Repository Checkout
→ Runtime 확인
→ Dependency 준비
→ Docker / Browser 준비
→ First Command
→ Task 시작
```

이 구간이 길면 모델이 빨라도 전체 작업은 느리다.

따라서 9장의 대상은 에이전트의 추론 속도가 아니라 **환경 준비 비용**이다.

핵심 원칙은 다음과 같다.

> 에이전트에게 개발환경을 설치하게 하지 말고 바로 작업 가능한 환경을 제공한다.

> 클라우드 환경도 소스 코드처럼 버전 관리하고 재현 가능하게 만든다.

> 캐시는 재사용하되 소스 코드와 실행 상태는 새로 준비한다.

---

## 1. Cold Start를 구성 요소로 나눈다

클라우드 작업의 시작시간을 `Agent 응답 시작시간` 하나로 보면 병목을 찾기 어렵다.

다음처럼 분해해서 본다.

```text
Provisioning Time
Checkout Time
Dependency Restore Time
Environment Warm-up Time
First Command Time
```

설명용 예로 환경 준비가 8분이고 실제 수정이 3분이라면 모델만 바꿔서는 전체 시간이 크게 줄지 않는다.

먼저 어느 구간이 반복되는지 측정해야 한다.

---

## 2. Prepared Environment를 만든다

매 작업마다 JDK나 브라우저를 설치하지 않는다.

기본 구조:

```text
Base Image
   ↓
Runtime 설치
   ↓
Development Tools 설치
   ↓
Dependency 준비
   ↓
Cache Warming
   ↓
Prepared Environment
```

작업 실행 시에는 준비된 환경에 새로 준비한 소스 코드를 올린다.

```text
Prepared Environment
        +
Fresh Repository / Branch
        ↓
Task
```

포함 후보:

```text
Java / Node
Gradle / Maven
Docker CLI
Playwright Browser
Static Analysis Tool
DB Client
기본 OS Package
```

제품별 스냅샷 기능이 없어도 Dockerfile, Dev Container, Bootstrap Script 같은 방식으로 같은 원칙을 적용할 수 있다.

---

## 3. Cloud Environment as Code

환경 구성을 사람의 기억에 맡기지 않는다.

예:

```text
cloud-env/
├─ Dockerfile
├─ versions.env
├─ install-tools.sh
└─ warm-cache.sh
```

환경 버전도 코드처럼 관리한다.

```text
cloud-env: backend-test-v12
java: 21
node: 22
playwright: pinned
```

핵심은 특정 제품의 환경 설정 UI가 아니다.

`어떤 Runtime과 Tool을 사용했는지 다시 만들 수 있는가`가 중요하다.

---

## 4. Task별 Environment를 분리한다

모든 작업자에 모든 도구를 넣을 필요는 없다.

`campus-platform`에서는 다음 정도로 나눌 수 있다.

### backend-test

```text
Java 21
Gradle
Docker
Testcontainers
DB Client
```

### frontend-e2e

```text
Node
Playwright
Chrome
```

### migration-test

```text
Java 21
Migration Tool
PostgreSQL
DB Client
```

프런트엔드와 백엔드 등 여러 기술 영역에 걸친 문제처럼 필요한 경우에만 더 큰 `fullstack` 환경을 사용한다.

실행 위치 결정은 단순하다.

```text
Backend Unit / Integration
→ backend-test

Frontend E2E
→ frontend-e2e

Migration Validation
→ migration-test
```

환경에 도구를 많이 넣는다고 항상 좋은 것은 아니다. 사용하지 않는 도구까지 포함하면 실행환경 이미지가 커지고 준비 비용도 늘어난다.

---

## 5. 재사용할 것과 초기화할 것을 분리한다

미리 준비한 실행환경에서 가장 중요한 경계다.

재사용하기 좋은 상태:

```text
Gradle Dependency Cache
Maven Repository
npm Cache
Docker Layer
Playwright Browser
Compiler Cache
```

매 작업마다 새로 만들어야 할 상태:

```text
Source Checkout
Branch
Task Input
DB State
Temporary File
Mutable Fixture
Test Output
Process / Session State
```

구조:

```text
Reusable
- Runtime
- Tools
- Dependency Cache
        +
Fresh
- Source
- Branch
- Task
- Runtime Data
        ↓
Execution
```

목표는 `빠른 시작`과 `깨끗한 실행`을 동시에 얻는 것이다.

---

## 6. Cache는 Invalidation까지 설계한다

한번 준비한 내용을 다시 쓰기 위해 저장하는 캐시(Cache)는 오래 보관하는 것보다 언제 버릴지 정하는 것이 중요하다.

캐시를 구분하는 키 후보:

```text
OS / Architecture
JDK Version
Gradle / Maven Lock State
Package Manifest Hash
Dockerfile Hash
Tool Version
```

예:

```text
gradle-cache
key = os + jdk + dependency-lock-hash
```

잘못된 캐시는 다음 문제를 만든다.

```text
Stale Dependency
이전 Generated Code 잔존
다른 Branch 결과 혼입
Test Pollution
```

소스 코드 상태에 강하게 의존하는 빌드 출력이나 테스트 결과를 무조건 재사용하지 않는다.

---

## 7. Snapshot은 시작 상태를 미리 준비한다

특정 시점의 상태를 저장한 스냅샷(Snapshot)은 작업 시작 전에 실행환경과 도구가 준비된 상태를 보관하는 방법으로 볼 수 있다.

```text
Base Image
→ Tool 설치
→ Dependency Restore
→ Browser 설치
→ Snapshot
```

작업 시작:

```text
Snapshot
→ Fresh Source Checkout
→ Task Branch
→ Execute
```

책의 기본값은 다음과 같다.

```text
Environment / Dependency
→ 재사용

Source / Branch / Runtime State
→ Fresh
```

소스 코드까지 스냅샷에 포함한다면 최신 커밋과의 차이를 반영하는 데 드는 비용도 고려해야 한다. 오래된 소스 코드를 그대로 사용할 위험도 함께 살펴봐야 한다.

---

## 8. Warm Worker는 선택지다

짧고 반복되는 작업은 READY 상태 작업자를 재사용해 시작시간을 줄일 수 있다.

```text
READY
→ Task
→ Reset
→ READY
```

후보:

```text
Lint
Compile
Small Unit Test
PR Verification
```

반대로 장시간 작업이나 격리가 중요한 작업은 작업마다 만들고 폐기하는 임시 작업자가 더 단순할 수 있다.

```text
Task
→ New Worker
→ Execute
→ Artifact
→ Destroy
```

미리 대기시킨 작업자 자체를 기본값으로 두지 않는다. 시작 비용과 격리 요구를 보고 선택한다.

---

## 9. 반복되는 환경 실패는 Prompt 문제가 아니다

다음 상황을 보자.

```text
Agent
→ JDK 없음
→ 설치 방법 추론
→ 설치 실패
→ 다시 추론
```

이 문제를 프롬프트에 설치 방법을 더 길게 적어서 해결하지 않는다.

권장 대응:

```text
Prepared Environment 수정
→ Java 21 포함
→ Environment Version 갱신
```

Playwright 브라우저가 반복해서 없다면 `frontend-e2e` 실행환경에 포함한다.

> 반복되는 환경 실패는 에이전트의 추론 문제가 아니라 실행환경 문제로 취급한다.

이 원칙은 18장의 Harness Engineering과도 연결된다.

---

## 10. Secret은 Image에 넣지 않는다

미리 준비한 실행환경과 실행 시점의 인증 정보를 분리한다.

```text
Prepared Environment
- Runtime
- Tools
- Cache

Execution-time Injection
- Secret
- Token
- Temporary Credential
```

비밀번호나 키 같은 비밀 정보(Secret)는 실행환경 이미지(Image)에 넣지 않는다.

클라우드에서 어떤 인증 정보를 제공할 수 있는지는 조직 정책과 5장의 실행 위치 결정 기준을 따른다.

---

## 11. Cold Start를 측정한다

다음 값을 기록할 수 있다.

```text
worker_provision_ms
checkout_ms
dependency_restore_ms
image_pull_ms
browser_ready_ms
first_command_ms
```

설명용 예:

```text
Before
Worker ready: 7m 40s

After
Worker ready: 55s
```

이 숫자는 제품 기준이 아니라 측정 방법을 설명하기 위한 예다.

실제 프로젝트에서는 반복 작업의 준비시간을 측정해 실행환경 개선 효과를 확인한다.

---

## 12. campus-platform에서는 Environment 이름으로 Task를 연결한다

예:

```text
backend-test
→ Java 21 / Gradle / Docker / Testcontainers

frontend-e2e
→ Node / Playwright / Chrome

migration-test
→ Java 21 / Migration Tool / PostgreSQL
```

작업 명세에는 설치 절차 대신 실행환경 이름만 넣는다.

```text
Task: AUTH-142
Environment: backend-test
```

그리고 실행 상태는 새로 준비해 시작한다.

```text
Fresh Git Checkout
Task Branch
Disposable Test DB
New Test Output Directory
```

Tibero, HSM, Internal Jenkins처럼 클라우드에서 제공하지 않는 내부 자원을 미리 준비한 실행환경으로 억지로 복제하지 않는다. 그런 작업은 Hybrid로 남긴다.

---

## 13. 작은 입력과 작은 출력 다음에는 빠른 시작이 필요하다

7~9장의 흐름은 다음과 같다.

```text
7장
Task Contract
→ Small Input / Context
        ↓
8장
Result Gateway
→ Small Output / Evidence
        ↓
9장
Prepared Environment
→ Small Startup Overhead
```

이제 작업의 입력, 결과, 시작 환경이 정리됐다.

다음 장에서는 이 미리 준비한 실행환경에서 어떤 작업을 LLM이 아닌 실행기에게 먼저 맡길지 다룬다.

```text
Prepared Environment
        ↓
Cloud Runner
        ↓
PASS → 종료
FAIL + 판단 필요 → Cloud Agent
```

9장이 시작 비용을 줄이는 장이라면 10장은 **LLM이 필요한 실행 구간을 줄이는 장**이다.
