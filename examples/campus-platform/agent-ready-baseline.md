# campus-platform Agent Ready Baseline

이 문서는 3장 `Agent Ready 프로젝트의 기준`에서 사용할 Stage 0 기준선이다.

목적은 하나의 점수를 만드는 것이 아니라 어떤 조건이 Agent 작업을 막는지 확인하는 것이다.

평가 상태는 다음 세 값만 사용한다.

- `PASS`: 사람의 추가 설명 없이 반복 실행 가능
- `PARTIAL`: 일부 자동화가 있지만 수동 작업이나 숨겨진 환경에 의존
- `FAIL`: 해당 기능이 없거나 Agent가 독립적으로 사용할 수 없음

모든 평가는 증거와 함께 기록한다.

## Baseline

| 기준 | 상태 | 증거 또는 현재 상태 | 주요 장애물 |
| --- | --- | --- | --- |
| Reproducibility | PARTIAL | Gradle/JDK 기반 실행은 가능 | DB 및 일부 환경 준비가 수동 |
| Discoverability | FAIL | 일반적인 Spring Boot package 구조 | 프로젝트 목적, 변경 경계, canonical 문서 부족 |
| Executability | PARTIAL | Gradle 명령 직접 실행 가능 | build/test/verify 표준 진입점 없음 |
| Testability | FAIL | 일부 Unit Test 실행 가능 | 외부 DB 및 통합 환경 의존 |
| Verifiability | FAIL | 개별 테스트 결과 존재 | 전체 작업 완료를 판단하는 통합 verify 없음 |
| Isolation | FAIL | 개발자별 branch 사용 가능 | 공유 DB와 외부 자원 격리 미정의 |
| Parallelizability | PARTIAL | package 단위 기능 구분 존재 | shared configuration, migration, common 변경 충돌 가능 |
| Observability | FAIL | build/test 로그 존재 | task 상태, artifact, retry, Agent 호출 여부 구조화 안 됨 |
| Security Boundary | FAIL | 개발자 권한 정책만 존재 | Agent용 Repository/Secret/Network 권한 경계 미정의 |

## 작업 유형별 현재 가능성

### Local Developer + Agent 보조

`PARTIAL`

개발자가 환경을 준비하고 실행 순서를 알려주면 Agent가 일부 코드를 수정할 수 있다.

### Cloud Test Runner

`FAIL`

주요 이유:

- fresh environment bootstrap 불완전
- 외부 DB 의존
- 표준 verify command 없음
- artifact/result format 없음

### Cloud Agent Worker

`FAIL`

Runner 조건 외에 다음 문제가 있다.

- Repository 탐색 기준 부족
- 변경 금지 영역 미정의
- 완료 조건 미정의
- Agent용 권한 경계 미정의

### Parallel Agent Development

`FAIL`

모듈과 shared file의 ownership이 정리되지 않았고 독립 검증 단위가 부족하다.

## 책 진행에 따른 예상 변화

이 표는 장 집필 과정에서 실제 예제 구조와 함께 갱신한다.

```text
4~5장
Discoverability 개선

6장
작업 범위와 완료 조건 명시

7장
Reproducibility 개선

8장
Executability 개선

9장
Testability / Isolation 개선

10~11장
Verifiability 개선

12~13장
Parallelizability / Isolation 개선

14~15장
Security Boundary 개선

16~17장
Project State / Observability 개선
```

## 평가 원칙

다음과 같이 하나의 기능만 보고 Agent Ready라고 판단하지 않는다.

```text
AGENTS.md 있음
≠ Agent Ready

Dockerfile 있음
≠ Cloud Agent Ready

테스트 많음
≠ Agent가 독립 검증 가능
```

평가의 기준은 실제 작업 흐름이다.

```text
fresh clone
→ setup
→ task
→ test
→ verify
→ result
```

각 단계에서 사람이 개입해야 하는 지점을 찾아 이후 장에서 제거하거나 명시적인 handoff로 전환한다.
