# 3장 설계 - Agent Ready 프로젝트의 기준

## 장의 목표

`Agent Ready`를 추상적인 표현이나 특정 AI 제품의 지원 여부가 아니라 프로젝트가 실제로 제공하는 확인 가능한 특성으로 정의한다.

이 장에서는 이후 장 전체에서 반복해서 사용할 공통 평가 기준을 만든다. 각 기준은 설명으로 끝내지 않고 실제 Repository, 실행 명령, 테스트 결과, 권한 정책처럼 확인 가능한 증거와 연결한다.

핵심 질문은 다음과 같다.

> 이 프로젝트를 처음 보는 Agent가 사람의 지속적인 설명 없이 작업을 발견하고, 환경을 구성하고, 실행하고, 검증하며, 안전한 범위 안에서 결과를 남길 수 있는가?

2장에서 정의한 Local / Cloud / Hybrid와 Runner-first 구조는 실행 방식이다. 3장에서는 그러한 실행 방식이 가능한 프로젝트인지 평가하는 기준을 정의한다.

---

## 문제 정의

`Agent Ready`라는 표현은 쉽게 다음과 같이 오해될 수 있다.

- `AGENTS.md`가 있으면 Agent Ready다.
- Claude Code나 Codex가 Repository를 열 수 있으면 Agent Ready다.
- 테스트가 많으면 Agent Ready다.
- Dockerfile이 있으면 Cloud Agent에서 실행 가능하다.
- Agent가 한 번 작업에 성공했으니 Agent Ready다.

하지만 실제 작업에서는 하나의 조건만 충족해도 다음 단계에서 막힐 수 있다.

```text
Repository 탐색 성공
        ↓
setup 실패
        ↓
사람 개입
```

또는:

```text
build 성공
    ↓
test 성공
    ↓
완료 조건을 판단할 방법 없음
    ↓
사람 개입
```

또는:

```text
Cloud Agent 작업 성공
        ↓
내부 DB/HSM 검증 불가
        ↓
실제 완료 여부 확인 불가
```

따라서 Agent Ready 여부는 하나의 기능이 아니라 여러 프로젝트 속성의 조합으로 평가해야 한다.

---

## 핵심 주장

> Agent Ready는 Agent 제품의 능력이 아니라 프로젝트가 제공하는 작업 가능성의 속성이다.

그리고 다음 원칙을 사용한다.

> 평가 항목마다 설명이 아니라 증거를 요구한다.

예:

```text
"테스트가 잘 되어 있다"
```

보다:

```text
./scripts/verify.sh
exit code 0
fresh clone 환경에서 재현 가능
```

을 증거로 사용한다.

또한 여러 기준을 단순 평균한 하나의 점수만으로 프로젝트를 평가하지 않는다.

예를 들어 다음 프로젝트는 평균 점수가 높아도 Cloud Agent 작업에 적합하지 않다.

```text
Discoverability   PASS
Testability       PASS
Verifiability    PASS
Observability     PASS
Security Boundary FAIL
```

보안 경계가 없으면 나머지 조건이 좋아도 Agent에게 Repository 쓰기나 Secret 접근을 맡길 수 없다.

따라서 이 장에서는 `Agent Ready Score`보다 `Agent Ready Profile`을 기본 표현으로 사용한다.

---

## 독자가 얻는 것

- Agent Ready를 제품 기능과 분리해 설명할 수 있다.
- 기존 프로젝트를 공통 기준으로 진단할 수 있다.
- 각 기준을 `PASS / PARTIAL / FAIL`로 평가할 수 있다.
- 평가마다 필요한 증거를 정의할 수 있다.
- 어떤 약점이 Local/Cloud/Parallel Agent 도입을 막는지 찾을 수 있다.
- 이후 장에서 어떤 문제를 먼저 개선해야 하는지 우선순위를 정할 수 있다.
- Runner가 처리할 수 있는 정상 경로와 Agent 판단이 필요한 예외 경로가 분리되어 있는지 평가할 수 있다.

