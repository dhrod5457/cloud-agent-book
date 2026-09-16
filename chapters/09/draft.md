# 9장. Prepared Cloud Environment, Cache, Snapshot

Cloud Agent가 Task를 받았다고 해서 바로 코드 수정이 시작되는 것은 아니다.

실제 작업 전에는 환경을 준비해야 한다.

```text
Worker 생성
→ Repository checkout
→ Runtime 확인
→ Dependency 준비
→ Docker / Browser 준비
→ Build bootstrap
→ 실제 Task 시작
```

이 과정이 길면 Agent가 아무리 빠르게 판단해도 전체 작업은 느려진다.

예를 들어 코드 수정에는 3분이 걸리는데 JDK, Gradle Dependency, Docker Image, Browser를 준비하는 데 8분이 걸린다면 문제는 모델 속도가 아니다.

환경 준비가 병목이다.

그래서 이 장에서는 Cloud Agent의 실행환경을 매번 즉석에서 만드는 공간이 아니라 **미리 준비하고 재사용하는 개발 자산**으로 본다.

핵심 원칙은 세 가지다.

> Agent에게 개발환경을 설치하게 하지 말고, 바로 작업 가능한 환경을 제공한다.

> Cloud 환경도 Source Code처럼 버전 관리하고 재현 가능하게 만든다.

> 재사용 가능한 Cache와 매 실행 초기화해야 하는 Fresh State를 분리한다.

---

## 1. Cloud Agent의 시작시간은 모델 응답시간만이 아니다

Cloud Coding Agent의 체감 속도를 평가할 때 첫 응답까지 걸린 시간만 보는 경우가 있다.

하지만 실제 개발 Task의 시작시간은 더 길 수 있다.

```text
Task 생성
→ Worker Provisioning
→ Repository Checkout
→ Runtime 준비
→ Dependency Restore
→ Container / Browser 준비
→ 첫 Build/Test 가능
```

따라서 최소한 다음 시간을 구분하는 편이 낫다.

```text
Provisioning Time
Checkout Time
Dependency Restore Time
Environment Warm-up Time
Agent Execution Time
```

예를 들어 다음과 같은 측정 결과가 있다고 하자.

```text
Worker Provisioning       40s
Repository Checkout       20s
Gradle Dependency Restore 180s
Docker Image Pull         120s
Agent Code Fix            90s
```

실제 수정은 90초인데 준비에 6분이 넘게 걸린다.

이 경우 Agent Prompt를 더 정교하게 만드는 것보다 환경 준비 구조를 바꾸는 편이 효과가 크다.

Cloud Agent를 빠르게 만드는 일은 모델만 빠르게 만드는 일이 아니다.

**Task가 실제로 실행 가능해질 때까지의 시간을 줄이는 일**도 포함한다.

---

## 2. Prepared Cloud Environment

Prepared Cloud Environment는 Agent가 Task를 받을 때 필요한 Runtime과 도구가 이미 준비된 상태를 뜻한다.

기본 흐름은 다음과 같다.

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
Snapshot / Image 생성
```

실제 Task에서는 이 준비된 환경을 사용한다.

```text
Prepared Environment
       ↓
Fresh Source Checkout
       ↓
Task Branch
       ↓
