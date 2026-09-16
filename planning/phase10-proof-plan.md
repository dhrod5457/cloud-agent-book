# Phase 10 - 교정쇄 생성 / 시각 검수 계획

기준일: 2026-09-16

Phase 9에서 출판 원고의 Front Matter, 6개 Part, 18개 Chapter, Back Matter, 최종 조립 순서와 포맷 정책을 확정했다.

Phase 10의 목적은 구조가 확정된 Markdown Source를 실제 교정쇄로 만들고 페이지 단위 문제를 찾는 것이다.

## 1. 작업 원칙

- 새 본문을 추가하지 않는다.
- `manuscript/assembly-manifest.md` 순서를 그대로 사용한다.
- PDF에서 직접 본문을 수정하지 않는다.
- 모든 교정 결과는 원본 Markdown에 역반영한다.
- 제품 기능이 바뀐 경우에만 References와 관련 본문 사실을 다시 검증한다.
- 디자인보다 먼저 내용 누락, 잘림, 코드블록, 표, 제목 계층을 확인한다.

## 2. 1차 작업 - Master 원고 생성

32개 출판 대상 파일을 순서대로 결합해 다음 파일을 만든다.

```text
manuscript/master.md
```

Source of Truth:

```text
manuscript/assembly-manifest.md
```

Master 원고는 파생 파일이다. 개별 장과 Front / Back Matter 파일이 원본이며, 수정 후 Master를 다시 생성한다.

## 3. 2차 작업 - 구조 검사

Master 원고에서 다음 순서를 확인한다.

```text
Title Page
→ Preface
→ About This Book
→ How to Read
→ Table of Contents
→ Part I ~ VI
→ Chapters 1 ~ 18
→ Appendix A
→ Glossary
→ References
```

확인 항목:

- Chapter 18개 존재
- Part 6개 존재
- Front Matter 누락 없음
- Back Matter 누락 없음
- 장 제목 중복 없음
- 구목차 19~23장 잔존 없음

## 4. 3차 작업 - PDF 교정쇄 생성

첫 번째 실제 출판 산출물은 PDF Proof다.

PDF Proof에서는 최종 인쇄 디자인보다 다음을 먼저 확인한다.

```text
한글 글리프
영문 / 한글 혼합
제목 계층
코드블록
표
ASCII Diagram
긴 URL
페이지 분리
Part 전환
```

PDF 생성 방식은 원본 Markdown을 변경하지 않는 변환 파이프라인을 사용한다.

## 5. 4차 작업 - 렌더링 검수

PDF를 페이지 이미지로 렌더링해 시각적으로 검사한다.

필수 확인:

- 잘린 문장 없음
- 겹친 텍스트 없음
- 검은 사각형 / 깨진 글리프 없음
- 표 잘림 없음
- 코드블록이 페이지 밖으로 나가지 않음
- YAML / JSON 들여쓰기 유지
- `Local → Cloud → Local` 같은 화살표 표현 유지
- Part 전환 페이지 위치 확인

## 6. 교정 이슈 기록

첫 번째 교정쇄에서 발견한 문제는 다음 파일에 기록한다.

```text
review/proof-01.md
```

각 이슈는 최소한 다음 정보를 가진다.

```text
Location
Type
Current
Expected
Source File
Status
```

예:

```text
Location: Chapter 7 / Task Contract example
Type: code-block-wrap
Source File: chapters/07/draft.md
Status: open
```

## 7. 수정 경로

```text
PDF에서 문제 발견
→ 원본 Markdown 파일 확인
→ 원본 수정
→ master.md 재생성
→ PDF 재생성
→ 렌더링 재검수
```

`master.md`만 수정하지 않는다.

## 8. 내용 고정 이후

PDF 교정쇄가 안정된 뒤 다음 순서로 진행한다.

```text
PDF Proof 내용 고정
→ EPUB 생성 / 확인
→ 판형 결정
→ Print-ready PDF 생성
```

인쇄용 PDF는 출판사 또는 POD 사양이 정해진 뒤 만든다.

## 9. Phase 10 완료 조건

- `manuscript/master.md`가 Assembly Manifest와 동일한 순서로 생성되어 있다.
- PDF Proof가 생성되어 있다.
- PDF를 렌더링해 시각 검수했다.
- 발견한 문제를 `review/proof-01.md`에 기록했다.
- 교정 결과를 원본 Markdown에 역반영했다.
- 재생성한 PDF에 치명적인 잘림 / 겹침 / 깨진 글리프가 없다.
- 출판용 6 Part / 18 Chapter / Back Matter 순서가 유지된다.

Phase 10에서는 원고 구조를 다시 설계하지 않는다. 실제 페이지에서 드러나는 문제를 수정하는 데 집중한다.