---

## 예제 Stage

Stage 0 → Stage 1 사이의 Baseline Assessment

아직 `campus-platform`을 본격적으로 개선하지 않는다.

Stage 0 상태를 평가해 이후 장에서 어떤 항목이 어떻게 개선되는지 추적할 기준선을 만든다.

예:

```text
campus-platform / Stage 0

Reproducibility   PARTIAL
Discoverability  FAIL
Executability    PARTIAL
Testability      FAIL
Verifiability    FAIL
Isolation        FAIL
Parallelizability PARTIAL
Observability    FAIL
Security Boundary FAIL
```

이후 각 장이 끝날 때 해당 항목의 상태가 어떻게 변했는지 연결할 수 있다.

---

# 평가 방법

## PASS / PARTIAL / FAIL

### PASS

사람의 추가 설명 없이 Repository와 자동화된 환경에서 조건을 반복해서 충족할 수 있다.

### PARTIAL

일부 자동화가 존재하지만 사람의 사전 작업, 특정 개발자 PC, 수동 판단 또는 숨겨진 환경에 의존한다.

### FAIL

해당 기능이 없거나 Agent가 독립적으로 사용할 수 없다.

평가표에는 반드시 `Evidence`를 함께 기록한다.

```text
Criterion: Executability
Status: PASS
Evidence: ./scripts/test.sh
Expected: exit code 0/1로 결과 판정 가능
```

`PASS`라는 상태만 기록하고 증거를 남기지 않는 방식은 사용하지 않는다.

---

# 핵심 평가 기준

## 3.1 Reproducibility - 새로운 환경에서 같은 상태를 만들 수 있는가

### 확인 질문

- fresh clone에서 필요한 환경을 만들 수 있는가?
- JDK, Node, Gradle 등 toolchain 버전이 명시되어 있는가?
- 필요한 의존성을 자동 설치하거나 명확히 검증할 수 있는가?
- 개발자 개인 PC에만 존재하는 설정이 있는가?
- setup을 여러 번 실행해도 같은 결과를 만드는가?

### 좋은 증거

```text
./scripts/setup.sh
```

또는 동등한 bootstrap 진입점.

### 실패 사례

```text
README:
"개발팀 공용 서버에서 설정 파일을 복사하세요."
```

사람에게는 익숙할 수 있지만 Cloud Agent나 신규 개발 환경에서는 재현할 수 없다.

### 후속 장

7장 `재현 가능한 개발환경`

---

## 3.2 Discoverability - 무엇을 어디서 찾아야 하는지 알 수 있는가

### 확인 질문

- Repository의 목적이 명확한가?
- 주요 모듈과 진입점을 찾을 수 있는가?
- build/test/verify 명령 위치가 명확한가?
- canonical 문서와 파생 문서를 구분할 수 있는가?
- generated file과 직접 수정 가능한 파일을 구분할 수 있는가?
- Agent가 전체 Repository를 무작정 읽지 않아도 필요한 영역으로 좁힐 수 있는가?

### 핵심 관점

Discoverability는 단순히 문서 개수를 늘리는 것이 아니다.

```text
적은 Context
    ↓
관련 영역 발견
    ↓
필요할 때만 상세 Context 확장
```

이 구조가 가능해야 한다.

### 후속 장

4장 `Repository as Interface`
5장 `Agent Contract`

---

## 3.3 Executability - 사람이 IDE를 조작하지 않아도 명령으로 실행 가능한가

### 확인 질문

- build 명령이 하나의 명확한 진입점으로 제공되는가?
- test, lint, validation을 CLI에서 실행할 수 있는가?
- 성공과 실패를 exit code로 판단할 수 있는가?
- CI와 Local/Cloud Runner가 같은 명령을 사용할 수 있는가?
- GUI 또는 수동 클릭만 가능한 필수 단계가 있는가?

