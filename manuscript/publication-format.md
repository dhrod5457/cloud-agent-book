# 출판 포맷과 교정쇄 생성 기준

이 문서는 완성된 Markdown 원고를 PDF, EPUB, 인쇄 원고로 변환할 때 사용하는 기준을 정의한다.

## 1. 원본은 Markdown으로 유지한다

출판 원고의 Source of Truth는 Repository의 Markdown 파일이다.

```text
Markdown Source
→ 교정쇄 PDF
→ EPUB
→ 인쇄용 PDF
```

PDF나 EPUB에서 직접 본문을 수정하지 않는다. 오탈자나 사실 오류를 발견하면 원본 Markdown을 수정한 뒤 다시 생성한다.

최종 결합 순서는 `manuscript/assembly-manifest.md`를 따른다.

## 2. 첫 번째 산출물은 교정쇄 PDF다

가장 먼저 하나의 PDF 교정쇄를 만든다.

목적:

- 장과 Part 순서 확인
- 제목 계층 확인
- 한글과 영문 혼합 문장 확인
- 코드블록 줄바꿈 확인
- 표 잘림 확인
- 페이지 단위 흐름 확인
- Front Matter / Back Matter 위치 확인

이 단계에서는 인쇄 판형이나 최종 디자인을 확정하지 않는다.

## 3. EPUB은 PDF 교정 후 생성한다

EPUB은 원고 구조가 안정된 뒤 생성한다.

PDF 교정 전에 EPUB을 동시에 수정하면 같은 오탈자나 구조 수정이 여러 산출물에 반복될 수 있다.

순서는 다음과 같다.

```text
Markdown
→ PDF Proof
→ Source 수정
→ PDF 재생성
→ 내용 고정
→ EPUB 생성
```

EPUB에서는 고정 페이지 배치보다 다음을 확인한다.

- Heading 구조
- 목차 링크
- 코드블록
- 표
- 긴 URL
- 모바일 화면에서의 가독성

## 4. 인쇄용 PDF는 판형이 정해진 뒤 만든다

인쇄용 PDF는 다음 조건이 정해진 뒤 생성한다.

```text
판형
여백
본문 Font
Code Font
페이지 번호 위치
머리말 / 꼬리말
도련 여부
출판사 PDF 사양
```

이 값이 정해지기 전에 인쇄용 페이지 수를 고정하지 않는다.

출판사나 POD 서비스의 요구사항이 없다면 먼저 일반 교정쇄 PDF로 내용과 구조를 확정한다.

## 5. 교정은 Source에 역반영한다

교정쇄에서 발견할 수 있는 오류:

```text
오탈자
깨진 링크
표 잘림
코드블록 줄바꿈
장간 참조 오류
제목 계층 오류
페이지 전환 위치 문제
```

수정 경로:

```text
PDF에서 오류 발견
→ 원본 Markdown 위치 확인
→ Markdown 수정
→ PDF 재생성
→ 다시 확인
```

PDF 자체를 직접 편집해서 원본과 다른 상태를 만들지 않는다.

## 6. 교정쇄 검사 순서

### 1차: 구조

- Title Page
- Front Matter
- 6개 Part 전환 페이지
- 18개 Chapter
- Appendix
- Glossary
- References

### 2차: 내용 표현

- 제목 / 절 제목 계층
- 표
- 코드블록
- 인용문
- 목록
- URL

### 3차: 기술 서적 특성

- 긴 명령어가 잘리지 않는가
- YAML / JSON 들여쓰기가 유지되는가
- ASCII Diagram이 무너지지 않는가
- `Local → Cloud → Local` 같은 화살표 표현이 깨지지 않는가
- 한글과 영문 고정폭 Font의 크기가 지나치게 다르지 않은가

### 4차: 최종 사실 확인

- 제품 공식 URL
- 기준일
- 장간 참조
- 설명용 수치 표시

## 7. 산출물 우선순위

```text
1. Markdown Source
2. PDF Proof
3. EPUB
4. Print-ready PDF
```

Markdown이 최종 원본이며 나머지는 파생 산출물이다.

## 8. Phase 9와 다음 단계의 경계

Phase 9는 다음 상태에서 끝난다.

- 출판 원고 구성 파일이 모두 준비됨
- 결합 순서가 고정됨
- References / Glossary / Appendix 준비됨
- 산출물 포맷 정책이 고정됨
- 교정쇄 생성 절차가 정의됨

실제 PDF 생성과 시각 검사는 다음 단계에서 수행한다.
