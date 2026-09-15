# 9장 설계 - Prepared Cloud Environment, Cache, Snapshot

## 장의 목표

Cloud Agent가 Task를 받았을 때 환경 설치부터 시작하지 않고 바로 작업할 수 있도록 `Prepared Cloud Environment`를 설계한다.

3장에서 Compute와 Token을 분리했고, 4장에서 독립 실행환경의 가치를 설명했으며, 7~8장에서 Context와 Tool Output 비용을 줄였다.

9장에서는 Cloud Worker가 실제 작업을 시작하기 전 발생하는 `환경 준비 비용`을 줄인다.

핵심 질문:

> Cloud Agent가 매번 JDK, Node, Dependency, Browser, Docker Image를 다시 준비하지 않고 바로 작업을 시작하게 하려면 무엇을 미리 준비해야 하는가?

---

## 핵심 주장

> Agent에게 개발환경을 설치하게 하지 말고, 바로 작업 가능한 환경을 제공한다.

> Cloud 환경도 Source Code처럼 버전 관리하고 재현 가능하게 만든다.

> 재사용 가능한 Cache와 매 실행 초기화해야 하는 Fresh State를 분리한다.

Cloud Agent의 시작 비용에는 다음이 포함될 수 있다.

- VM/Container 생성
- Repository clone
- JDK/Node 설치
- Gradle/Maven/npm dependency 다운로드
- Docker image pull
- Playwright browser 설치
- Code generation
- build cache 생성

이 시간을 `Cloud Cold Start`의 일부로 본다.

---

## 독자가 얻는 것

- Cloud Agent Cold Start를 구성 요소별로 분해할 수 있다.
- Prepared Cloud Environment를 팀 자산으로 관리할 수 있다.
- Base Image, Snapshot, Cache의 역할을 구분할 수 있다.
- 작업별로 서로 다른 Cloud Environment를 선택할 수 있다.
- Reusable Cache와 Disposable/Fresh Runtime State를 분리할 수 있다.
- Cache 때문에 테스트 재현성이 깨지는 문제를 피할 수 있다.
- Java/Spring Boot 환경에서 Gradle/Testcontainers/Docker 준비 비용을 줄일 수 있다.
- Warm Worker가 유리한 Task와 Ephemeral Worker가 유리한 Task를 구분할 수 있다.

---

# 절 구성

## 9.1 Cloud Agent의 실제 시작시간은 모델 응답시간만이 아니다

좋지 않은 측정:

```text
Cloud Agent 응답 시작
→ 첫 LLM 응답 시간만 측정
```

실제 Task 시작 전에는 다음 단계가 존재할 수 있다.

```text
Worker 생성
→ Repository checkout
→ Runtime 확인
→ Dependency 준비
→ Docker/Browser 준비
→ Build bootstrap
→ Agent 작업 시작
```

따라서 다음 값을 구분한다.

```text
Provisioning Time
Checkout Time
Dependency Restore Time
Environment Warm-up Time
Agent Execution Time
```

환경 준비가 8분이고 실제 수정이 3분이라면 모델만 빠르게 바꿔도 전체 시간은 크게 줄지 않는다.

---

## 9.2 Prepared Cloud Environment

권장 준비 흐름:

```text
Base Image
   ↓
Runtime 설치
   ↓
Development Tools 설치
   ↓
Dependency 준비
   ↓
Code Generation
   ↓
Cache Warming
   ↓
Snapshot / Image 생성
```

Worker 실행:

```text
Prepared Environment
       ↓
Fresh Branch Checkout
       ↓
Task 실행
```

Prepared Environment에 포함할 수 있는 것:

- Java 21
- Gradle / Maven
- Node
- npm/pnpm/yarn
- Docker CLI
- Playwright browser
- Static Analysis Tools
- DB client
- 기본 OS package

Repository 자체를 Snapshot에 얼마나 포함할지는 변경 빈도와 보안 정책에 따라 결정한다.

---

## 9.3 Cloud Environment as Code

환경 구성을 사람의 기억이나 수동 설정에 의존하지 않는다.

후보:

- Dockerfile
- Dev Container
- image build script
- package manifest
- toolchain version file
- bootstrap script

예:

```text
cloud-env/
├─ Dockerfile
├─ versions.env
├─ install-tools.sh
└─ warm-cache.sh
```

환경 정의도 Repository의 변경 이력과 함께 관리한다.

환경 버전 예:

