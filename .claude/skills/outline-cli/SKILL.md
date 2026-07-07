---
name: outline-cli
description: Outline 위키 API 래퍼 CLI(bin/outline)의 구현 명세 — 서브커맨드, 인증 해석, 에러 매핑, 출력 형식, 셀프 테스트 매트릭스. bin/outline 구현·수정, outline CLI 서브커맨드 추가/변경, 인증·토큰 처리, 에러 메시지, CLI 테스트 작업 시 반드시 이 스킬을 사용할 것. Outline API를 직접 호출하는 코드를 작성하기 전에도 먼저 확인할 것.
---

# outline CLI 구현 명세

셀프 호스트 Outline 위키 API를 감싸는 단일 파일 Python CLI. 플러그인의 `bin/outline`으로 배포되며, 플러그인 `bin/`은 Bash PATH에 자동 추가되므로 배포 에이전트는 `outline <subcommand>`로 호출한다.

**회사 정보 비내장 (2026-07-05 사용자 결정)**: 코드·문구 어디에도 특정 회사의 위키 URL·회사명·조직명을 넣지 않는다. 베이스 URL은 기본값 없이 `OUTLINE_URL`로만 공급받는다.

## 구현 원칙

- **Python 3 표준 라이브러리만, 단일 파일.** shebang `#!/usr/bin/env python3`, `chmod +x`. 팀원이 pip 셋업 없이 설치 즉시 사용해야 하므로 외부 의존성 금지 (macOS 기본 python3 = 3.14 기준).
- HTTP는 `urllib.request`. curl 서브프로세스 금지 — 토큰이 프로세스 인자에 들어가면 `ps`로 노출된다.
- argparse 서브커맨드 구조. `outline --help`가 전체 서브커맨드 목록을 보여줘야 한다.

## 인증·URL 해석

| 항목 | 해석 순서 |
|---|---|
| 토큰 | ① `OUTLINE_API_TOKEN` env (빈 문자열은 미설정으로 취급) → ② `security find-generic-password -s outline-token -w` |
| 베이스 URL | `OUTLINE_URL` env **필수** — 기본값 없음 (빈 문자열은 미설정으로 취급) |

- 확인 순서: ① 토큰 → ② URL (테스트 재현성을 위해 순서 고정).
- 토큰이 둘 다 없으면: stderr로 등록 방법 안내(키체인 명령 예시 + `OUTLINE_API_TOKEN` 대안) 후 **exit 2**.
- `OUTLINE_URL` 미설정이면: stderr로 설정 방법 안내 후 **exit 2**. 안내는 두 방법 병기 — ① 셸 프로필 `export OUTLINE_URL="https://<위키 도메인>"` ② Claude Code 사용자 설정(`~/.claude/settings.json`)의 `"env": { "OUTLINE_URL": "https://<위키 도메인>" }` (Claude Code 세션에만 적용됨을 명시).
- `security`는 subprocess로 호출하되 토큰은 stdout으로만 받는다. `security`가 없는 플랫폼(비 macOS)에서는 조용히 건너뛰고 env만 본다.

## API 호출 규약

전 엔드포인트 공통: `POST {base}/api/{endpoint}`, JSON body, 헤더 `Authorization: Bearer <token>` + `Content-Type: application/json`, timeout **30초**.

## 서브커맨드

| 커맨드 | API | 비고 |
|---|---|---|
| `doctor` | `auth.info` | 토큰·계정 확인 (셋업 검증용) |
| `search <query> [--collection <id>] [--limit N]` | `documents.search` | limit 기본 10 |
| `read <docId\|URL>` | `documents.info` | 메타 헤더 + 마크다운 본문 |
| `list [--collection <id>] [--limit N]` | `documents.list` | limit 기본 25, updatedAt DESC |
| `tree <collectionId>` | `collections.documents` | 들여쓰기 트리 + id |
| `collections` | `collections.list` | id, 이름 |
| `create --title <t> [--collection <id>] [--parent <docId>] --body-file <f> [--publish]` | `documents.create` | |
| `update <docId> [--body-file <f>] [--title <t>] [--append]` | `documents.update` | 전체 본문 교체 시 저장 후 왕복 검증 → 불일치면 **exit 3** (아래 '본문 정규화·왕복 검증') |
| `move <docId> [--collection <id>] [--parent <docId>]` | `documents.move` | |
| `archive <docId>` | `documents.archive` | |
| `delete <docId>` | `documents.delete` | 휴지통 이동만. **permanent 옵션 미노출** — 영구삭제 미노출은 확정된 안전장치다 |
| `restore <docId>` | `documents.restore` | |
| `revisions <docId\|URL> [--limit N]` | `revisions.list` | limit 기본 10. docId가 UUID 형식이 아니면(urlId·URL 유래) `documents.info`로 UUID를 해석한 뒤 조회 — revisions.list는 documents.*와 달리 urlId를 받지 않는다 |
| `revert <docId\|URL> <revisionId>` | `documents.restore` | 특정 리비전으로 되돌리기 — 본문과 제목이 함께 리비전 시점으로 돌아간다 |

