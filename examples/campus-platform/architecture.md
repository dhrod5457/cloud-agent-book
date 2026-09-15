# campus-platform Architecture

## 설계 목표

`campus-platform`은 처음부터 복잡한 분산 시스템으로 설계하지 않는다.

초기에는 하나의 Spring Boot 애플리케이션으로 시작하고, 책의 진행에 따라 모듈 경계와 외부 Adapter를 명확히 하면서 Agent가 안전하게 변경하기 쉬운 구조로 발전시킨다.

핵심 목표는 다음과 같다.

- 변경 범위를 쉽게 판단할 수 있을 것
- 외부 시스템과 핵심 도메인을 분리할 것
- 여러 Agent가 서로 다른 영역을 동시에 수정할 수 있을 것
- 테스트와 검증이 특정 개발자 PC에 의존하지 않을 것
- 실제 운영 인프라가 없어도 대부분의 기능을 검증할 수 있을 것

## 도메인 경계

### auth

책의 보안·권한 예제를 위한 최소 인증 경계를 제공한다.

책의 중심 도메인은 아니므로 OAuth/OIDC 전체 구현을 다루지 않는다.

책에서 사용하는 책임:

- 요청 사용자 식별
- 역할 확인
- 인증된 사용자 컨텍스트 제공

### student

학생의 기본 정보와 수강 관계를 관리한다.

책에서 사용하는 책임:

- 학생 조회
- 학생 상태 조회
- 수강 정보 조회
- 외부 학사 데이터 반영

### attendance

학생과 수업 정보를 참조하여 출결을 기록한다.

책에서 사용하는 책임:

- 출결 등록
- 출결 상태 조회
- 출결 정책 검증
- 출결 이벤트 발행

### notification

도메인에서 발생한 이벤트를 사용자 알림으로 변환한다.

책에서 사용하는 책임:

- 알림 요청 생성
- 발송 상태 관리
- 외부 Push/Message Adapter 호출

### integration

내부 도메인이 외부 시스템 세부 구현에 직접 의존하지 않도록 경계를 제공한다.

책에서 사용하는 책임:

- 외부 학사 API
- 외부 DB
- Push Provider
- HSM
- 기타 사내 시스템 Adapter

## 최종 목표 모듈 구조

초기 Stage에서는 아래 구조를 모두 구현하지 않는다. 책의 진행에 따라 점진적으로 도달한다.

```text
campus-platform/
├─ app/
├─ modules/
│  ├─ auth/
│  ├─ student/
│  ├─ attendance/
│  ├─ notification/
│  └─ integration/
├─ common/
├─ scripts/
├─ docs/
└─ infra/
```

### app

애플리케이션 실행과 조립을 담당한다.

허용 책임:

- Spring Boot entry point
- module wiring
- runtime configuration

금지 책임:

- 핵심 도메인 로직
- 특정 외부 시스템 구현

### modules/*

업무 기능의 소유 경계다.

각 모듈은 가능하면 다음 내부 구조를 가진다.

```text
module/
├─ domain/
├─ application/
├─ adapter/
│  ├─ in/
│  └─ out/
└─ config/
```

모든 장에서 이 구조를 기계적으로 강제하지는 않는다. 핵심은 각 변경의 소유권과 의존성 방향을 명확히 하는 것이다.

### common

최소화한다.

허용 후보:

- 공통 오류 모델
- 공통 테스트 유틸리티
- 시간/ID와 같이 명확한 기술 공통 요소

금지 후보:

- 여러 도메인의 업무 로직
- 편의를 위해 이동한 잡다한 Util
- 여러 Agent가 자주 동시에 수정하는 중앙 파일

`common`의 크기가 커지는 것은 병렬 작업 충돌 위험 신호로 본다.

## 의존성 방향

기본 규칙은 다음과 같다.

```text
adapter -> application -> domain
```

외부 기술은 안쪽 계층이 직접 알지 않도록 한다.

예:

```text
attendance domain
      ↑
attendance application
      ↑
web / mybatis / kafka adapter
```