```text
cloud-env: backend-test-v12
java: 21
node: 22
playwright: pinned
```

제품별 환경 기능에 종속하지 않고 `환경 정의를 재현 가능한 코드로 관리한다`는 원칙을 설명한다.

---

## 9.4 모든 Task에 같은 Environment가 필요하지 않다

Task별 환경 예:

```text
backend-test
- Java 21
- Gradle
- Docker
- Testcontainers
- DB client

frontend-e2e
- Node
- Chrome
- Playwright

migration-test
- Java
- Flyway
- DB client
- migration scripts

fullstack
- Java
- Node
- Docker
- Browser
```

Routing:

```text
Backend Unit/Integration
→ backend-test

Frontend E2E
→ frontend-e2e

Migration Validation
→ migration-test

Cross-stack Bug
→ fullstack
```

필요하지 않은 Tool을 모든 Worker에 넣으면 image size, startup, attack surface가 커질 수 있다.

---

## 9.5 Reusable Cache와 Fresh State를 분리한다

재사용 가능한 상태:

- Gradle cache
- Maven repository
- npm cache
- Docker layer
- Playwright browser
- compiler cache
- immutable code generation artifact
- LLM과 무관한 Repository index

매 실행 초기화할 상태:

- Source checkout
- Branch
- Task input
- DB state
- temporary file
- mutable fixture
- generated user data
- test output
- process/session state

구조:

```text
Prepared Environment
- Runtime
- Tools
- Dependency Cache
        +
Fresh State
- Source
- Branch
- Task
- Runtime Data
        ↓
Execution
```

핵심은 `빠른 시작`과 `깨끗한 실행`을 동시에 확보하는 것이다.

---

## 9.6 Java/Spring Boot에서 Cache 영향이 큰 영역

Java 프로젝트에서는 다음 반복 다운로드가 전체 실행시간에 영향을 줄 수 있다.

- Gradle dependency
- Maven artifact
- Docker base image
- Testcontainers image

예:

```text
매 Task
→ dependency 1.5GB download
→ Docker base image pull
→ Testcontainers image pull
```

보다:

```text
Prepared Cache
→ dependency restore
→ Docker layer reuse
→ Fresh Test Runtime
```

를 사용한다.

다만 Gradle build output이나 test result처럼 Source 상태에 강하게 의존하는 결과는 무조건 재사용하지 않는다.

---

## 9.7 Cache Key와 Invalidation

Cache는 오래 유지하는 것보다 `언제 폐기할지`가 중요하다.

Cache Key 후보:

- OS/architecture
- JDK version
- Gradle/Maven lock state
- package-lock/package manifest hash
- Dockerfile hash
- tool version

예:

```text
gradle-cache
key = os + jdk + gradle-lock-hash
```

잘못된 Cache로 인한 문제:

- 이전 dependency 잔존
- stale generated code
- 다른 branch 결과 혼입
- test pollution

따라서 Runtime State는 Cache와 분리한다.

---

## 9.8 Snapshot

Snapshot은 Worker를 빠르게 시작하기 위한 준비된 상태다.

예:

```text
Base Image
→ tools install
→ dependency restore
→ browser install
→ snapshot
```

Task 실행:

```text
Snapshot
→ Fresh Source checkout
→ Branch checkout
→ Task
```

Snapshot에 Source Code를 포함할 경우 최신 commit과의 차이를 적용하는 비용, stale source 위험을 함께 고려한다.

책의 기본 예제는 `환경/Dependency는 재사용, Source/Branch는 Fresh`를 기본값으로 둔다.

---

## 9.9 Warm Worker

짧고 반복되는 Task는 항상 새 Worker를 만드는 것보다 READY 상태 Worker가 유리할 수 있다.

적합한 예:

- lint
- compile
- 작은 unit test
- PR verification

장시간/격리 중요 Task:

- 대규모 integration test
- full E2E
- 독립 feature work

에는 별도 Ephemeral Worker가 더 단순할 수 있다.

Warm Worker는 미래 Agent Platform 일반론으로 확장하지 않고 `Cloud Task 시작시간을 줄이는 선택지`로만 설명한다.

---

## 9.10 Environment가 실패하면 Prompt보다 Environment를 고친다

반복되는 문제:

```text
Agent
→ JDK 못 찾음
→ 설치 방법 추론
→ install 실패
→ 다시 추론
```

좋지 않은 대응:

```text
Prompt에 JDK 설치 방법을 더 길게 적는다.
```

