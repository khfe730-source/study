# 프로젝트 지침

같이 공부하기 위한 프로젝트

## 파일 구조 규칙

- 책(book) 별로 디렉토리가 존재한다
- 해당 디렉토리 하위에 `docs` 디렉토리를 생성 후 그 안에 정리 파일을 생성한다
- 책 성격에 따라 챕터 단위 또는 아이템 단위로 `.md` 파일을 분리한다
  - 챕터 단위: `chapterXX_{챕터제목}.md` — 예) `chapter01_Introduction.md`
  - 아이템 단위: `itemXX_{아이템제목}.md` — 예) `item01_UsePropertiesInsteadOfAccessibleDataMembers.md`
  - 기존 예외: 게임프로그래밍패턴은 `03_command.md` 형식 유지

## 작성 규칙

- 프로그래머 전문가가 공부할 수준으로 작성한다 (기초 설명 생략)
- 한국어로 작성하되, 전문 용어는 괄호 안에 영어 병기 가능 예: 캐시 라인(cache line)
- 필요한 경우 그림(ASCII 다이어그램 등)도 첨부한다
- 핵심 코드 예제를 포함한다

## README.md 자동 업데이트 규칙

새 아이템/챕터 파일을 추가하거나 기존 파일을 수정할 때, **별도 요청 없이 아래 두 README를 자동으로 업데이트**한다.

### 1. 책 단위 README (`{book}/README.md`)
- 위치: `docs`와 같은 수준의 디렉토리 (예: `E:\book\EffectiveC#\README.md`)
- 새 아이템/챕터가 추가되면 해당 항목을 테이블에 추가한다
- 링크 형식: `./docs/chapterXX_{제목}.md` 또는 `./docs/itemXX_{제목}.md`

### 2. 루트 README (`E:\book\README.md`)
- 새 책 디렉토리가 추가되면 목록 테이블에 해당 책을 추가한다
- 기존 책의 아이템/챕터 수가 변경되면 해당 행을 업데이트한다