Build / Test / Agent Work
```

Java/Spring Boot 프로젝트라면 Prepared Environment에 다음이 포함될 수 있다.

```text
Java 21
Gradle 또는 Maven
Git
Docker CLI
PostgreSQL client
Static Analysis Tool
기본 OS package
```

Frontend E2E 환경이라면 다음이 더 중요할 수 있다.

```text
Node
npm / pnpm
Playwright
Chrome
Browser dependency
```

핵심은 Task가 시작될 때 Agent가 `Java가 없으니 설치해야겠다`부터 판단하지 않게 하는 것이다.

---

## 3. 환경 설치를 Agent의 추론 문제로 만들지 않는다

다음 흐름을 생각해 보자.

```text
Agent
→ ./gradlew test 실행
→ java: command not found
→ JDK 설치 방법 검색
→ package manager 확인
→ JDK 설치
→ JAVA_HOME 설정
→ 다시 test
```

이 작업은 Agent가 할 수 있다.

하지만 반복되는 Cloud Task마다 이 과정을 다시 수행한다면 비용이 낭비된다.

더 좋지 않은 대응은 Prompt에 설치 방법을 길게 적는 것이다.

```text
JDK가 없으면 apt로 Java 21을 설치하고
JAVA_HOME을 설정한 뒤 Gradle을 실행하고...
```

환경 문제를 Prompt로 보상하고 있다.

권장 방식은 환경 자체를 수정하는 것이다.

```text
backend-test image
→ Java 21 preinstall
→ Gradle 준비
→ 필요한 OS package 설치
```

Playwright Browser가 반복해서 없어진다면 Agent에게 설치 방법을 설명할 것이 아니라 `frontend-e2e` 환경에 Browser를 포함한다.

> 반복되는 환경 실패는 Agent 지시문보다 Environment Definition을 먼저 고친다.

---

## 4. Cloud Environment as Code

준비된 환경을 사람이 수동으로 만들어 두기만 하면 재현성이 떨어진다.

누군가 서버에 직접 들어가 JDK를 설치하고 package를 추가하면 다음 질문에 답하기 어려워진다.

```text
현재 Java version은 무엇인가?
어떤 package가 설치되어 있는가?
지난주 Worker와 같은 환경인가?
환경을 다시 만들 수 있는가?
```

따라서 Cloud Environment도 코드로 관리한다.

예:

```text
cloud-env/
├─ backend-test/
│  ├─ Dockerfile
│  └─ warm-cache.sh
├─ frontend-e2e/
│  ├─ Dockerfile
│  └─ install-browser.sh
├─ migration-test/
│  └─ Dockerfile
└─ versions.env
```

`versions.env`에는 프로젝트에서 관리할 Runtime 기준을 둘 수 있다.

```text
JAVA_VERSION=21
NODE_VERSION=22
GRADLE_VERSION=8.x
```

특정 버전 숫자는 프로젝트 정책에 따라 달라진다.

중요한 점은 환경이 사람의 기억이 아니라 파일로 설명된다는 것이다.

```text
Git Commit
→ Environment Definition
→ Image Build
→ Worker Runtime
```

이렇게 해야 환경 변경도 코드 변경처럼 Review할 수 있다.

---

## 5. 모든 Task에 같은 Environment를 주지 않는다

Cloud Worker가 할 수 있는 일이 많다고 해서 하나의 거대한 Image에 모든 Tool을 넣을 필요는 없다.

`campus-platform` 기준으로 환경을 나누면 다음과 같다.

### backend-test

```text
Java 21
Gradle
Docker CLI
PostgreSQL Client
Testcontainers 관련 Image Cache
```

### frontend-e2e

```text
Node
Playwright
Chrome
npm Cache
Browser Cache
```

### migration-test

```text
Java
Migration Tool
PostgreSQL
DB Client
```

### fullstack

```text
Java
Node
Docker
Browser
```

Task Routing은 다음처럼 단순해진다.

```text
Backend Unit / Integration
→ backend-test

Frontend E2E
→ frontend-e2e

Migration Validation
→ migration-test

Cross-stack Bug
→ fullstack
```

항상 `fullstack` 하나만 사용하면 Image가 커지고 준비시간이 길어질 수 있다.

또 Task가 사용하지 않는 Tool까지 모두 포함하게 된다.

Cloud 환경도 Task Scope와 같은 원리로 작게 나누는 편이 좋다.

---

## 6. Cache는 무엇을 재사용할지 결정하는 문제다

Ephemeral Worker라고 해서 모든 것을 매번 처음부터 다운로드할 필요는 없다.

반복 비용이 큰 항목은 Cache 후보가 된다.

Java 프로젝트에서는 다음이 대표적이다.

```text
Gradle Dependency Cache
Maven Repository
Docker Base Layer
Testcontainers Image
```

Frontend에서는 다음이 있다.

```text
npm Cache
pnpm Store
Playwright Browser Binary
```

예를 들어 매 Cloud Task마다 1GB 이상의 Dependency를 다시 다운로드하면 실제 코드 수정보다 다운로드가 더 오래 걸릴 수 있다.

그래서 다음 구조를 사용할 수 있다.

```text
Reusable Cache
      ↓
