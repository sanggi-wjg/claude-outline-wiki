# 09 — OUTLINE_URL 설정 안내에 Claude Code settings.json env 방식 병기

- 일시: 2026-07-06
- 요청: 사용자 결정 — 토큰은 키체인 방식 유지, OUTLINE_URL은 `~/.claude/settings.json`의 `env`로도 설정 가능함을 문서·코드에 반영
- 배경: 사용자 로컬 `~/.claude/settings.json`에 `"env": {"OUTLINE_URL": ...}` 적용 후 `outline doctor`로 URL 해석 통과 확인 (동작 검증 완료된 방식의 문서화)
- 실행 모드: 부분 재실행 (cli-developer + plugin-packager 병렬 `mode: fix` → plugin-qa 통합 scope)
- 수정 루프: 0회

## 하네스 명세 갱신 (오케스트레이터 직접, 사용자 결정 선반영 원칙)

| 파일 | 변경 |
|------|------|
| `.claude/skills/outline-cli/SKILL.md` | L27 URL 미설정 안내 명세 — 두 방법 병기(셸 프로필 export / settings.json env, "Claude Code 세션에만 적용" 명시) |
| `.claude/skills/plugin-packaging/SKILL.md` | 비내장 문단(공급 경로 두 방법 명시), README 셋업 스냅샷(a/b 병기), "추가로 명시할 것"(적용 범위 차이 항목 신설) |

## 산출물 수정

| 담당 | 파일 | 변경 |
|------|------|------|
| cli-developer (adcd1b4def7fe7df6) | `bin/outline` | `URL_HINT`(L33-40) 문자열만 — REGISTER_HINT의 `1)/2)` 번호 관례로 두 방법 병기. 로직·exit code 불변, 실행권한 유지 |
| plugin-packager (a15bed2869763ec27) | `README.md` (프로젝트 루트) | 셋업 2단계 a/b 병기, 환경별 설정 섹션에 적용 범위 차이(세션 한정·터미널은 export 병행·둘 다 무방) |

플레이스홀더 `https://<위키 도메인>` 유지 — 회사 URL 비내장 원칙 준수.

## QA (plugin-qa, a8aafd528ff88db67) — PASS, 발견 0건

- cli: py_compile·실행권한·shebang PASS, T7 독립 재현(exit 2 + 두 방법 stderr 출력), T4·T3 회귀 무결, URL 해석·exit code 계약 불변
- packaging: README ↔ 갱신 스킬 스냅샷 문자 단위 정합, 회사 정보 grep 0건, `outline-wiki@sanggi-wjg` 불변, 코드펜스 균형, 매니페스트 VALID
- B4 경계면: env 변수명·settings.json 경로·플레이스홀더·"세션에만 적용" 취지 CLI↔README 교차 일치. 키체인 서비스명 `outline-token`·`OUTLINE_API_TOKEN` 회귀 무결
- 참고: `03_qa_findings_cli.md`의 구 기본값 표기는 스크럽 이전 시점 기록(point-in-time) — 이력 파일이므로 미수정, 혼선 방지용으로 여기 명시

## 후속 (사용자 몫)

- 팀원 안내 시 두 방법 중 택1 (Claude Code만 쓰면 settings.json이 간편, 터미널 병용 시 셸 프로필)
