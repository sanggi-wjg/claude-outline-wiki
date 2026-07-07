# 11 · Packager fix — 2026-07-07 코드 리뷰 지적사항 반영

mode: fix. 갱신된 `.claude/skills/plugin-packaging/SKILL.md` 스냅샷(version 0.2.0, 행동 규칙 exit 3 재정의 + exit 1 신설, permission 스냅샷 revisions allow + revert 쓰기 열거) 기준으로 산출물 동기화.

## 수정한 파일

1. `/Users/raynor/vscode_workspace/claude-outline-wiki/README.md` (루트 단일 README, 2026-07-05 결정)
   - permission 절 쓰기 계열 열거에 `revert` 추가 + revert가 본문·제목을 통째로 되돌리는 파괴적 쓰기임을 한 줄 명시.
   - allow 목록에 `"Bash(outline revisions:*)"` 추가 (collections 다음).
   - README에는 서브커맨드 목록·exit 코드 표가 없어 구조 보존 — revisions/revert/exit 3 표 항목 추가 없음.
2. `/Users/raynor/vscode_workspace/claude-outline-wiki/claude-plugins/plugins/outline-wiki/agents/outline-wiki.md`
   - 쓰기 규칙 exit 3 문장(L26) 교체: ① `outline revisions`로 "이번 저장 직전" 리비전 식별(최상단 최신 항목은 방금 저장분일 수 있음 명시) ② revert가 본문과 함께 제목까지 되돌린다는 경고 ③ 사용자 확인 후 실행.
   - 그 아래 exit 1 규칙 한 줄 신설: update가 exit 1이면 재시도 전 `outline read`로 저장 반영 여부 확인(이중 수정·리비전 중복 방지).
   - L25 `update --append` 우선 규칙 유지, frontmatter·다른 규칙 무변경.

## 확인만 (무변경)

- `/Users/raynor/vscode_workspace/claude-outline-wiki/claude-plugins/plugins/outline-wiki/.claude-plugin/plugin.json`: version 이미 `0.2.0` — 변경 불필요.
- `bin/outline` 미변경 (cli-developer 영역).
- `marketplace.json` 미변경 (이번 지적사항 범위 밖).

## 검증 결과

| 항목 | 결과 |
|------|------|
| plugin.json `python3 -m json.tool` | PASS |
| marketplace.json `python3 -m json.tool` | PASS |
| `test -x bin/outline` | PASS (실행권한 유지) |
| plugin.json version | 0.2.0 |
| name 삼중 일치 (plugin.json ↔ marketplace entry ↔ 디렉토리) | PASS (outline-wiki) |
| 회사 정보 비내장 grep (fitpet·위키 도메인·개인 이메일) | 0건 |

## 공식 문서 스키마 대조 — 대조 완료

- plugins.md: plugin.json 필드 `name`/`description`/`version`(optional)/`author`(name required, email optional), `bin/` = PATH 등록 실행파일, `agents/` = 플러그인 루트. 현 산출물과 일치. version 0.2.0 유효.
- plugin-marketplaces.md: `owner` **required**(name required, email optional), 동일 리포 플러그인 `source`는 상대경로(`./plugins/outline-wiki`) 유효, entry-level version은 optional — 스킬 스냅샷의 "version은 plugin.json에만" 원칙과 일치.
- 스킬 스냅샷과 공식 문서 간 상충 없음.

## 미확정 / 후속

- 없음. 지적 3건 전부 반영·검증 완료. `/plugin install outline-wiki@sanggi-wjg` 로컬 리허설은 대화형 세션 필요 — 사용자 몫.