모듈 간 참조는 공개된 application API 또는 명시적인 contract를 통해 이루어지게 한다.

예를 들어 `attendance`가 `student`의 Mapper나 DB 테이블에 직접 접근하는 것은 금지한다.

## 모듈 간 통신

초기에는 동일 프로세스 안의 직접 호출을 기본으로 한다.

필요한 경우에만 이벤트를 도입한다.

예:

```text
attendance
  └─ AttendanceRecorded
           ↓
      notification
```

Kafka는 분산 시스템을 흉내 내기 위해 무조건 사용하는 기술이 아니라, 이벤트 기반 경계와 테스트 전략을 설명할 필요가 있는 장에서 사용한다.

## 외부 의존성

### PostgreSQL

기본 로컬/CI 데이터베이스다.

- 학생
- 수강
- 출결
- 알림 상태

데이터를 저장한다.

### Redis

캐시와 임시 상태를 설명하는 예제로 사용한다.

핵심 기능이 Redis 없이는 전혀 테스트되지 않는 구조는 피한다.

### Kafka

출결 이벤트와 알림 처리의 비동기 흐름을 설명하는 데 사용한다.

### 외부 학사 시스템

HTTP API 또는 DB Adapter로 추상화한다.

기본 테스트에서는 Mock/Fake를 사용한다.

### HSM

실제 HSM SDK는 기본 예제 환경에 포함하지 않는다.

```text
HsmPort
  ├─ FakeHsmAdapter
  └─ EnterpriseHsmAdapter
```

형태로 경계를 둔다.

### Jenkins

기본 CI 구현은 GitHub Actions로 설명하고, Jenkins는 내부망 기업 환경에서 Local/Hybrid Agent의 검증과 배포 예제로 사용한다.

## 데이터 소유권

모듈별 데이터 소유권을 명확히 한다.

- student: 학생, 수강
- attendance: 출결
- notification: 알림과 발송 상태

다른 모듈의 테이블을 직접 수정하지 않는다.

초기 단일 DB에서는 물리적으로 같은 PostgreSQL을 사용해도 논리적 소유권은 분리한다.

## Migration 소유권

DB Migration은 병렬 Agent 충돌이 자주 발생할 수 있는 공유 자원이다.

원칙:

- migration 파일은 생성 주체를 명확히 한다.
- 이미 적용된 migration을 수정하지 않는다.
- 파일명 충돌을 피할 규칙을 둔다.
- 서로 다른 Agent가 동일 테이블을 동시에 변경해야 한다면 PM/Integrator가 순서를 결정한다.

구체적인 규칙은 병렬 Agent 장에서 확정한다.

## Agent 작업 경계

이 구조의 중요한 목적 중 하나는 작업 범위와 파일 범위를 연결하는 것이다.

예:

```text
Task: 출결 사유 필드 추가
Primary owner: attendance
Allowed change:
- modules/attendance/**
- attendance-owned migration
- attendance tests

Potential dependency:
- API contract

Forbidden by default:
- modules/student/internal/**
- modules/notification/internal/**
```

Task Contract가 모듈 구조와 직접 연결되게 한다.

## 아키텍처 검증 대상

향후 ArchUnit 또는 동등한 도구로 다음 규칙을 검증한다.

- domain은 adapter에 의존하지 않는다.
- 업무 모듈은 다른 모듈의 adapter 구현을 참조하지 않는다.
- integration 구현이 핵심 도메인으로 침투하지 않는다.
- common이 도메인 모듈에 역으로 의존하지 않는다.

## 일반 원칙과 구현 예의 경계

책에서 먼저 설명할 것은 다음 원칙이다.

- 변경의 소유권
- 의존성 방향
- 외부 의존성 격리
- 병렬 변경 충돌 감소
- 자동 검증 가능성

그 이후에 Java/Spring Boot, MyBatis, ArchUnit 구조를 구현 예로 제시한다.

Java 패키지 구조 자체를 Agent Ready의 정답으로 제시하지 않는다.