### 중요한 평가 항목

2장의 Runner-first 원칙과 연결한다.

다음 작업이 LLM 없이 실행 가능한지 확인한다.

- Build
- Unit Test
- Integration Test
- Lint
- Static Analysis
- Migration Validation
- E2E

즉 다음 경로가 존재해야 한다.

```text
Task
  ↓
Runner
  ↓
PASS / FAIL
```

정상 경로를 실행하기 위해 항상 LLM이 필요하다면 Executability가 낮다고 평가한다.

### 후속 장

8장 `Agent 실행 인터페이스`

---

## 3.4 Testability - 외부 환경이 없어도 필요한 범위를 검증할 수 있는가

### 확인 질문

- 비즈니스 로직을 단위 테스트할 수 있는가?
- DB, Redis, Kafka, 외부 API가 없으면 모든 테스트가 중단되는가?
- Fake, Mock, Testcontainers 등 대체 경로가 있는가?
- 실패를 재현할 수 있는가?
- 특정 테스트만 선택 실행할 수 있는가?

### 평가 관점

테스트 개수보다 `Agent가 자신의 변경을 검증할 수 있는가`를 본다.

테스트가 10,000개 있어도 VPN 내부 DB 없이는 하나도 실행할 수 없다면 Cloud Agent 관점의 Testability는 낮다.

### 후속 장

9장 `외부 시스템 없이 테스트 가능한 구조`

---

## 3.5 Verifiability - 완료 여부를 기계적으로 판단할 수 있는가

### 확인 질문

- 작업 완료 조건이 명시되어 있는가?
- build/test/static rule 결과를 자동 수집할 수 있는가?
- 사람의 육안 확인만으로 완료를 판단하는 단계가 있는가?
- 변경 범위가 Task Contract를 벗어나지 않았는지 확인할 수 있는가?
- PASS/FAIL 결과와 원본 artifact를 남길 수 있는가?

### 구분

```text
Testability
= 테스트를 실행할 수 있는가

Verifiability
= 작업이 완료되었다고 판정할 수 있는가
```

두 개념을 같은 것으로 취급하지 않는다.

### 후속 장

10장 `자동 검증과 Agent Definition of Done`
11장 `Architecture Rule을 코드로 검증하기`

---

## 3.6 Isolation - 작업이 다른 작업과 환경에 미치는 영향을 제한할 수 있는가

### 확인 질문

- 작업별 branch/worktree/sandbox를 만들 수 있는가?
- 테스트 데이터가 다른 실행에 영향을 주지 않는가?
- 외부 시스템 변경을 Fake 또는 isolated resource로 대체할 수 있는가?
- Agent가 작업 중 다른 Agent의 파일을 덮어쓸 가능성이 있는가?
- 실패한 작업 환경을 폐기하고 새 환경에서 재시도할 수 있는가?

### 예

```text
Agent A → worktree A → test DB A
Agent B → worktree B → test DB B
```

### 후속 장

9장, 13장

---

## 3.7 Parallelizability - 작업을 독립적으로 분리하고 동시에 실행할 수 있는가

### 확인 질문

- 기능 변경이 특정 모듈에 국소화되어 있는가?
- 여러 작업이 동일한 common file을 반복 수정하는가?
- migration 번호나 공유 설정 파일이 병렬 작업의 병목이 되는가?
- 테스트를 module 또는 task 단위로 fan-out할 수 있는가?
- 각 Worker가 Repository 전체를 다시 분석해야 하는가?

### 2장과의 연결

병렬성은 Agent 수를 늘리는 것이 아니다.

```text
독립된 작업 단위
+ 격리된 실행 환경
+ 최소 Context
+ 독립 검증 명령
```

이 있어야 실제 Parallelizability가 높다고 평가한다.

### 후속 장

12장 `병렬 Agent 개발을 위한 모듈 경계`
13장 `Git, Worktree, Branch, Cloud Sandbox`

