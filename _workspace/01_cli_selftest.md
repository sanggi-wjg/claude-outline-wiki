# bin/outline 셀프 테스트 결과

- 대상: `/Users/raynor/vscode_workspace/claude-outline-wiki/claude-plugins/plugins/outline-wiki/bin/outline`
- 명세: `.claude/skills/outline-cli/SKILL.md` (2026-07-05 갱신본 — 회사 정보 비내장, OUTLINE_URL 필수, T5·T7 변경)
- 실행 환경: macOS (Darwin 24.6.0), python3 3.14.0 (`/Library/Frameworks/Python.framework/Versions/3.14`)
- 최종 실행일: 2026-07-05 (fix 모드 재실행 — 회사 정보 비내장 반영 후 전체 매트릭스 재실행)
- 요약: **PASS 6 / FAIL 0 / 미검증 1** (미검증 = T5, 실서버 네트워크 불가)

`BIN` = 위 대상 절대 경로. `WORKSPACE` = `/Users/raynor/vscode_workspace/claude-outline-wiki/_workspace`.

---

## fix 모드 수정 요약 (2026-07-05, 회사 정보 비내장)

사용자 결정 "회사 정보 비내장"에 따른 수정. 지적된 부분만 수정, 전체 재작성 없음.

1. **DEFAULT_BASE_URL 제거** — 내장 기본값(`https://wiki.fitpet.kr`) 상수 삭제. `resolve_base_url()`은 `OUTLINE_URL` env 만 읽고, 빈 문자열/미설정은 `None`(미설정 취급). `main()`에서 URL 미설정 시 `print_url_missing()`로 stderr 안내(`export OUTLINE_URL="https://<위키 도메인>"`) 후 exit 2. `URL_HINT` 상수 신설.
2. **확인 순서 고정** — `main()`: ① 토큰 → ② URL. 토큰이 우선 실패하므로 T4 재현성 유지.
3. **회사 정보 스크럽** — 모듈 docstring("셀프 호스트 Outline 위키 API 래퍼 CLI"), argparse description("… 베이스 URL 은 OUTLINE_URL env")에서 회사 URL 제거.
4. **보존** — 이전 리뷰 후속으로 오케스트레이터가 수정한 `--parent` metavar(create·move) `docId|URL` 그대로 유지.

### 스크럽 검증
- `grep -in "fitpet" bin/outline` → **0건** (exit 1 = 매치 없음)
- `grep -in "wiki.fitpet|DEFAULT_BASE_URL|fitpetmall|FitpetKorea" bin/outline` → **0건**
- 오프라인: `DEFAULT_BASE_URL` 심볼 부재, `resolve_base_url` unset→None / empty→None / set→값, 401 매핑(exit 1 + 토큰 안내) 정상 유지 확인.

---

## T1 — 컴파일

- 명령: `python3 -m py_compile "$BIN"`
- 기대: exit 0
- 실제: `T1 exit=0`
- 판정: **PASS**

## T2 — 실행권한 + shebang

- 명령: `test -x "$BIN" && head -1 "$BIN"`
- 기대: 실행권한 존재 + 첫 줄 `#!/usr/bin/env python3`
- 실제: `executable: yes` / `#!/usr/bin/env python3`
- 판정: **PASS**

## T3 — 전체 서브커맨드 목록

- 명령: `"$BIN" --help`
- 기대: exit 0, 전체 서브커맨드 목록 표시
- 실제: exit 0. 12개 서브커맨드 표시. description = `셀프 호스트 Outline 위키 API 래퍼 CLI (베이스 URL 은 OUTLINE_URL env)` — 회사 URL 없음.
- 판정: **PASS**

## T4 — 토큰 없음 시뮬레이션 (PATH 심)

- 준비:
  ```bash
  mkdir -p "$WORKSPACE/shim" && printf '#!/bin/sh\nexit 44\n' > "$WORKSPACE/shim/security" && chmod +x "$WORKSPACE/shim/security"
  ```
- 명령: `OUTLINE_API_TOKEN= PATH="$WORKSPACE/shim:$PATH" "$BIN" doctor`
- 기대: exit 2, stderr에 키체인 + env 등록 안내
- 실제: `T4 exit=2`, stderr:
  ```
  오류: Outline API 토큰을 찾을 수 없습니다.
  토큰 등록 방법:
    1) macOS 키체인 (권장):
       security add-generic-password -s outline-token -a "$USER" -w "<API 토큰>"
    2) 환경변수:
       export OUTLINE_API_TOKEN="<API 토큰>"
    API 토큰은 Outline 위키 설정 → API Tokens 에서 발급합니다.
  ```
