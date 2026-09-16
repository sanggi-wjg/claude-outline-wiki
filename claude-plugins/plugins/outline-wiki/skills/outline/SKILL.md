---
name: outline
description: Outline 위키 문서를 outline CLI로 검색·조회·작성·수정·관리한다. "위키에서 찾아줘", "위키 문서 읽어줘/만들어줘/수정해줘", outline 명령 사용법, 위키 컬렉션·리비전 확인 등 Outline 위키 관련 요청 시 사용.
argument-hint: [위키 작업 요청]
allowed-tools: Bash(outline doctor:*) Bash(outline search:*) Bash(outline read:*) Bash(outline list:*) Bash(outline tree:*) Bash(outline collections:*) Bash(outline revisions:*)
---

# Outline 위키 작업

사내 Outline 위키를 `outline` CLI로 다룬다. 위키 주소는 `OUTLINE_URL` 환경변수로 공급되며, 플러그인이 PATH에 등록한 실행 파일을 `outline <subcommand>` 형태로만 호출한다 — 스크립트 경로를 직접 참조하지 않는다.

## 1. 인자 처리

- `$ARGUMENTS`가 주어지면 그 요청을 아래 규칙대로 수행한다.
- 인자 없이 호출되면 `outline doctor`로 연결·토큰 상태를 확인하고, 아래 CLI 참조의 명령 요약을 보여준 뒤 무엇을 할지 묻는다.

## 2. 위임 기준

- 여러 문서를 읽어 종합해야 하거나 검색 범위가 넓으면 `outline-wiki` 서브에이전트에 위임한다 (격리 컨텍스트에서 검색·발췌하므로 메인 세션 컨텍스트를 아낀다).
- 단일 명령으로 끝나는 소규모 작업(특정 문서 읽기, 컬렉션 목록 확인 등)은 메인 세션에서 직접 실행한다.

## 3. CLI 참조

읽기 계열 (permission allow 대상):

| 명령 | 시그니처 | 설명 |
| :--- | :--- | :--- |
| `doctor` | `outline doctor` | 토큰·계정 확인 (셋업 검증용) |
| `search` | `outline search [--collection COLLECTION] [--limit LIMIT] query` | 문서 검색. `--collection`은 컬렉션 ID로 범위 제한, `--limit` 기본 10 |
| `read` | `outline read <docId\|URL>` | 문서 조회 — 메타 헤더 + 마크다운 본문 |
| `list` | `outline list [--collection COLLECTION] [--limit LIMIT]` | 문서 목록 (updatedAt DESC), `--limit` 기본 25 |
| `tree` | `outline tree <collectionId>` | 컬렉션 문서 트리 (들여쓰기 + id) |
| `collections` | `outline collections` | 컬렉션 목록 (id, 이름) |
| `revisions` | `outline revisions [--limit LIMIT] <docId\|URL>` | 문서 리비전 목록 (최신순), `--limit` 기본 10 |

쓰기 계열 (사용자가 명시적으로 요청한 경우에만 — permission 프롬프트를 거친다):

| 명령 | 시그니처 | 설명 |
| :--- | :--- | :--- |
| `create` | `outline create --title TITLE [--collection COLLECTION] [--parent <docId\|URL>] --body-file FILE [--publish]` | 문서 생성 (기본 draft). `--publish` 사용 시 `--collection` 필수 |
| `update` | `outline update [--body-file FILE] [--title TITLE] [--append] <docId\|URL>` | 문서 수정 (전체 본문 교체, 저장 후 왕복 검증). `--append`는 `--body-file` 필수 |
| `move` | `outline move [--collection COLLECTION] [--parent <docId\|URL>] <docId\|URL>` | 문서 이동 (대상 컬렉션 또는 상위 문서) |
| `archive` | `outline archive <docId\|URL>` | 문서 아카이브 |
| `delete` | `outline delete <docId\|URL>` | 문서 삭제 (휴지통 이동만) |
| `restore` | `outline restore <docId\|URL>` | 문서 복원 (휴지통에서) |
| `revert` | `outline revert <docId\|URL> <revisionId>` | 문서를 특정 리비전으로 되돌리기 |

`docId|URL` 자리에는 문서 ID 또는 전체 URL을 그대로 넣을 수 있다.

## 4. 읽기 규칙

- 모든 위키 작업은 `outline` CLI로만 수행한다. **원시 curl로 API를 직접 호출하지 않는다** — 인증·에러 매핑·출력 정리가 모두 CLI에 들어 있다. CLI가 없거나 실패하면 우회하지 말고 그 사실을 그대로 보고한다.
- `outline search`로 관련 문서를 찾고 → `outline read`로 조회한 뒤 → 요청에 필요한 내용만 발췌·정리한다.
- 출처를 반드시 표기한다: 문서 제목 · 절대 URL · 근거 섹션. 원문 전체 인용은 사용자가 명시적으로 요청한 경우에만 한다.
- 검색이 빗나가면 키워드를 바꿔 2~3회 재시도하고, `outline tree` 또는 `outline list`로 구조를 훑는다. 그래도 찾지 못하면 "없다"고 보고한다 — 추측으로 내용을 지어내지 않는다.

## 5. 쓰기 규칙 (사용자가 명시적으로 요청한 경우에만)

- 문서 본문은 임시 파일에 작성한 뒤 `--body-file`로 전달한다.
- 생성은 기본적으로 draft로 만든다. "바로 발행"이 명시된 경우에만 `--publish`를 사용한다.
- 컬렉션이 지정되지 않았으면 문서를 생성하지 말고, `outline collections` 목록과 추천 후보를 제시해 사용자 확인을 받는다.
- 수정 전에는 반드시 `outline read`로 현재 본문을 확인하고, 수행 후 변경 요약을 보고한다. 읽기 작업 중 문제를 발견하더라도 임의로 고치지 않고 보고만 한다.
- 기존 본문을 유지한 채 덧붙일 때는 `update --append`를 우선 사용한다(기존 본문을 재변환하지 않아 왕복 손실이 없다). 전체 교체는 기존 내용을 고쳐야 할 때만 쓴다.
- `update`가 종료 코드 3으로 끝나면 저장은 됐지만 왕복 검증이 불일치한 것이다. 임의로 재시도하지 않는다. `outline revisions`로 **이번 저장 직전** 리비전을 식별하고(최상단 최신 항목은 방금 저장분일 수 있다), `revert`가 본문과 함께 **제목까지** 리비전 시점으로 되돌린다는 점을 명시한 뒤 사용자 확인을 받아 `outline revert <doc> <revisionId>`를 실행한다.
- `update`가 종료 코드 1로 끝나면 재시도하기 전에 `outline read`로 저장 반영 여부를 먼저 확인한다(이중 수정·리비전 중복 방지).
- `delete`는 휴지통 이동임을 보고에 명시한다(`restore`로 복구 가능).

## 6. 에러·셋업

- 종료 코드 2(토큰 미등록 또는 `OUTLINE_URL` 미설정)면 stderr의 셋업 안내를 그대로 사용자에게 전달한다.
- 401 오류가 나면 `outline doctor`로 진단하고 토큰 재등록 방법을 안내한다.

## 7. 공통

- 문서 언어는 한국어를 기본으로 한다(요청 시 예외).
- 코드 분석은 하지 않는다 — 문서화할 내용은 사용자 요청으로 받거나 지정된 로컬 파일을 읽어서 확보한다.
- 쓰기 작업 보고 형식: 수행 작업 / 대상 문서(제목·URL) / 변경 요약 / 후속 필요 사항(예: draft 발행 필요).