Fresh Runtime
      ↓
Build / Test
```

그러나 Cache를 많이 유지한다고 항상 좋은 것은 아니다.

중요한 것은 **재사용해도 되는 상태와 반드시 초기화해야 하는 상태를 구분하는 것**이다.

---

## 7. Reusable Cache와 Fresh State를 분리한다

재사용하기 좋은 예:

```text
Gradle dependency
Maven artifact
npm package cache
Docker layer
Browser binary
Compiler cache
Immutable generated artifact
```

반대로 실행마다 초기화하는 편이 좋은 상태가 있다.

```text
Source Checkout
Task Branch
Database State
Temporary File
Mutable Fixture
Test Output
Process State
Session State
Generated User Data
```

기본 구조는 다음과 같다.

```text
Prepared Environment
├─ Runtime
├─ Tools
└─ Reusable Cache
       +
Fresh State
├─ Source
├─ Branch
├─ Task
├─ DB
└─ Temporary Data
       ↓
Execution
```

이 분리가 중요한 이유는 두 목표가 서로 충돌할 수 있기 때문이다.

```text
빠르게 시작하고 싶다
vs
깨끗한 상태에서 재현하고 싶다
```

Cache는 전자를 돕는다.

Fresh State는 후자를 지킨다.

Cloud Agent 환경 설계는 둘 중 하나를 고르는 것이 아니라 둘을 분리하는 일이다.

---

## 8. Source와 Runtime State는 Fresh하게 유지한다

Cloud Worker가 이전 작업의 Source나 DB 상태를 그대로 이어받으면 예상하지 못한 문제가 생길 수 있다.

예:

```text
Task A
→ DB에 test user 생성
→ 종료

Task B
→ 같은 DB 상태 재사용
→ 이미 존재하는 test user 때문에 실패
```

또는:

```text
Task A
→ generated file 수정
→ Worker 종료하지 않음

Task B
→ stale generated file 사용
```

이런 문제는 Agent가 코드를 잘못 작성해서가 아니다.

환경 오염이다.

그래서 책의 기본값은 다음으로 둔다.

```text
Environment / Dependency
→ 재사용 가능

Source / Branch / Runtime Data / Test Output
→ Fresh
```

Task 간 상태 공유가 반드시 필요한 경우에만 명시적으로 예외를 둔다.

---

## 9. Cache Key가 없으면 오래된 상태를 재사용할 수 있다

Cache는 `있다/없다`보다 `언제 같은 Cache로 볼 것인가`가 중요하다.

예를 들어 Gradle Dependency Cache를 생각해 보자.

다음 값이 바뀌면 Cache 유효성도 달라질 수 있다.

```text
OS / Architecture
JDK Version
Gradle Version
Dependency Lock
Build Script
```

개념적인 Cache Key는 다음처럼 만들 수 있다.

```text
gradle-cache
= os
+ architecture
+ jdk-version
+ dependency-lock-hash
```

Frontend라면 `package-lock.json` 또는 동등한 lock file hash를 사용할 수 있다.

Docker Layer는 Dockerfile과 build context 변경 영향을 받는다.

Cache Invalidation을 무시하면 다음 문제가 생긴다.

```text
이전 Dependency 잔존
Stale Generated Code
다른 Branch 결과 혼입
Test Pollution
```

Cache를 오래 유지하는 것이 목표가 아니다.

**정확하게 다시 사용할 수 있을 때만 재사용하는 것**이 목표다.

---

## 10. Snapshot은 준비된 상태를 빠르게 복원하기 위한 수단이다

Image나 Snapshot을 사용하면 Worker 생성 후 모든 설치 과정을 다시 반복하지 않아도 된다.

예:

```text
Base Image
→ Java / Node 설치
→ Tool 설치
→ Dependency 준비
→ Browser 설치
→ Snapshot 생성
```

Task 시작:

```text
Snapshot
→ Fresh Source Checkout
→ Task Branch
→ 실행
```

Snapshot에 Source까지 포함할 수도 있다.

하지만 Source는 빠르게 변한다.

Snapshot 속 Source가 오래되면 최신 Commit과 차이를 맞추는 과정이 필요해진다.

그래서 책에서는 다음을 기본값으로 사용한다.

```text
Environment / Dependency
→ Snapshot 또는 Cache