- 부가 검증: `OUTLINE_URL` 미설정 상태에서도 **토큰 안내가 먼저** 나옴(확인 순서 ① 토큰 → ② URL 고정). 빈 문자열 `OUTLINE_API_TOKEN=`가 미설정으로 취급됨 확인.
- 판정: **PASS**

## T5 — 더미 토큰 401 (실 인스턴스 URL 지정, 네트워크 필요)

- 명령: `OUTLINE_API_TOKEN=dummy OUTLINE_URL=<실 인스턴스 URL> "$BIN" doctor`
- 기대: exit 1, stderr에 401 안내
- 실제 (1차 + 재시도 1회 모두 동일):
  ```
  오류: 네트워크 오류 — [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate (_ssl.c:1077)
  T5 exit=1
  ```
- 판정: **미검증** — 실서버에 도달하지 못함. 사유(환경 요인, CLI 결함 아님):
  1. python.org 배포 python3 3.14 에 CA 번들 미설치(`ssl.get_default_verify_paths()` cafile/capath 모두 None) → TLS 핸드셰이크에서 실패(HTTP 응답 도달 전).
  2. 진단용 curl(시스템 CA)로도 이 샌드박스 경로에서 `403 Forbidden` 평문 — Outline 표준 401 JSON 아님(엣지 차단으로 보임).
- 이번 fix 확인 사항: OUTLINE_URL 지정 시 URL 해석이 정상 동작하여 **HTTP 호출 단계까지 진입**함(네트워크 예외로 exit 1). URL 미지정이면 T7처럼 exit 2로 조기 종료됨. 401 매핑 로직은 오프라인 주입으로 정상 재확인(exit 1 + 401 안내 + message). 정상 TLS·실서버 환경에서는 PASS 예상.

## T6 — usage 에러 (API 호출 전 거부)

- 명령: `OUTLINE_API_TOKEN=dummy OUTLINE_URL=https://example.invalid "$BIN" create --title t --publish --body-file /tmp/x` (`--collection` 없이)
- 기대: usage 에러 exit 2 — API 호출 전 거부
- 실제: `T6 exit=2`, stderr:
  ```
  usage: outline create [-h] --title TITLE [--collection COLLECTION]
                        [--parent docId|URL] --body-file FILE [--publish]
  outline create: error: --publish 지정 시 --collection 이 필요합니다 (발행 문서는 컬렉션 필수)
  ```
- 부가 검증: usage 검증이 토큰·URL 해석·본문 파일 읽기·API 호출보다 선행됨(`/tmp/x` 부재에도 거부). `--parent` metavar `docId|URL` 보존 확인.
- 판정: **PASS**

## T7 — OUTLINE_URL 미설정 → exit 2 (신설)

- 명령: `OUTLINE_API_TOKEN=dummy OUTLINE_URL= "$BIN" doctor`
- 기대: exit 2, stderr에 `OUTLINE_URL` 설정 안내 (빈 문자열=미설정 검증 겸함)
- 실제: `T7 exit=2`, stderr:
  ```
  오류: 베이스 URL(OUTLINE_URL)이 설정되지 않았습니다.
  베이스 URL 설정 방법:
       export OUTLINE_URL="https://<위키 도메인>"
    사용 중인 Outline 인스턴스의 주소로 지정하세요.
  ```
- 부가 검증: `OUTLINE_URL`을 아예 unset 한 케이스(`env -u OUTLINE_URL`)도 동일하게 exit 2 + 동일 안내. 토큰(dummy)은 존재하므로 URL 미설정 경로가 정확히 트리거됨(확인 순서 ② URL).
- 판정: **PASS**

---

## 보조 검증 — 이전 fix 회귀 방지 (유지 확인)

- fix#1(네트워크 예외 포착 확대: http.client.HTTPException·OSError 계열): 코드 변경 없음, 유지.
- 401 HTTPError 매핑: 오프라인 주입으로 exit 1 + 토큰 안내 재확인(광역 except 절보다 HTTPError 절이 앞서 매칭).
- abs_url(leading-slash 보정), normalize_doc_id(URL→마지막 segment), clean_snippet, 출력 형식: 변경 없음.
