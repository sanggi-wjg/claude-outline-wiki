# 패키저 보고 — 배포용 스킬 번들 추가 (2026-09-15, 오케스트레이터 기록)

plugin-packager `mode: fix` 반환 메시지를 오케스트레이터가 기록한 것 (서브에이전트 환경 가드레일로 보고서 파일 직접 작성 불가 — 2026-07-05 plugin-qa 항목과 동일 제약).

## 생성/수정 파일

- 신규 `claude-plugins/plugins/outline-wiki/skills/outline/SKILL.md` → 설치 후 `/outline-wiki:outline`
- `claude-plugins/plugins/outline-wiki/.claude-plugin/plugin.json` — version 0.2.0→0.3.0, description "에이전트·스킬과 outline CLI" 반영
- `claude-plugins/.claude-plugin/marketplace.json` — 플러그인 항목 description 동일 갱신 (version 미기재 원칙 유지)
- 루트 `README.md` — 제공 플러그인 표·사용법 절 신설·리포 구조도(`skills/outline/SKILL.md`)·설치 후 확인(`/outline-wiki:outline` 노출). 구조도의 `README.md (이 파일)` 잔여 줄 제거 + "README는 이 파일 하나만" 문장 추가
- 미수정: `agents/outline-wiki.md`, `bin/outline`

## 검증 결과

| 항목 | 결과 |
|---|---|
| json.tool marketplace.json / plugin.json | PASS / PASS |
| `claude plugin validate claude-plugins` | ✔ Validation passed |
| `claude plugin validate claude-plugins/plugins/outline-wiki` | ✔ Validation passed |
| `test -x bin/outline` | PASS |
| SKILL.md 첫 바이트 `---` (BOM·선행 공백 없음) | PASS |
| frontmatter 키 | name·description·argument-hint·allowed-tools 4종, 추가 없음 |
| description 길이 | 135자 (≤1,536) |
| allowed-tools | 읽기 7종만, 쓰기 유출 0 |
| CLI 참조 표 | 14개 서브커맨드, 각 `--help` 실행 결과와 프로그램 대조 불일치 0 |
| 회사 정보 grep | 0건 |

## 공식 문서 대조 — 완료, 상충 없음

- plugins.md: `skills/<name>/SKILL.md`는 플러그인 루트 직속, 폴더명=스킬명, plugin.json name=네임스페이스, `$ARGUMENTS` 지원, version·author optional
- skills.md: allowed-tools는 공백/쉼표/YAML 리스트 허용, description+when_to_use 1,536자 컷, 여는 `---`는 첫 줄 필수, 플러그인 스킬은 항상 `/plugin:skill`

## 후속

- README "로컬 검증" 절 `claude plugin validate .` 경로 불일치 → 오케스트레이터가 SendMessage로 fix 지시
- 설치 리허설(`/plugin marketplace add` → `/plugin install outline-wiki@sanggi-wjg` → `/outline-wiki:outline` 노출·`which outline`)은 대화형 전용 → 사용자 스모크 이관