Source / Branch
→ Fresh Checkout
```

환경은 재사용하되 작업 상태는 새로 시작한다.

---

## 11. Warm Worker와 Ephemeral Worker

Worker를 Task마다 새로 만드는 방식만 있는 것은 아니다.

짧은 작업이 매우 자주 반복된다면 준비된 Worker를 READY 상태로 유지할 수 있다.

예:

```text
lint
compile
small unit test
PR verification
```

이런 Task는 Worker 생성시간이 작업시간보다 클 수 있다.

Warm Worker를 사용하면 시작시간을 줄일 수 있다.

반대로 다음 작업은 격리된 Ephemeral Worker가 더 단순할 수 있다.

```text
대규모 Integration Test
Full E2E
독립 Feature 작업
Migration Validation
```

Warm Worker에는 상태 오염 위험이 있다.

따라서 다음을 확인해야 한다.

```text
Source reset 가능한가?
Process 정리 가능한가?
DB state 초기화 가능한가?
Temporary file 제거 가능한가?
```

이 책에서는 Warm Worker를 Agent Platform Scheduler 문제로 확장하지 않는다.

Cloud Task의 Cold Start를 줄이는 하나의 선택지로만 다룬다.

---

## 12. Secret은 Prepared Image에 넣지 않는다

Prepared Environment에는 Runtime과 Tool을 넣는다.

Secret은 별도로 다룬다.

```text
Prepared Environment
- Runtime
- Tool
- Cache

Execution Time
- Token
- Credential
- Secret
```

예를 들어 Registry Credential이나 Test API Token을 Image에 포함시키면 Image 자체가 Secret을 운반하게 된다.

대신 Task가 실행될 때 필요한 범위만 주입한다.

또 모든 Cloud Task에 같은 Secret을 제공할 필요도 없다.

```text
unit-test
→ Secret 없음

integration-test
→ Test DB Credential

registry-push
→ Registry Credential
```

Cloud에 제공할 수 없는 Secret이나 내부망 자원이 있다면 5장에서 다룬 Routing Hard Constraint로 돌아간다.

환경 준비 기술로 보안 정책을 우회하려 하지 않는다.

---

## 13. Cold Start는 측정해야 개선할 수 있다

환경 최적화는 느낌으로 판단하지 않는다.

최소한 다음 값을 측정할 수 있다.

```text
worker_provision_ms
checkout_ms
dependency_restore_ms
image_pull_ms
browser_ready_ms
first_command_ms
```

예를 들어 다음 결과가 나올 수 있다.

```text
Before
Worker Ready: 7m 40s

