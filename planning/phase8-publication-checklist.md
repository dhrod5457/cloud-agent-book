# Phase 8 최종 교정 / 출판 준비 기준

## 목적

Phase 8은 1~18장 편집본을 출판 가능한 원고 형태로 마감하는 단계다.

새 장이나 새 개념을 추가하지 않는다. Phase 7에서 확정한 장별 역할과 흐름을 유지하면서 문장, 표기, 코드블록, 장간 참조, 참고자료를 최종 정리한다.

---

## 작업 순서

```text
문장 단위 교정
→ 제목 / 절 제목 / 용어 표기 통일
→ 코드블록 / 표 형식 통일
→ 장간 참조 검사
→ 설명용 수치 표기 검사
→ 제품 사례 / 공식 출처 검사
→ 참고자료 형식 통일
→ 도입 / 결론 연결 검사
→ 최종 목차와 본문 제목 대조
→ 출판용 원고 묶음 준비
```

---

## 1. 문장 교정

다음 기준으로 수정한다.

- 한 문단에 하나의 판단을 둔다.
- 같은 의미를 바로 다음 문장에서 반복하지 않는다.
- 주어가 불명확한 문장을 줄인다.
- 긴 문장은 의미 단위로 나눈다.
- 형용사보다 조건, 입력, 출력, 수치, 실행 결과를 사용한다.
- 새로운 주장이나 사례를 추가하지 않는다.
- 과장 표현과 AI 상투어를 사용하지 않는다.

책의 기존 문체인 짧은 설명 문장과 `text` 흐름도를 유지한다.

---

## 2. 핵심 용어 표기

다음 표기를 기본값으로 사용한다.

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
PR
Build
Test
Integration Test
E2E
Docker Build
```

원칙:

- 같은 개념을 장마다 다른 한글 번역으로 바꾸지 않는다.
- 제품명과 기술 고유명사는 공식 표기를 따른다.
- `Repository`, `Workspace`, `Branch`, `Commit`, `Runtime`, `Environment`는 기술 개념을 가리킬 때 기존 영문 표기를 유지한다.
- 일반 문장 안에서 불필요하게 영문 대문자를 늘리지 않는다.
- 코드, 파일명, 명령, 상태값은 백틱 또는 코드블록으로 구분한다.

---

## 3. 설명용 수치

책의 설명을 위해 만든 수치는 실제 운영 측정값처럼 보이지 않게 한다.

예:

```text
설명용 예
Tests: 10,000
Failures: 3
Log: 100MB
```

또는 문장에 `설명용 예`라고 명시한다.

다음 값은 출처 없이 실제 제품 사실처럼 고정하지 않는다.

- vCPU
- RAM
- Disk
- 동시 Session 수
- Session 유지시간
- 가격
- Rate Limit
- 제품별 실행시간

---

## 4. 코드블록과 표

- 흐름과 개념 구조는 `text` 코드블록을 사용한다.
- 실제 실행 명령은 `bash`를 사용한다.
- JSON/YAML 예시는 각각 `json`, `yaml`을 사용한다.
- 동일 종류의 상태 표기는 `PASS / FAIL`을 기본으로 한다.
- 표의 열 제목은 짧게 유지한다.
- 본문에서 이미 설명한 내용을 표 아래에서 다시 장문으로 반복하지 않는다.

---

## 5. 장간 참조

현재 목차 번호만 사용한다.

주요 경계:

```text
2장 = 실행 위치 차이
5장 = 시작 Routing
17장 = 실행 중 재Routing

4장 = 병렬화 가치
12장 = 병렬화 비용

7장 = Task Contract 정의
8장 = Result Gateway / Evidence
9장 = Prepared Environment
10장 = Runner-first

13장 = Handoff Protocol
14장 = Event-driven Task
15장 = 정적 운영 모델
16장 = 시간순 End-to-End 사례
18장 = 후속 자동화 방향
```

참조 문장은 해당 장의 역할을 침범하지 않는 수준으로 유지한다.

---

## 6. 제품 사례와 출처

제품별 기능이나 현재 동작을 직접 언급하는 문장은 공식 자료를 우선한다.

본문에서는 가능한 한 변하지 않는 공통 원칙을 유지한다.

변경 가능성이 큰 사실은 `research/` 문서에 기준일과 출처를 둔다.

출처 검사 대상:

- GitHub Copilot cloud agent 관련 문장
- OpenAI Codex 관련 문장
- Anthropic Claude Code Web 관련 문장
- 제품별 CPU / RAM / Session / 가격 / 제한

본문에 제품별 수치를 새로 추가하지 않는다.

---

## 7. 참고자료 형식

장 말미 참고자료는 다음 원칙을 사용한다.

- 본문에 직접 사용한 자료만 남긴다.
- 내부 조사 문서는 Repository 상대 경로로 표기한다.
- 외부 자료는 조직/문서명 중심으로 표기한다.
- 변경 가능한 제품 정보는 조사 문서에서 기준일과 URL을 관리한다.
- 같은 출처를 장마다 불필요하게 반복하지 않는다.

---

## 8. 최종 검사

전체 교정이 끝나면 다음을 확인한다.

```text
1~18장 제목 == planning/toc.md
장간 참조 번호 유효
핵심 용어 표기 일관
설명용 수치 표시
제품별 가변 사실 분리
코드블록 언어 표기
표 형식
참고자료 형식
AUTH-142 예제 일관성
Agent Platform 범위 확장 없음
```

---

## 완료 조건

Phase 8 완료 조건은 다음과 같다.

- 1~18장 최종 문장 교정 완료
- 장 제목과 목차 일치 확인
- 장간 참조 점검 완료
- 용어/코드블록/표 형식 점검 완료
- 제품 관련 가변 사실의 공식 출처 및 기준일 점검 완료
- 참고자료 형식 통일
- 최종 정합성 점검 문서 작성
- 출판용 원고 묶음 기준 확정
