# bin/outline 셀프 테스트 결과 (fix 모드 — 2026-07-06)

- 대상: `/Users/raynor/vscode_workspace/claude-outline-wiki/claude-plugins/plugins/outline-wiki/bin/outline`
- 스킬 명세: `.claude/skills/outline-cli/SKILL.md` (2026-07-06 갱신 — L27 대조)
- 변경 요청: `OUTLINE_URL` 미설정 안내(`URL_HINT`)에 셸 프로필 export + Claude Code `~/.claude/settings.json` env 두 방법 병기
- 변경 범위: `URL_HINT` 문자열 1개소만 (L33-40). 로직·exit code·해석 순서 불변.
- 실행 환경: macOS (Darwin 24.6.0), python3

## 매트릭스

| # | 명령 | 기대 | 실제 | 판정 |
|---|------|------|------|------|
| T1 | `python3 -m py_compile bin/outline` | exit 0 | exit 0 | PASS |
| T2 | `test -x bin/outline && head -1` | 실행권한 + `#!/usr/bin/env python3` | executable: yes / `#!/usr/bin/env python3` | PASS |
| T3 | `outline --help` | exit 0, 서브커맨드 12종 표시 | exit 0, doctor/search/read/list/tree/collections/create/update/move/archive/delete/restore 표시 | PASS |
| T4 | 토큰 없음 시뮬 후 `doctor` | exit 2, stderr 키체인+env 등록 안내 | exit 2, REGISTER_HINT(키체인 1) + env 2)) 출력 | PASS |
| T5 | `OUTLINE_API_TOKEN=dummy OUTLINE_URL=<실인스턴스> doctor` | exit 1, 401 안내 (네트워크) | 실 인스턴스 URL 부재 — 미실행 | 미검증 |
| T6 | `create --title t --publish --body-file /tmp/x` (--collection 없이) | usage 에러 exit 2 (API 호출 전 거부) | exit 2, `--publish 지정 시 --collection 이 필요합니다` | PASS |
| T7 | `OUTLINE_API_TOKEN=dummy OUTLINE_URL= doctor` | exit 2, `OUTLINE_URL` 설정 안내 (빈 문자열=미설정) | exit 2, URL_HINT 두 방법 병기 출력 | PASS |

## T7 상세 (이번 변경의 핵심)

명령:
```
OUTLINE_API_TOKEN=dummy OUTLINE_URL= bin/outline doctor
```

실제 stderr 출력:
```
오류: 베이스 URL(OUTLINE_URL)이 설정되지 않았습니다.
베이스 URL 설정 방법:
  1) 셸 프로필 (예: ~/.zshrc, ~/.bashrc):
     export OUTLINE_URL="https://<위키 도메인>"
  2) Claude Code 사용자 설정 (~/.claude/settings.json) — Claude Code 세션에만 적용됨:
     "env": { "OUTLINE_URL": "https://<위키 도메인>" }
  사용 중인 Outline 인스턴스의 주소로 지정하세요.
exit=2
```

병기 검증 (grep 어서션, 전부 PASS):
- `export OUTLINE_URL=` (방법 1) 존재
- `~/.claude/settings.json` (방법 2 경로) 존재
- `"env": { "OUTLINE_URL"` (settings.json 스니펫) 존재
- `Claude Code 세션에만 적용됨` (적용 범위 명시) 존재
- `https://<위키 도메인>` 플레이스홀더 유지 (회사 URL 내장 0건)

스타일: 기존 REGISTER_HINT의 `  1) ... / 5칸 들여쓴 명령 / 2) ...` 관례를 그대로 따름.

## URL 해석 회귀 확인 (비어있지 않은 OUTLINE_URL 통과)

명령:
```
OUTLINE_API_TOKEN=dummy OUTLINE_URL="https://outline.invalid.test" bin/outline doctor
```
결과: `오류: 네트워크 오류 — [Errno 8] nodename nor servname...`, exit 1
→ 비어있지 않은 URL은 URL 가드(exit 2)를 통과해 API 호출 단계(네트워크 exit 1)까지 도달함을 확인. URL 해석 순서·동작 불변.

## 부가 확인

- 스코프 diff: `git diff` 결과 URL_HINT 4줄 추가만, 다른 변경 없음. 파일 모드 100755 유지.
- 실행권한: `test -x` PASS.
- 회사정보 스크럽: `grep -nE 'fitpet|getoutline|\.co\.kr'` → 0건.

## 요약

PASS 6 / FAIL 0 / 미검증 1 (T5 — 실 Outline 인스턴스 네트워크 필요, 사용자 스모크로 이관)
