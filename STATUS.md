# Current Phase

Phase 5 - 장별 설계 진행 중

# Completed

- Phase 1 방향 정의
- Phase 2 전체 목차 설계
- Phase 3 목차 검증
- Phase 4 `campus-platform` 예제 프로젝트 설계
- Phase 5 1장 설계
- Phase 5 2장 설계 및 Cloud Runner 패턴 보강
- Cloud compute resource와 LLM usage 분리
- Cloud Runner / Agent Worker 역할 분리
- Runner-first / Agent-on-exception 구조 추가
- Result Filter 설계 추가
- Event-driven Agent / Nightly Agent 사례 추가
- Fan-out 병렬 실행 패턴 추가
- UI/E2E 반복 검증 사례 추가
- Agent Harness 개념 추가
- GitHub Continuous AI / token efficiency 사례 조사
- Claude Code Web PR auto-fix 사례 조사

# Phase 4 Artifacts

- `examples/campus-platform/README.md`
- `examples/campus-platform/architecture.md`
- `examples/campus-platform/testing.md`
- `examples/campus-platform/evolution.md`
- `examples/campus-platform/cloud-test-runner.md`

# Phase 5 Artifacts

- `chapters/01/plan.md`
- `chapters/02/plan.md`
- `research/anthropic/claude-code-web-execution-resources.md`
- `research/github/continuous-ai-runner-first.md`

# Decisions

## 책의 기본 방향

- 책의 상위 개념은 `Agent Ready Software Engineering`으로 정의한다.
- `Cloud-Agent Ready`는 하위 개념으로 다룬다.
- Java/Spring Boot는 주요 실전 예제이지만 책의 원칙은 언어와 제품에 종속되지 않는다.
- 특정 제품의 사용법보다 프로젝트 구조와 개발 프로세스 설계를 중심으로 다룬다.
- 프로젝트 파일을 Source of Truth로 사용한다.

## Cloud Agent 핵심 원칙

2장 `Local Agent, Cloud Agent, Hybrid Agent`에서는 다음 세 문장을 핵심 원칙으로 사용한다.

> LLM은 판단하고, 컨테이너는 실행한다.

> CPU에는 일을 많이 시키고, LLM에는 필요한 결과만 보여준다.

> 정상 경로는 Runner가 처리하고, 예외 경로에서만 Agent를 호출한다.

클라우드 에이전트 시스템의 목표를 다음과 같이 정의한다.

> AI를 최대한 많이 사용하는 시스템이 아니라, AI가 반드시 필요한 순간에만 호출되는 시스템.

## Runner-first / Agent-on-exception

기본 실행 구조:

```text
Task Scheduler
      ↓
Cloud Runner
      ↓
Build / Test / Lint / Validation
      ↓
 +----+----+
 |         |
PASS      FAIL
 |         |
종료   Result Filter
           ↓
      Agent Worker
           ↓
         수정
           ↓
      Runner 재검증
           ↓
          PR
```

Cloud Runner는 build/test/E2E/Docker/lint/static analysis/migration/security scan처럼 결정론적으로 실행 가능한 작업을 담당한다.

Agent Worker는 코드 분석, 실패 원인 판단, 설계 결정, 코드 수정처럼 추론이 필요한 작업을 담당한다.

## Result Filter

Result Filter는 기본적으로 LLM이 아니라 일반 프로그램 또는 스크립트로 구현한다.

수집 항목:

- exit code
- 성공/실패 건수
- 실패 테스트명
- root cause 후보
- 핵심 stack trace
- error/warning count
- artifact path
- raw log path

원본 로그는 artifact로 보존하고 Agent에는 기본적으로 축약 결과만 전달한다.

## Event-driven Agent

Agent를 상시 프로세스로 두지 않는다.

Agent 활성화 후보:

- Runner failure
- PR CI failure
- review comment
- security alert
- scheduled regression failure
- human escalation

Nightly 또는 repository event에서 deterministic Runner가 PASS하면 Agent를 호출하지 않는다.

## Fan-out

독립 작업은 여러 Runner로 병렬 실행할 수 있다.

Fan-out의 목적은 Agent 수 증가가 아니라 독립적인 CPU/RAM 작업의 병렬화다.

각 Worker/Runner가 Repository 전체를 반복 분석하지 않도록 module/path, command, acceptance criteria와 관련 Context를 제한한다.

## Agent Harness

Agent Harness는 Agent가 프로젝트를 이해하고 실행하고 검증할 수 있도록 Repository가 제공하는 지원 구조다.

포함 후보:

- build/test/lint 명령
- validation script
- architecture 문서
- coding convention
- Agent Contract
- 작은 Task 단위
- result parser
- machine-readable report
- PR template
- artifact 규칙

Agent 실패 때마다 프롬프트를 길게 만드는 대신 실행 환경과 검증 인터페이스를 개선한다.

## 공개 사례 사용 원칙

### GitHub Continuous AI

공개된 Continuous test improvement 실험:

- 약 45일
- 1,400개 이상 테스트
- coverage 약 5% → near 100%
- 공개된 당시 token 비용 약 $80
- 작은 PR을 지속적으로 생성

이 수치는 사례로만 사용하며 일반 비용 예측값으로 사용하지 않는다.

GitHub의 2026년 token efficiency 사례에서는 deterministic data gathering을 LLM loop 밖으로 이동하고 relevance gate를 통해 불필요한 LLM 호출을 제거한 사례를 확인했다.

관련 조사:

- `research/github/continuous-ai-runner-first.md`

### Claude Code Web

Claude Code Web은 격리 Cloud Session, 병렬 작업, PR auto-fix의 제품 사례로만 사용한다.

2026-09-16 기준 공식 문서의 대략적 Cloud Session resource limit:

- 4 vCPU
- 16 GB RAM
- 30 GB disk

현재 수치는 변경 가능한 제품 사양이므로 본문의 핵심 논리와 분리한다.

PR auto-fix에서 CI failure와 review comment가 Agent 활성화 이벤트가 될 수 있다는 점을 Event-driven Agent 사례로 사용한다.

관련 조사:

- `research/anthropic/claude-code-web-execution-resources.md`

# In Progress

Phase 5 장별 설계.

현재 1장과 2장의 `plan.md`가 작성되었고 2장의 Cloud Agent 관련 설계를 보강한 상태다.

# Next

Phase 5를 계속 진행한다.

다음 대상:

`chapters/03/plan.md` - Agent Ready 프로젝트의 기준

3장에서는 2장에서 정의한 Runner/Agent 패턴을 반복 설명하지 않고, 프로젝트가 Agent Ready인지 판단할 수 있는 평가 기준으로 연결한다.

장 설계가 확정되기 전에는 Phase 6 본문 초고를 시작하지 않는다.

# Open Questions

Phase 5 이후 실제 구현과 집필 과정에서 검증한다.

- MyBatis 예제가 특정 독자층에 지나치게 종속되지 않는지
- Redis/Kafka가 모든 장에 불필요한 복잡도를 만들지 않는지
- Stage별 Git tag가 독자의 실습 흐름에 가장 적절한지
- 기업 환경 장에서 Tibero/HSM/Jenkins 사례의 깊이를 어느 수준까지 둘지
- Result Filter를 shell 기반 helper로 시작할지 별도 도구로 만들지
- Test Runner 결과 포맷을 JSON Schema로 고정할지
- `Runner-first / Agent-on-exception` 용어를 최종 용어로 유지할지
- Agent Harness를 독립 용어로 정의할지 기존 Agent Contract/실행 인터페이스의 상위 개념으로 둘지
