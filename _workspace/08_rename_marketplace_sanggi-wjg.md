# 08 — 마켓플레이스명 변경 (team → sanggi-wjg) 및 QA 재검증

- 일시: 2026-07-06
- 요청: 사용자 결정 — 설치 참조를 `outline-wiki@sanggi-wjg`로 변경 (배포자 개인 GitHub 계정 리포로 배포 예정)
- 실행 모드: 부분 재실행 (plugin-packager `mode: fix` → plugin-qa `scope: packaging`)
- 수정 루프: 0회

## 하네스 명세 갱신 (오케스트레이터 직접, 사용자 결정 선반영 원칙)

| 파일 | 변경 |
|------|------|
| `.claude/skills/plugin-packaging/SKILL.md` | 마켓플레이스명 `team` → `sanggi-wjg` 6곳 (최종 사용 경험·비내장 원칙 문단·리포 구조·매니페스트 스냅샷·README 예시·검증 절차). 비내장 원칙 문단에 "개인 GitHub 계정명은 회사 식별 정보가 아니므로 무충돌" 명시 |
| `.claude/skills/outline-plugin-builder/SKILL.md` | Phase 4 로컬 설치 리허설 명령 `@sanggi-wjg`로 갱신 |
| `.claude/agents/plugin-qa.md` | B3 확인 내용의 설치 명령 `@sanggi-wjg`로 갱신 |

## 산출물 수정 (plugin-packager, agentId a8df8a94be147de9c)

| 파일 | 변경 |
|------|------|
| `claude-plugins/.claude-plugin/marketplace.json` | `"name": "team"` → `"name": "sanggi-wjg"` (owner·description·plugins 유지) |
| `README.md` (프로젝트 루트) | team 참조 4곳: 제목(L1)·안내 문장(L3)·설치 명령(L16)·리포 구조 주석(L68) |

미변경(의도): `bin/outline` (내부 "team"은 Outline API 응답 필드), `plugin.json` name, 플러그인 디렉토리명.

## QA 재검증 (plugin-qa, agentId a56fbbf4add78d179) — PASS, 발견 0건

- B3: marketplace.json name=`sanggi-wjg`, plugin.json↔marketplace entry↔디렉토리명 삼중 일치, source 경로 실존, owner·description 존재, version 단일화 유지
- B5 퇴행 없음: agents/outline-wiki.md frontmatter·본문 정합 유지
- JSON 유효성: marketplace.json·plugin.json VALID
- 실행권한: `bin/outline` -rwxr-xr-x 유지
- `team` 잔존 참조: QA 범위(claude-plugins/ + README) 내 0건 (bin/outline의 API 필드 2건 제외), README 4곳 일관 전환, `fitpet` 등 회사 정보 신규 노출 0건. **주의: 이 grep은 리포 전체가 아닌 QA 범위 한정이었음** — 커밋 전 코드 리뷰(아래)에서 CLAUDE.md 목표 라인 잔존 1건 추가 발견

## 커밋 전 코드 리뷰 후속 (2026-07-06, 파인더 10앵글 + 스윕)

- **수정 완료**: CLAUDE.md L7 목표 라인의 `outline-wiki@team` 잔존 → `@sanggi-wjg`로 동기화, 변경 이력 행에 대상 추가
- **확인(수정 불필요)**: `sanggi-wjg`는 공식 문서 명명 규칙(kebab-case) 적합, marketplace.json JSON·인코딩 정상, 이름-경로 삼중 일치, 실행권한 유지. 구 `team` 마켓플레이스는 등록·배포된 적 없어(세션 첫 에러 "Marketplace not found"로 확인) 마이그레이션 안내 불필요
- **미해결(사용자 결정 필요)**: README 위치 불일치 — 실물은 프로젝트 루트, plugin-packaging 스킬 L89·README 자체 트리 다이어그램은 `claude-plugins/README.md`를 명시. claude-plugins/만 별도 리포로 push하면 README가 배포 리포에 포함되지 않음. CONTEXT.md는 여전히 fitpet 명명(2회 리네임 이전) — 변경 이력으로 무효화 기록됨

## 후속 (사용자 몫)

- 로컬 설치 리허설: `/plugin marketplace add <repo 절대경로>` → `/plugin install outline-wiki@sanggi-wjg`
- GitHub 배포 시 리포는 `claude-plugins/` 내용이 리포 루트가 되도록 별도 리포로 push
