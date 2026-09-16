# Current Phase

Phase 8 - 최종 교정 / 출판 준비 완료

1~18장 본문 최종 교정과 출판 정합성 점검을 완료했다.

현재 원고는 **본문 작성 → 전체 편집 → 최종 교정 → 출판 정합성 점검**까지 완료된 상태다.

다음 단계는 새 본문 작성이 아니라 출판 산출물 준비다.

# Source of Truth

현재 책의 방향은 다음 문서를 우선한다.

1. `planning/concept.md`
2. `planning/scope.md`
3. `planning/toc.md`
4. `planning/cloud-agent-remote-worker-model.md`
5. `planning/phase5-consistency-check.md`
6. `planning/phase6-draft-consistency-check.md`
7. `planning/phase7-editing-plan.md`
8. `planning/phase7-editing-consistency-check.md`
9. `planning/phase8-publication-checklist.md`
10. `planning/phase8-publication-consistency-check.md`
11. 각 `chapters/NN/plan.md`
12. `planning/future-topics.md`
13. `STATUS.md`

과거 Agent-Native 독립 장 설계와 초기 22~23장 체계는 현행 목차보다 우선하지 않는다.

# Book Direction

책의 중심 주제는 **Cloud Agent 활용**이다.

핵심 질문:

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

최종 독자 판단:

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

# Core Definition

```text
Cloud Agent
= LLM
+ Repository
+ Independent Execution Environment
+ CPU / RAM / Disk
+ Development Tools
```

확장 정의:

> Cloud Agent는 필요할 때 독립된 개발환경을 할당받고, Git을 통해 Task를 받아 비동기적으로 작업하며, Test와 Artifact를 포함한 검증 가능한 결과를 반환하는 Remote Development Worker다.

# Core Messages

- Cloud Agent는 Local Agent를 대체하지 않는다.
- Cloud Agent의 핵심 가치는 더 많은 Token이 아니라 독립 실행환경과 병렬성이다.
- CPU와 RAM 사용량은 LLM Token 사용량과 직접적으로 같은 개념이 아니다.
- CPU에는 일을 많이 시키고, LLM에는 필요한 정보만 보여준다.
- Runner가 할 수 있으면 Runner에게 맡긴다.
- 작은 Task와 작은 Context를 전달한다.
- Agent에게 개발환경을 설치하게 하지 말고 바로 작업 가능한 환경을 제공한다.
- Cloud Agent에게 결과를 요구하지 말고 검증 가능한 결과물을 요구한다.
- 병렬화의 대상은 Agent가 아니라 서로 독립적으로 실행하고 검증할 수 있는 Task다.
- Task는 Local 또는 Cloud에 영구적으로 속하지 않는다. 작업 단계에 따라 실행 위치를 이동할 수 있다.
- 이벤트가 없으면 Agent도 실행하지 않는다.
- Cloud를 쓰지 않는 결정도 올바른 Routing 결과다.
- Local Fallback은 실패가 아니라 Routing의 일부다.

# Current TOC

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

# Phase 5 Result

Phase 5 완료.

- 1~18장 설계 완료
- 장별 역할/중복/용어/범위 정합성 점검 완료
- `planning/phase5-consistency-check.md` 작성

# Phase 6 Result

Phase 6 완료.

- `chapters/01/draft.md` ~ `chapters/18/draft.md` 초고 작성 완료
- 설계 대비 장별 검토 완료
- 전체 초고 정합성 점검 완료
- `planning/phase6-draft-consistency-check.md` 작성

# Phase 7 Result

Phase 7 완료.

- 1~18장 전체 편집/교정 완료
- 중복 설명 압축
- 장별 역할 경계 명확화
- 핵심 용어 표기 정리
- 설명용 숫자와 제품별 변경 가능한 사실의 본문 원칙 분리
- 장간 연결부 정리
- `AUTH-142 / expired token` 반복 예제를 공통 연결 사례로 정리
- Agent Platform 일반론 확장 방지
- `planning/phase7-editing-consistency-check.md` 작성

# Phase 8 Result

Phase 8 완료.

## 본문

- 1~18장 문장 최종 교정 완료
- 제목 / 절 제목 / 핵심 용어 표기 점검
- 코드블록 / 표 형식 점검
- 장간 역할과 참조 점검
- 설명용 수치 표기 점검
- AUTH-142 예제 일관성 점검