권장 대응:

```text
Prepared Image에 JDK 추가
→ 환경 버전 갱신
```

다른 예:

```text
Playwright browser 없음
→ Agent가 설치 시도
```

보다:

```text
frontend-e2e image에 browser preinstall
```

환경 문제를 Agent reasoning 문제로 만들지 않는다.

---

## 9.11 Secret과 Environment를 섞지 않는다

Prepared Image에 Secret을 bake하지 않는다.

구분:

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

Cloud Agent에 어떤 Secret을 제공할 수 있는지는 5장 Routing과 13장 Hybrid 판단으로 연결한다.

---

## 9.12 Cold Start를 측정한다

측정 후보:

```text
worker_provision_ms
checkout_ms
dependency_restore_ms
image_pull_ms
browser_ready_ms
first_command_ms
```

최적화 전후를 비교한다.

예:

```text
Before
Worker ready: 7m 40s

After Prepared Environment
Worker ready: 55s
```

수치는 예시이며 실제 프로젝트에서 측정한다.

목표는 `Agent 자체를 빠르게 보이게 하는 것`이 아니라 `Task가 실제 실행되기까지의 시간을 줄이는 것`이다.

---

## 9.13 campus-platform Environment 예제

### backend-test

```text
Java 21
Gradle
PostgreSQL client
Docker CLI
Testcontainers image cache
Gradle dependency cache
```

### frontend-e2e

```text
Node
Playwright
Chrome
npm cache
browser cache
```

### migration-test

```text
Java 21
Migration Tool
PostgreSQL
DB client
```

### Fresh State

모든 환경 공통:

```text
Fresh Git checkout
Task Branch
Disposable test DB
New test output directory
```

Tibero/HSM/Jenkins처럼 Cloud에서 제공하지 않는 내부 자원은 Prepared Environment로 해결하려 하지 않고 Hybrid로 남긴다.

---

# 좋은 사례와 나쁜 사례

## 매번 설치

좋지 않은 방식:

```text
Agent Start
→ JDK install
→ Node install
→ Gradle download
→ npm install
→ Browser install
→ Test
```

권장 방식:

```text
Prepared Environment
→ Fresh checkout
→ Test
```

## 모든 상태 Cache

좋지 않은 방식:

```text
Dependency + DB + temp + test output
→ 모두 재사용
```

권장 방식:

```text
Dependency/Tool Cache 재사용
Runtime/Test State 초기화
```

## 하나의 거대한 Image

좋지 않은 방식:

```text
모든 Task
→ fullstack image
```

권장 방식:

```text
Task Type
→ 필요한 Environment 선택
```

---

# 필요한 그림

1. Cold Start Breakdown
2. Prepared Environment 생성 흐름
3. Reusable Cache + Fresh State
4. Task-specific Environment Routing
5. Snapshot / Warm Worker 비교

---

# Phase 6 구현 후보

```text
cloud-env/
├─ backend-test/
├─ frontend-e2e/
├─ migration-test/
└─ versions.env

scripts/
├─ build-env.sh
├─ warm-cache.sh
└─ verify-env.sh
```

실제 Cloud 제품 종속 구성보다 Dockerfile/스크립트 수준의 재현 가능한 예제를 우선한다.

---

# 필요한 조사

- Gradle build cache/dependency cache 공식 동작
- Docker layer cache 공식 문서
- Playwright browser caching/installation
- 주요 Cloud coding agent의 environment/preinstall/snapshot 지원 방식

제품별 현재 환경 사양과 제한은 research 문서로 분리한다.

---

# 앞뒤 장 연결

8장:

```text
Result/Output 비용 감소
```

9장:

```text
Environment Startup 비용 감소
```

10장:

```text
준비된 환경에서 결정론적 Runner를 반복 실행
```

---

# 의도적으로 다루지 않을 내용

- 범용 Sandbox Scheduler
- Kubernetes 운영 입문
- Agent Platform Resource Broker
- Image Registry 운영 상세
- Secret Management 제품 비교

---

# 장의 결론 메시지

> Agent에게 개발환경을 설치하게 하지 말고, 바로 작업 가능한 환경을 제공한다.

> 빠른 Cloud Agent는 빠른 모델만으로 만들어지지 않는다. Task가 시작되기까지의 환경 준비시간도 줄여야 한다.

> Cache는 재사용하되 Source와 Runtime State는 Fresh하게 유지한다.