---

## 3.8 Observability - 현재 무슨 일이 일어나고 있는지 확인할 수 있는가

### 확인 질문

- 작업 상태를 queued/running/failed/completed 등으로 확인할 수 있는가?
- 실행 시간과 retry 수를 기록하는가?
- 실패 원인과 원본 로그 위치를 찾을 수 있는가?
- Runner가 생성한 artifact를 추적할 수 있는가?
- LLM usage와 compute execution을 별도 지표로 볼 수 있는가?
- Agent가 왜 호출되었는지 기록할 수 있는가?

### 2장과의 연결

Runner-first 구조에서는 다음 비율도 관찰 대상이 될 수 있다.

```text
전체 Runner 실행 수
Agent 호출 수
PASS without Agent 수
FAIL → Agent 전환 수
filtered result size
raw result size
```

이 장에서는 지표의 존재 여부만 평가하고 상세 운영 모델은 17장에서 다룬다.

### 후속 장

17장 `Agent Observability와 Quality Evaluation`

---

## 3.9 Security Boundary - Agent가 할 수 있는 일과 할 수 없는 일이 명확한가

### 확인 질문

- Agent가 접근할 수 있는 Repository와 branch가 제한되는가?
- Secret 전달 범위가 명확한가?
- Production 접근이 기본적으로 분리되어 있는가?
- 외부 Issue, 문서, 로그를 비신뢰 입력으로 취급하는가?
- 명령 실행과 Tool 사용 범위를 제한할 수 있는가?
- Merge/Deploy 권한이 Worker 권한과 분리되어 있는가?
- 작업 종료 후 임시 credential을 회수할 수 있는가?

### 중요한 원칙

기능적으로 잘 동작하는 프로젝트라도 권한 경계가 불명확하면 Agent Ready라고 평가하지 않는다.

### 후속 장

14장 `Trust Boundary`
15장 `CI/CD Gate와 Agent 권한 연결`

---

# Agent Ready Profile

이 장에서 사용할 기본 평가표는 다음과 같다.

| 기준 | 상태 | 증거 | 주요 장애물 | 개선 장 |
| --- | --- | --- | --- | --- |
| Reproducibility | PASS/PARTIAL/FAIL | setup 명령/환경 정의 | 수동 환경 설정 | 7장 |
| Discoverability | PASS/PARTIAL/FAIL | README/구조/진입점 | 숨겨진 규칙 | 4~5장 |
| Executability | PASS/PARTIAL/FAIL | build/test CLI | IDE 의존 | 8장 |
| Testability | PASS/PARTIAL/FAIL | 선택 테스트/Fake/Testcontainer | 외부 시스템 의존 | 9장 |
| Verifiability | PASS/PARTIAL/FAIL | verify/DoD | 수동 판정 | 10~11장 |
| Isolation | PASS/PARTIAL/FAIL | sandbox/worktree/test resource | 공유 상태 | 9, 13장 |
| Parallelizability | PASS/PARTIAL/FAIL | 모듈/task 분할 | shared file | 12~13장 |
| Observability | PASS/PARTIAL/FAIL | 상태/로그/artifact/usage | 상태 불명 | 17장 |
| Security Boundary | PASS/PARTIAL/FAIL | 권한/secret/network policy | 과도한 권한 | 14~15장 |

## 왜 총점 하나를 기본으로 사용하지 않는가

다음 두 프로젝트가 있다고 가정한다.

```text
Project A
- 8개 기준 PASS
- Security Boundary FAIL

Project B
- 6개 기준 PASS
- 3개 기준 PARTIAL
- Security Boundary PASS
```

단순 평균 점수에서는 A가 더 높게 보일 수 있다.

하지만 실제 Cloud Agent 도입 위험은 A가 더 클 수 있다.

따라서 책에서는:

