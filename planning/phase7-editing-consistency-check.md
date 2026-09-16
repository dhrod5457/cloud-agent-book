# Phase 7 편집 정합성 점검

## 점검 목적

Phase 7에서 편집한 `chapters/01/draft.md` ~ `chapters/18/draft.md`가 책의 중심 질문과 장별 역할을 유지하는지 확인한다.

점검 기준:

- `planning/concept.md`
- `planning/scope.md`
- `planning/toc.md`
- `planning/cloud-agent-remote-worker-model.md`
- `planning/phase6-draft-consistency-check.md`
- `planning/phase7-editing-plan.md`
- 각 장의 `plan.md`

Phase 7의 목표는 새 내용을 추가하는 것이 아니라 초고의 중복을 줄이고 각 장의 역할을 선명하게 만드는 것이었다.

---

# 1. 결과

Phase 7 편집 완료.

```text
1~18장
초고 존재
+
장별 편집 완료
+
전체 역할 경계 유지
```

책의 중심 질문은 변하지 않았다.

> 클라우드 코딩 에이전트를 실제 개발에서 어떻게 더 빠르고, 저렴하고, 효율적으로 사용할 것인가?

독자가 마지막에 내려야 할 판단도 유지된다.

> 이 Task는 Local에서 해야 하는가, Cloud로 보내야 하는가?

---

# 2. 최종 장별 역할

## 1~4장 - Cloud Agent를 이해하는 기준

```text
1장
Cloud Agent = Remote Development Worker

2장
Local / Cloud / Hybrid 실행 위치 차이

3장
Compute / LLM / Human Cost 분리

4장
독립 실행환경 / 비동기 / Developer Blocking Time / 병렬 실행 가치
```

4장에서 병렬화의 가치까지만 설명하고 중복 Context, Merge, Review, Fan-in 비용은 12장으로 이동했다.

## 5~6장 - Task를 어디에 둘 것인가

```text
5장
Task Routing Framework

6장
실제 개발 작업 Catalog
```

5장은 판단 기준, 6장은 Build/Test/E2E/Bug Fix 등의 적용 사례에 집중한다.

17장은 5장을 반복하지 않고 실행 중 재Routing과 Local Fallback을 담당한다.

## 7~10장 - 작은 입력, 작은 출력, 준비된 실행, Runner-first

```text
7장
Task Contract / Small Input

8장
Result Gateway / Evidence / Small Output

9장
Prepared Environment / Cache / Snapshot

10장
Runner-first / Agent-on-failure
```

각 장의 역할을 분리했다.

- 7장: 입력 경계
- 8장: 결과 경계
- 9장: 시작 비용
- 10장: 실행 주체 선택

## 11~12장 - 격리와 병렬화

```text
11장
Source / Runtime / Evidence Isolation

12장
Independent Task Fan-out / Fan-in Cost
```

11장은 안전하게 나눌 조건, 12장은 실제 병렬화 이득과 비용을 다룬다.

## 13~14장 - Handoff의 두 형태

```text
13장
Human-driven Local → Cloud → Local Handoff

14장
Event-driven Task Candidate → Queue → Execution
```

13장은 Git + Task Contract를 입력 경계, Evidence + Commit/PR을 반환 경계로 사용한다.

14장은 CI/Review/Nightly/Dependency Update를 서로 다른 특수 Workflow가 아니라 동일한 Event-driven 경로로 통합한다.

## 15~16장 - 운영 모델과 실제 기능 Timeline

```text
15장
campus-platform 정적 운영 모델

16장
하나의 기능을 Requirement → Merge까지 시간순 실행
```

15장은 구성 요소와 역할 관계를 보여주고, 16장은 같은 구조를 실제 기능 하나에 적용한다.

16장에서 Task Contract, Result Gateway, Prepared Environment의 정의를 반복하지 않고 실제 단계의 입력과 Evidence만 사용하도록 압축했다.

## 17장 - 역방향 Routing

```text
Cloud Task를 언제 중단하는가?
언제 Local / Hybrid로 되돌리는가?
Fallback 시 무엇을 반환하는가?
```

5장이 시작 Routing이라면 17장은 실행 중 재Routing이다.

## 18장 - 다음 단계와 책의 결론

```text
Harness
→ 반복 Routing 자동화
→ Orchestration
```

Agent Platform 일반론으로 확장하지 않고 현재 Workflow에서 반복되는 수동 결정을 자동화하는 다음 단계만 소개한다.

---

# 3. 최종 흐름