## 동작 규칙

- **docId 인자**: 전체 URL도 허용 — 마지막 path segment를 추출한다 (Outline API는 UUID와 urlId 모두 허용, 검증 완료).
- **create**: 기본 `publish=false` (draft). `--publish` 지정 시 `--collection` 필수 — 없으면 usage 에러로 거부.
- **update**: `--append`는 `--body-file` 필수 (documents.update의 append 파라미터 사용).
- **본문 전달은 항상 `--body-file <파일>`** — 셸 인자로 본문을 받으면 JSON 이스케이핑·인용부호 문제가 재발하므로 원천 차단.
- **no-op 가드** (2026-07-05 확정 해석): `update`는 `--body-file`/`--title` 중 1개 이상, `move`는 `--collection`/`--parent` 중 1개 이상 필수 — 없으면 usage 에러 exit 2. 무의미한 API 호출을 사전 차단한다.
- `--body-file` 읽기 실패는 입력 오류로 exit 2.
- `list` 출력은 id·제목만 (URL 없음) — search/read/tree만 절대 URL 포함.

## 본문 정규화·왕복 검증 (2026-07-07 코드 리뷰 반영)

**normalize_tables** — Outline이 셀 안에 블록 요소가 있는 표를 여러 물리 줄로 직렬화하는 결함을 병합 복구한다. 적용 지점: `read` 출력, `create`/`update`의 송신 본문. **`--append` 조각에는 적용하지 않는다** (신규 저작 조각을 변형할 이유가 없고, append는 검증도 불가). 오병합 방지 가드 3종 필수:

1. **표 모드 진입은 구분자 행 확인 후에만** — `|` 시작 줄이라도 구분자 행(`|---|` 형태)이 확인된 표 블록 안에서만 병합한다. (들여쓰기 코드 블록의 파이프 줄, `|x|` 같은 문단을 표로 오인 금지.)
2. **새 블록 시작 줄은 병합 제외** — 헤딩(`#`)·인용(`>`)·펜스 시작 줄은 표 직후 빈 줄이 없어도 병합하지 않고 표를 종료한다. (목록 마커·일반 문단은 셀 내 블록 직렬화의 주 대상이므로 병합 유지.)
3. **펜스 추적은 문자 종류·길이 매칭** — 여는 펜스의 문자(```` ``` ````/`~~~`)와 길이를 기억하고, 같은 문자·같은 길이 이상의 닫는 줄만 펜스를 닫는다. 무조건 토글 금지 (``` 블록 안의 ~~~ 줄이 상태를 뒤집으면 안 된다).

**왕복 검증 (update 전체 본문 교체 시)** — `documents.update` 응답의 `data.text`(서버가 재직렬화한 저장본)와 송신 본문을 canonical 비교한다. **검증용 `documents.info` 재호출 금지** — 응답에 저장본이 이미 있고, 재호출은 불필요한 왕복 + 동시 편집 레이스 오탐 + 저장 성공 후 exit 1 오신호를 만든다. 불일치 시: stderr 경고(현재 상태 read·`revisions`·`revert` 안내 — 안내에 넣는 docId는 응답의 UUID를 사용, urlId 금지) 후 **exit 3**. `--append`·`--title` 단독 수정은 검증 제외.

**canonical 비교 규칙** — 펜스 내부는 엄격 비교(행 우측 공백만 무시), 산문 구간만 느슨 비교:

- 이스케이프 해제는 CommonMark 규칙대로 **ASCII 구두점 앞 백슬래시만** (`\C` 같은 비구두점·개행은 유지 → `C:\new` 오탐 방지).
- 하드 브레이크 동치화: 백슬래시+개행·`<br>`·행말 스페이스 2개를 모두 단순 개행으로. `&nbsp;` 엔티티는 공백으로.
- 목록 마커 정규화는 들여쓰기 깊이 무제한 (`[ \t]*`) — 불릿과 순서 목록 동일 규칙.
- 제목은 ATX 마커 제거 + setext 밑줄(`===`/`---` 단독 줄) 제거로 동치화. 수평선(`-`/`=`/`*`/`_` 3개 이상 단독 줄)은 제거.
- 나머지 공백 차이는 무시. **알려진 한계(명시적 트레이드오프)**: 산문의 문단 경계·단어 간 공백 손실은 흡수된다 — 오경보(exit 3 남발)가 손실 미탐보다 배포 에이전트 흐름에 더 해롭다는 판단. 코드 펜스 내부는 엄격 비교라 이 한계가 적용되지 않는다.

## 출력 형식 — 컨텍스트 다이어트

호출자는 컨텍스트가 유한한 서브에이전트다. 출력은 필요 최소한으로:

- `search`: 문서당 id · 제목 · **절대 URL** · 매칭 스니펫(HTML 태그 제거, ~200자)만.
- `read`: 본문 앞에 한 줄 메타 헤더 (제목 · id · collectionId · updatedAt · 절대 URL).
- `tree`: 들여쓰기 트리 + 각 노드의 id.
- `collections` / `list`: 한 줄에 하나, id와 제목 중심.
- **절대 URL 조합은 스크립트 책임** — API 응답의 상대 경로(`url` 필드)에 base URL을 붙여서 출력한다. 호출자가 조합하게 두면 실수한다.

## 에러 매핑

API 에러는 stderr에 안내 + 응답 JSON의 `message` 병기, **exit 1**. (토큰 미설정·usage 오류는 exit 2.) **exit 3**은 update 왕복 검증 불일치 전용 — 저장은 완료된 부분 성공 신호로, 실패(1)·거부(2)와 구분된다.

| HTTP | stderr 안내 |
|---|---|
| 401 | 토큰이 유효하지 않거나 만료됨. 키체인 `outline-token` 재등록 방법 안내 |
| 403 | 권한 부족 — 문서/컬렉션 권한 또는 API 키 스코프 확인 |
| 404 | 문서/컬렉션 ID 확인 |
| 429 | rate limit — 잠시 후 재시도 |
| 기타/네트워크 | 상태코드(또는 예외)와 message 그대로 |

## 셀프 테스트 매트릭스

구현 완료 판정 기준. 결과는 `_workspace/01_cli_selftest.md`에 명령·기대값·실제 출력과 함께 기록한다.

| # | 명령 | 기대 |
|---|---|---|
| T1 | `python3 -m py_compile bin/outline` | exit 0 |
| T2 | `test -x bin/outline && head -1 bin/outline` | 실행권한 + `#!/usr/bin/env python3` |
| T3 | `outline --help` | exit 0, 전체 서브커맨드 목록 표시 |
| T4 | 토큰 없음 시뮬레이션 (아래) 후 `doctor` | **exit 2**, stderr에 키체인+env 등록 안내 |
| T5 | `OUTLINE_API_TOKEN=dummy OUTLINE_URL=<실인스턴스 URL> bin/outline doctor` | **exit 1**, stderr에 401 안내 (실 서버 대상 — 네트워크 필요) |
| T6 | `OUTLINE_API_TOKEN=dummy bin/outline create --title t --publish --body-file /tmp/x` (`--collection` 없이) | usage 에러 exit 2 — API 호출 전에 거부 |
| T7 | `OUTLINE_API_TOKEN=dummy OUTLINE_URL= bin/outline doctor` | **exit 2**, stderr에 `OUTLINE_URL` 설정 안내 (빈 문자열=미설정 검증 겸함) |
| T8 | `OUTLINE_API_TOKEN=dummy OUTLINE_URL=x bin/outline revert doc-id` (revision 인자 없이) | usage 에러 **exit 2** — API 호출 전에 거부 |
| T9 | python3 단위 검증 (오프라인) | normalize_tables: ① 구분자 행 없는 `\|` 줄 비병합 ② 표 직후 헤딩/인용 비병합 ③ ``` 안의 ~~~ 무시 ④ 정상 다중 줄 표는 병합. canonical: ⑤ setext↔ATX 동치 ⑥ 4칸 중첩 불릿 마커 교체 동치 ⑦ `C:\new`↔`C:\\new` 동치 ⑧ 펜스 내부 들여쓰기 변경은 **불일치** 판정 |

**T4의 토큰 없음 시뮬레이션**: 개발 머신 키체인에 실토큰이 있어도 재현 가능해야 한다. PATH 심으로 `security`를 "항목 없음"으로 위장:

```bash
mkdir -p "$WORKSPACE/shim" && printf '#!/bin/sh\nexit 44\n' > "$WORKSPACE/shim/security" && chmod +x "$WORKSPACE/shim/security"
OUTLINE_API_TOKEN= PATH="$WORKSPACE/shim:$PATH" bin/outline doctor; echo "exit=$?"
```

(exit 44는 macOS `security`의 errSecItemNotFound. 빈 env가 미설정으로 취급되는지도 이 테스트가 함께 검증한다.)

## v1 의도적 제외 — 구현 금지

아래는 설계 인터뷰에서 **과설계로 판단해 제외**했다. 요청 없이 추가하지 않는다:

- 첨부파일/이미지 업로드 (`attachments.create`는 multipart)
- dry-run/diff 프리뷰, 로컬 감사 로그 (Outline 자체의 리비전 히스토리·휴지통이 감사 추적 제공)
- 자동 페이지네이션, 캐싱, `read --outfile`
- `documents.delete`의 permanent 파라미터

## 알려진 한계 (구현 시 인지할 것)

- **lost-update**: `documents.update`는 전체 본문 교체이고 revision 충돌 감지가 없다. 완화는 CLI가 아니라 배포 에이전트의 "수정 전 read" 규칙이 담당 — CLI에서 해결하려 들지 않는다.