1. 각 기준별 상태를 본다.
2. 작업 유형에 필요한 최소 조건을 확인한다.
3. 치명적인 FAIL을 먼저 해결한다.
4. 마지막에만 필요하면 조직별 점수 모델을 추가한다.

22장에서 성숙도 모델을 다룰 때 이 Profile을 다시 사용한다.

---

# 작업 유형별 최소 조건

Agent Ready는 모든 작업에 동일한 수준을 요구하지 않는다.

## Cloud Test Runner

최소한 다음 기준이 중요하다.

```text
Reproducibility
Executability
Testability
Isolation
Observability
```

## Cloud Agent Worker

추가로 다음 기준이 중요하다.

```text
Discoverability
Verifiability
Security Boundary
```

## Parallel Agent Development

추가로 다음이 필요하다.

```text
Parallelizability
Isolation
Verifiability
```

## PM Agent / Orchestration

후반부에서는 다음까지 요구한다.

```text
Observability
Project Memory
Task Contract
Failure Classification
Security Boundary
```

이 구분을 통해 `프로젝트 전체가 완전히 Agent Ready가 될 때까지 기다려야 한다`는 잘못된 결론을 피한다.

예를 들어 기존 프로젝트도 먼저 Cloud Runner만 도입할 수 있다.

---

# campus-platform Baseline 예제

Stage 0 상태를 다음과 같이 평가한다.

| 기준 | 상태 | 초기 문제 |
| --- | --- | --- |
| Reproducibility | PARTIAL | JDK는 명시되어 있으나 DB 준비가 수동 |
| Discoverability | FAIL | 프로젝트 구조와 변경 경계 문서 부족 |
| Executability | PARTIAL | Gradle 명령은 있으나 통합 진입점 없음 |
| Testability | FAIL | 일부 테스트가 외부 DB에 의존 |
| Verifiability | FAIL | 전체 완료 판정 명령 없음 |
| Isolation | FAIL | 공유 개발 DB 사용 |
| Parallelizability | PARTIAL | package 구분은 있으나 shared configuration 충돌 가능 |
| Observability | FAIL | 작업/검증 상태를 구조화해 남기지 않음 |
| Security Boundary | FAIL | Agent 권한 정책 자체가 정의되어 있지 않음 |

이 평가는 실제 코드 구현 전의 책용 baseline이다.

후속 장에서 구조를 변경할 때마다 모든 항목을 반복 설명하지 않고 해당 기준만 갱신한다.

예:

```text
7장 완료
Reproducibility: PARTIAL → PASS

8장 완료
Executability: PARTIAL → PASS

9~11장 완료
Testability / Verifiability 개선
```

---

# 실패 사례

## 문서만 추가하고 Agent Ready라고 판단

```text
AGENTS.md 추가
→ setup은 여전히 수동
→ 테스트는 내부 DB 필요
→ 완료 판단은 사람
```

Discoverability 일부만 개선된 것이다.

## Dockerfile만 추가하고 Cloud Ready라고 판단

Container image를 만들 수 있어도 다음이 없으면 충분하지 않다.

- 자동 test
- external dependency 격리
- result artifact
- secret boundary

## Agent 성공률만으로 평가

특정 고성능 모델이 많은 추론과 재시도로 작업에 성공할 수 있다.

하지만 모델을 바꾸거나 사용량 제한이 생기면 동일한 프로젝트가 실패할 수 있다.

프로젝트의 readiness와 특정 모델의 problem-solving capability를 분리한다.

## LLM을 실행 인터페이스로 사용

```text
"프로젝트를 알아서 빌드하고 테스트해줘"
```

라고 매번 Agent에게 요청해야만 정상 작업이 수행된다면 Executability가 충분하지 않다.

가능한 정상 경로는 다음처럼 LLM 없이 실행 가능해야 한다.

```text
./scripts/verify.sh
```

---

# 필요한 구조/그림

## 그림 1. Agent Ready는 하나의 점수가 아니라 Profile