After
Worker Ready: 55s
```

이 숫자는 설명을 위한 예시다.

실제 프로젝트에서는 직접 측정한다.

특히 Task별로 측정하는 편이 좋다.

```text
backend-test cold start
frontend-e2e cold start
migration-test cold start
```

환경마다 병목이 다르기 때문이다.

Backend는 Gradle Dependency가 문제일 수 있고, E2E는 Browser 설치가 더 클 수 있다.

---

## 14. campus-platform의 환경을 나누어 본다

`campus-platform`에서 세 가지 Cloud Environment를 만든다고 가정하자.

### backend-test

```text
Java 21
Gradle
Docker CLI
PostgreSQL Client
Gradle Dependency Cache
Testcontainers Image Cache
```

사용 Task:

```text
Unit Test
Integration Test
Backend Bug Fix
Refactoring Validation
```

### frontend-e2e

```text
Node
Playwright
Chrome
npm Cache
Browser Cache
```

사용 Task:

```text
Admin Web E2E
UI Regression
Screenshot / Video Validation
```

### migration-test

```text
Java 21
Migration Tool
PostgreSQL
DB Client
```

사용 Task:

```text
Migration Syntax
Clean Apply
Upgrade Validation
```

공통 Fresh State는 다음처럼 둔다.

```text
Fresh Git Checkout
Task Branch
Disposable Test DB
New Test Output Directory
```

Tibero, HSM, Internal Jenkins처럼 Cloud에서 제공할 수 없는 자원은 Prepared Environment에 억지로 넣지 않는다.

그 부분은 Local 또는 Hybrid로 남긴다.

---

## 15. 하나의 거대한 Image보다 Task-specific Environment를 먼저 본다

모든 Tool을 한 Image에 넣으면 편해 보인다.

```text
fullstack-all-in-one
- Java
- Node
- Docker
- Browser
- DB Client
- Migration Tool
- Android Tool
- Static Analysis Tool
- 기타 모든 Tool
```

하지만 모든 Task가 이를 필요로 하지는 않는다.

작은 Unit Test를 위해 Browser와 Frontend Tool까지 준비할 이유가 없다.

그래서 환경도 Task Routing과 연결한다.

```text
Task Type
→ Required Tool
→ Environment Selection
```

이 구조는 10장의 Runner 역할 분리와 자연스럽게 이어진다.

```text
runner-unit
→ backend-test

runner-e2e
→ frontend-e2e

runner-migration
→ migration-test
```

즉 Prepared Environment는 Cloud Worker 하나를 위한 거대한 컴퓨터가 아니라 **Task별로 선택할 수 있는 실행 기반**이다.

---

## 16. 준비된 환경이 있어야 Runner-first가 단순해진다

10장에서는 정상 경로를 Runner가 처리하고 실패할 때만 Agent를 호출한다.

이 구조가 잘 동작하려면 Runner가 매번 환경을 만들 필요가 없어야 한다.

좋지 않은 흐름:

```text
Runner Start
→ JDK 설치
→ Dependency Download
→ Browser 설치
→ Test
```

권장 흐름:

```text
Prepared Environment
→ Fresh Checkout
→ Test
```

환경이 이미 준비되어 있으면 Runner는 더 단순해진다.

```text
Input
Git SHA + Command

Execution
Command 실행

Output
Status + Artifact
```

이제 다음 장에서는 이 Prepared Environment 위에서 Build/Test/E2E/Docker 같은 결정론적 작업을 LLM 없이 반복 실행하는 구조를 다룬다.

---

## 이 장에서 기억할 것

Cloud Agent가 Task를 시작할 때마다 개발환경부터 설치한다면 Cloud의 비동기성과 병렬성 이점이 환경 준비 비용에 묻힐 수 있다.

그래서 다음 원칙을 사용한다.

```text
Environment는 준비한다.
Dependency와 Tool Cache는 재사용한다.
Source와 Runtime State는 Fresh하게 시작한다.
Task별 Environment를 선택한다.
Cold Start를 측정한다.
```

그리고 환경에서 반복되는 문제를 발견하면 Prompt를 길게 만들기 전에 환경 정의를 수정한다.

> Agent에게 개발환경을 설치하게 하지 말고, 바로 작업 가능한 환경을 제공한다.

다음 장에서는 이렇게 준비된 환경을 `Cloud Runner`로 사용한다.