# Phase 8 출판 정합성 점검

기준일: 2026-09-16

## 목적

Phase 8에서 완료한 1~18장 최종 교정본을 출판 준비 관점에서 다시 점검한다.

이 문서는 새 구조나 새 개념을 추가하지 않는다. 현재 목차, 장간 참조, 핵심 용어, 설명용 수치, 코드블록/표, 참고자료, 제품 공식 근거, AUTH-142 예제의 일관성을 확인한다.

---

## 1. 점검 대상

- `planning/toc.md`
- `chapters/01/draft.md` ~ `chapters/18/draft.md`
- `planning/phase8-publication-checklist.md`
- `research/chapter-01-cloud-worker-official-sources.md`
- `research/chapter-02-local-cloud-official-sources.md`
- `research/anthropic/agent-native-development-environment.md`
- `research/anthropic/claude-code-web-execution-resources.md`
- `research/anthropic/infrastructure-noise.md`
- `research/github/continuous-ai-runner-first.md`
- `STATUS.md`

---

## 2. 목차와 본문 제목

`planning/toc.md`와 각 `draft.md`의 첫 줄을 직접 대조했다.

결과:

```text
1~18장 제목 일치
18 / 18
```

현재 장 제목:

```text
1. Coding Agent에서 Cloud Worker로
2. Local Agent와 Cloud Agent
3. Cloud Session, Container, Compute와 Token
4. 독립 실행환경, 장시간 작업, 병렬성, 시간 분리
5. Task Routing: Local인가 Cloud인가
6. Cloud에 보내기 좋은 개발 작업
7. Cloud Agent Task Contract: 작은 Task와 작은 Context
8. Tool Output을 줄이고 Evidence를 남기기
9. Prepared Cloud Environment, Cache, Snapshot
10. Cloud Agent를 Test Runner처럼 사용하기
11. Git, Branch, Worktree, Container로 작업 격리하기
12. 병렬 Worker와 중복 Context 비용
13. Local → Cloud → Local Handoff
14. Task Queue와 Event-driven Cloud Agent
15. campus-platform Cloud Agent Workflow 설계
16. 하나의 기능을 Local + Cloud로 끝까지 개발하기
17. Cloud가 항상 정답은 아니다
18. 다음 단계: Harness와 Orchestration
```

---

## 3. 장간 참조

현재 본문은 1~18장 체계를 사용한다.

Phase 8 교정에서 다음 역할 경계를 유지했다.

```text
2장  = 실행 위치 차이
5장  = 시작 Routing
17장 = 실행 중 재Routing / Local Fallback

4장  = 병렬화의 가치
12장 = 병렬화의 비용 / Fan-in

7장  = Task Contract / 입력
8장  = Result Gateway / Evidence / 출력
9장  = Prepared Environment
10장 = Runner-first / Agent-on-failure

13장 = Human-driven Handoff
14장 = Event-driven Handoff
15장 = 정적 운영 모델
16장 = 시간순 End-to-End 사례
18장 = Harness / Orchestration 후속 방향
```

구목차의 19~23장 체계를 현행 본문 참조로 사용하지 않는다.

---

## 4. 핵심 용어

본문 최종 교정에서 다음 용어를 기본값으로 통일했다.

```text
Cloud Agent
Local Agent
Cloud Runner
Cloud Worker
Remote Development Worker
Task
Task Contract
Context
Evidence
Artifact
Result Gateway
Prepared Environment
Runner-first
Agent-on-failure
Handoff
Local Fallback
Human Steering
Developer Blocking Time
Base SHA
Result SHA
Validation Result
Artifact Path / Artifact Reference
PR
Build
Test
Integration Test
E2E
Docker Build
```

특히 작업 추적 관계는 다음 구조로 통일했다.

```text
Task ID
→ Base SHA
→ Branch
→ Result SHA
→ Validation Result
→ Evidence / Artifact
→ PR
```

`Current SHA`, `Result Commit`, `result commit SHA`처럼 같은 의미를 다르게 표현하던 필드는 최종 교정에서 정리했다.

---

## 5. AUTH-142 예제

반복 예제는 `AUTH-142 / expired token`으로 유지한다.

핵심 조건:

```text
Task ID
AUTH-142

Goal
expired token → HTTP 401

Base SHA
작업 시작 기준점

Result SHA
Agent 수정 결과

Validation
AuthServiceTest.expiredToken 또는 auth module test

Evidence
Validation Result + Artifact Reference
```

장마다 모든 필드를 반복하지는 않지만 같은 필드가 등장할 때 의미가 바뀌지 않도록 교정했다.

16장에서는 이 예제를 End-to-End Timeline으로 확장하고, 17장에서는 Fallback 기준을 별도로 다룬다.

---

## 6. 설명용 수치

본문에서 실제 운영 측정값처럼 오해될 수 있는 값은 `설명용 예`임을 표시했다.

대표 예:

```text
Build 20분
Tests 10,000
Log 100MB
Cold Start 7m 40s → 55s
Runner Task 100개
PR 20/day vs Review 5/day
Worker 2 → 8
Retry #1 / #2
16장 Timeline 시각
```

제품별 다음 값은 본문 원칙으로 고정하지 않는다.

```text
vCPU
RAM
Disk
동시 Session 수
Session 유지시간
가격
Rate Limit
```

---

## 7. 코드블록과 표

Phase 8 교정 기준:

```text
개념 흐름 → text
실제 명령 → bash
상태 구조 → yaml
구조화 결과 → json
```

`PASS / FAIL` 상태 표현을 기본으로 유지한다.

표는 작업 분류, Routing, 역할, 측정 항목처럼 비교가 필요한 곳에만 사용하고 본문 설명을 그대로 반복하지 않도록 유지했다.

---

## 8. 제품 공식 근거

### GitHub

2026-09-16 기준 GitHub 공식 문서에서 다음 내용을 재검증했다.

- Copilot cloud agent는 자체 ephemeral development environment에서 작업한다.
- Repository를 탐색하고 코드를 변경하며 automated tests와 linters를 실행할 수 있다.
- development environment에 tools/dependencies를 사전 구성할 수 있다.
- Issue, PR, schedule/event 기반 automation과 연결할 수 있다.
- GitHub는 local sandbox와 cloud sandbox를 별도 실행 선택지로 제공한다.

Research:

- `research/chapter-01-cloud-worker-official-sources.md`
- `research/chapter-02-local-cloud-official-sources.md`

두 문서에 현재 공식 URL과 `공식 자료 재검증: 2026-09-16`을 기록했다.

### OpenAI Codex

공식 자료에서 다음 내용을 재검증했다.

- Codex는 cloud-based coding agent로 소개됐다.
- 각 agent가 cloud container에서 code와 development environment를 사용해 작업할 수 있다.
- 파일 편집과 tests, linters, type checkers 실행이 가능하다고 설명한다.
- 결과를 검토하거나 GitHub PR로 연결할 수 있다.

Research의 공식 근거:

- OpenAI, `Addendum to OpenAI o3 and o4-mini system card: Codex`, 2025-05-16
- OpenAI, `Codex is now generally available`, 2025-10-06

### Anthropic

2026-09-16 기준 Anthropic Engineering 자료를 재검증했다.

- `Quantifying infrastructure noise in agentic coding evals`, 2026-02-05
- `Scaling Managed Agents: Decoupling the brain from the hands`, 2026-04-08
- `Building a C compiler with a team of parallel Claudes`, 2026-02-05

본문에서는 특정 실험 수치를 일반 성능 기대값으로 사용하지 않고 다음 원칙만 사용한다.

```text
실행 자원은 Agent 작업 조건의 일부다.
Brain / Hands를 분리해서 볼 수 있다.
불필요한 Tool Output은 Context에 넣지 않는다.
독립 Task만 병렬화한다.
```

---

## 9. 내부 참고자료 경로 점검

점검 중 3장 참고자료에 다음 경로가 있었으나 파일이 존재하지 않았다.

```text
research/anthropic/infrastructure-noise.md
```

Anthropic 공식 원문과 기존 조사 문서를 기준으로 해당 Research 파일을 새로 작성해 경로를 복구했다.

현재 Anthropic Research 경로:

```text
research/anthropic/agent-native-development-environment.md
research/anthropic/claude-code-web-execution-resources.md
research/anthropic/infrastructure-noise.md
```

---

## 10. 범위 이탈 점검

현재 책의 중심 질문은 유지된다.

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

18장은 다음 주제를 후속 범위로만 남기고 현재 책의 본문으로 확장하지 않는다.

```text
Agent Memory Architecture
Agent Security Platform
Agent OS
Agent Chaos Engineering
Shadow / Canary Agent
Agent Governance Platform
General Multi-Agent Theory
Agent 조직론
```

현재 책은 `Task Routing → 실행환경 → Runner/Agent → Evidence → Handoff`에 집중한다.

---

## 11. 최종 출판 상태

Phase 8 기준 완료된 작업:

```text
1~18장 문장 최종 교정
목차와 본문 제목 대조
장간 역할/참조 점검
핵심 용어 통일
Base SHA / Result SHA / Evidence 추적 필드 통일
설명용 수치 표시
코드블록/표 형식 점검
제품 공식 근거 재검증
제품 Research URL/기준일 보강
내부 참고자료 깨진 경로 복구
AUTH-142 예제 일관성 점검
Agent Platform 범위 확장 방지
```

Phase 8의 본문/정합성 작업은 완료 상태로 본다.

다음 단계는 새 내용 작성이 아니라 출판 산출물 준비다.

```text
최종 원고 묶음
→ 편집 포맷 결정
→ 표지/서문/저자 소개 등 출판 요소 분리 작성
→ PDF / EPUB / 인쇄 원고 변환
→ 최종 교정쇄 확인
```