```text
Cloud Agent 정의
        ↓
Local / Cloud / Hybrid 이해
        ↓
Compute / LLM / Human Cost 분리
        ↓
Cloud의 실행 가치
        ↓
Task Routing
        ↓
실제 작업 Catalog
        ↓
Task Contract / Small Input
        ↓
Result Gateway / Small Output
        ↓
Prepared Environment
        ↓
Runner-first / Agent-on-failure
        ↓
Task Isolation
        ↓
Independent Task Parallelism
        ↓
Human-driven Handoff
        ↓
Event-driven Handoff
        ↓
campus-platform 운영 모델
        ↓
End-to-End 기능 개발
        ↓
Cloud Stop / Local Fallback
        ↓
Harness / Orchestration 미래 방향
```

구조적 충돌 없음.

---

# 4. 반복 압축 결과

Phase 6에서 편집 대상으로 지정했던 주요 중복 구간을 다음처럼 정리했다.

## 2 ↔ 5 ↔ 17

```text
2장 = 실행 위치의 차이
5장 = 시작 시 Routing
17장 = 실행 중 재Routing / Fallback
```

## 3 ↔ 8 ↔ 10

```text
3장 = Compute와 LLM 자원 분리
8장 = Tool Output과 Evidence
10장 = Runner-first 실행 규칙
```

## 4 ↔ 12

```text
4장 = 병렬 실행의 가치
12장 = 병렬화의 비용과 상한
```

## 7 ↔ 16

```text
7장 = Task Contract 정의
16장 = 이미 정의된 Task Contract를 실제 Timeline에서 사용
```

## 13 ↔ 15 ↔ 16

```text
13장 = Handoff Protocol
15장 = 정적 운영 모델
16장 = 실제 기능 Timeline
```

각 구간의 반복 설명을 줄이고 최초 정의 장을 참조하는 구조로 편집했다.

---

# 5. 핵심 용어

다음 표기를 책 전체 기준으로 사용한다.

```text
Cloud Agent
Cloud Runner
Local Agent
Task Contract
Prepared Environment
Result Gateway
Evidence
Artifact
Human Steering
Developer Blocking Time
Local Fallback
```

Cloud Agent의 확장 정의는 유지한다.

> Cloud Agent는 필요할 때 독립된 개발환경을 할당받고, Git을 통해 Task를 받아 비동기적으로 작업하며, 테스트와 Artifact를 포함한 검증 가능한 결과를 반환하는 Remote Development Worker다.

---

# 6. 반복 예제

`AUTH-142 / expired token` 예제는 다음 개념을 연결하는 공통 사례로 유지한다.

```text
Routing
→ Task Contract
→ Failure Summary
→ Agent Fix
→ Runner Revalidation
→ Evidence
→ Handoff
→ End-to-End Timeline
```

편집 후에는 매 장에서 배경을 다시 설명하지 않고 필요한 부분만 사용한다.

---

# 7. 설명용 숫자

Test Count, 실행시간, Retry 횟수 등 책에서 사용하는 숫자는 실제 프로젝트 기준값이 아닌 경우 `설명용 예시`임을 명시하도록 편집했다.

제품별 다음 값은 본문 원칙으로 고정하지 않는다.

```text
CPU / RAM
Disk
동시 Session 수
가격
Rate Limit
제품별 UI
```

변경 가능한 제품 사실은 `research/`에서 기준일과 출처를 관리한다.

---

# 8. 범위 점검

현재 본문은 다음 주제를 독립 본문으로 확장하지 않는다.

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

18장에서는 현재 Cloud Workflow의 반복 판단을 자동화하는 범위에서만 Harness와 Orchestration을 소개한다.

범위 이탈 없음.

---

# 9. Phase 8 최종 교정 대상

Phase 7에서 구조 편집은 끝났다.

다음 단계에서는 새 구조 변경보다 출판용 마감에 집중한다.

```text
1. 문장 단위 교정
2. 장 제목 / 절 제목 / 표기 통일
3. 코드블록과 표의 형식 통일
4. 장간 참조 번호 최종 검사
5. 제품명을 언급한 문장의 공식 출처와 기준일 재확인
6. 참고자료 표기 방식 통일
7. 설명용 숫자와 실제 수치 구분 재확인
8. Part 전환 문장과 책 전체 도입/결론 연결 점검
9. 최종 목차와 본문 제목 일치 확인
10. 출판용 원고 형태 준비
```

Phase 8에서는 Agent Platform 관련 새 주제를 추가하지 않는다.

---

# 판정

```text
Phase 7 편집 완료

18개 장 역할 경계 유지
중복 압축 완료
책의 중심 질문 유지
Agent Platform 범위 확장 없음
Phase 8 최종 교정 진행 가능
```