## 추적 필드

본문의 기본 추적 필드는 다음으로 통일했다.

```text
Task ID
Base SHA
Result SHA
Validation Result
Artifact Path / Artifact Reference
PR
```

기본 관계:

```text
Task ID
→ Base SHA
→ Branch
→ Result SHA
→ Validation Result
→ Evidence / Artifact
→ PR
```

## 실행 주체

```text
Local / Local Agent
→ 요구사항 / Architecture / Human Steering / Internal Validation / Review

Cloud Runner
→ Build / Test / E2E / Docker / 결정론적 검증

Cloud Agent
→ 재현 가능한 Failure 분석 / 제한된 코드 수정
```

## 출판 정합성 검사

- `planning/toc.md`와 1~18장 제목 직접 대조: **18 / 18 일치**
- 현행 본문에서 구목차 19~23장 체계를 사용하지 않음
- 제품 가변 수치는 본문 원칙과 분리
- GitHub / OpenAI / Anthropic 공식 근거를 2026-09-16 기준 재검증
- 1~2장 Research에 공식 URL과 재검증 기준일 기록
- Anthropic infrastructure 자료의 깨진 내부 경로 발견 및 복구
- `planning/phase8-publication-consistency-check.md` 작성

# Product Research

## GitHub

현재 공식 근거는 다음 Research에서 관리한다.

- `research/chapter-01-cloud-worker-official-sources.md`
- `research/chapter-02-local-cloud-official-sources.md`

2026-09-16 기준 공식 문서 URL을 재확인했다.

## OpenAI Codex

공식 근거:

- `Addendum to OpenAI o3 and o4-mini system card: Codex`, 2025-05-16
- `Codex is now generally available`, 2025-10-06

제품별 변경 가능한 세부사항은 본문의 일반 원칙으로 고정하지 않는다.

## Anthropic

현재 Research:

- `research/anthropic/agent-native-development-environment.md`
- `research/anthropic/claude-code-web-execution-resources.md`
- `research/anthropic/infrastructure-noise.md`

공식 실험 수치는 해당 실험의 결과로만 사용하고 일반 성능 기대값으로 확대하지 않는다.

# Final Edited Flow

```text
1장  Cloud Agent = Remote Development Worker
  ↓
2장  Local / Cloud / Hybrid
  ↓
3장  Compute / LLM / Human Cost
  ↓
4장  독립 실행환경 / 비동기 / 병렬 실행 가치
  ↓
5장  Task Routing
  ↓
6장  개발 작업 Catalog
  ↓
7장  Task Contract / Small Input
  ↓
8장  Result Gateway / Evidence / Small Output
  ↓
9장  Prepared Environment / Startup Cost
  ↓
10장 Runner-first / Agent-on-failure
  ↓
11장 Source / Runtime / Evidence Isolation
  ↓
12장 Independent Task Fan-out / Fan-in Cost
  ↓
13장 Human-driven Handoff
  ↓
14장 Event-driven Handoff
  ↓
15장 campus-platform 운영 모델
  ↓
16장 End-to-End 기능 Timeline
  ↓
17장 Cloud Stop / Local Fallback / 재Routing
  ↓
18장 Harness / Orchestration 미래 방향 + 결론
```

# Publication Readiness

현재 본문 구조와 정합성은 출판 산출물 준비 단계로 이동할 수 있는 상태다.

본문에 새 개념을 추가하기보다 이후 변경은 교정쇄에서 발견되는 오탈자, 사실 오류, 링크 변경처럼 명확한 수정으로 제한한다.

# Next

다음 단계는 출판 산출물 준비다.

1. 1~18장 최종 원고를 하나의 출판 원고 구조로 묶기
2. Part 제목과 장 사이 전환 페이지/문구 정리
3. 서문 / 책 소개 / 독자 대상 / 읽는 방법 작성
4. 표지용 제목 / 부제 / 책 소개 문구 확정
5. 참고자료 / 용어집 / 부록 필요 여부 결정
6. PDF / EPUB / 인쇄 원고 포맷 결정
7. 최종 교정쇄 생성 후 오탈자만 수정