```text
Reproducibility ───── PASS
Discoverability ───── PARTIAL
Executability ─────── PASS
Testability ───────── FAIL
Verifiability ─────── FAIL
Isolation ─────────── PARTIAL
Parallelizability ─── PARTIAL
Observability ─────── FAIL
Security Boundary ── FAIL
```

## 그림 2. 작업 종류별 필요한 기준

```text
Cloud Runner
  └─ Execute / Test / Isolate / Observe

Agent Worker
  └─ Discover / Execute / Test / Verify / Secure

Parallel Agents
  └─ Isolate / Parallelize / Verify

PM Agent
  └─ Observe / Schedule / Recover / Govern
```

## 그림 3. Evidence-based Assessment

```text
주장
"테스트 가능"
   ↓
증거
./scripts/test.sh
   ↓
재현
fresh clone에서 실행
   ↓
판정
PASS / PARTIAL / FAIL
```

---

# 필요한 코드/스크립트 예제

이 장에서는 실제 구현 코드를 작성하지 않는다.

평가 예제로 다음 형태의 명령과 artifact 이름만 사용한다.

```text
./scripts/setup.sh
./scripts/test.sh
./scripts/verify.sh
artifacts/verification/result.json
```

각 스크립트 구현은 7~10장에서 다룬다.

향후 체크리스트를 machine-readable format으로 만들 수 있지만 3장에서는 형식을 고정하지 않는다.

---

# 필요한 공식 자료 조사

이 장의 핵심은 제품 독립적인 기준이므로 제품별 사실 조사는 최소화한다.

필요할 경우 다음만 사례로 확인한다.

- 주요 Coding Agent가 Repository 기반 작업과 명령 실행을 어떻게 사용하는지
- Cloud Agent가 isolated environment를 제공하는 사례
- CI가 exit code와 artifact를 사용하는 일반적 방식

제품 사양을 Readiness 기준 자체로 사용하지 않는다.

---

# 앞 장과 뒤 장의 연결

## 1장과의 연결

1장에서 제기한 `사람이 알고 있는 프로젝트`의 문제를 9개 평가 기준으로 구체화한다.

## 2장과의 연결

Local / Cloud / Hybrid 실행 모델과 Runner-first 원칙을 `Executability`, `Isolation`, `Observability` 관점으로 평가한다.

특히 다음 원칙을 평가 기준에 포함한다.

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.

이를 위해 정상적인 build/test/validation이 LLM 없이 실행 가능한지를 Executability 증거로 본다.

## 4장과의 연결

3장에서 Discoverability가 낮다고 진단한 프로젝트를 4장에서 `Repository as Interface` 관점으로 개선한다.

이후 장들은 각각 특정 Readiness 기준을 개선하는 과정으로 읽을 수 있다.

---

# 본문에서 의도적으로 다루지 않을 내용

- Agent Ready 성숙도 Level의 최종 단계 정의: 22장에서 다룬다.
- 특정 제품별 지원 기능 비교표
- 조직별 비용 산정 공식
- 실제 Result Filter 구현
- PM Agent scheduler 구현
- 보안 정책의 상세 구현
- 모든 기준을 하나의 숫자로 환산하는 범용 점수 공식

---

# 장의 결론 메시지

`Agent Ready`는 도구를 설치한 상태가 아니다.

프로젝트가 Agent에게 다음을 제공하는 상태다.

```text
찾을 수 있다.
재현할 수 있다.
실행할 수 있다.
테스트할 수 있다.
완료를 판정할 수 있다.
격리할 수 있다.
병렬화할 수 있다.
관찰할 수 있다.
권한을 제한할 수 있다.
```

그리고 각 주장은 실행 가능한 증거로 확인할 수 있어야 한다.

3장에서 만든 Profile은 이후 장들의 개선 목표이자 22장의 최종 성숙도 평가 입력으로 사용한다.
