# claude-outline-wiki

Outline 위키용 Claude Code 플러그인(outline-wiki)을 만들어 사설 GitHub 마켓플레이스로 배포하는 프로젝트. 설계의 단일 출처는 `CONTEXT.md` (2026-07-05 확정 핸드오프).

## 하네스: Outline 플러그인 빌더

**목표:** CONTEXT.md 설계대로 마켓플레이스 리포 `claude-plugins/`(bin/outline CLI + 매니페스트 + 배포 에이전트 + README)를 구현·검증하여 팀원이 `/plugin install outline-wiki@sanggi-wjg`로 설치 가능하게 한다.

**트리거:** 플러그인 빌드·수정·재빌드·QA·검증·패키징·배포 준비 등 산출물 관련 작업 요청 시 `outline-plugin-builder` 스킬을 사용하라. 단순 질문은 직접 응답 가능.

**변경 이력:**
| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-07-05 | 초기 구성 (에이전트 3 + 스킬 3) | 전체 | - |
| 2026-07-05 | marketplace.json 스냅샷에 owner 필수·version 단일화 반영 | skills/plugin-packaging | 공식 문서 대조에서 스냅샷 누락 발견 (공식 문서 우선 원칙) |
| 2026-07-05 | QA 출력 프로토콜을 반환 메시지 일차 채널로 변경, 감사 파일은 오케스트레이터가 기록 | agents/plugin-qa | 서브에이전트 환경 가드레일이 보고서 파일 작성을 제한함을 첫 실행에서 발견 |
| 2026-07-05 | no-op 가드(update·move)·body-file exit 2·list 출력 형식 명문화 | skills/outline-cli | 빌더의 보수적 해석 4건 채택 — fix 모드에서의 퇴행 방지 |
| 2026-07-05 | 초기 빌드 완료 — claude-plugins/ 산출, QA 3 scope 전부 PASS (수정 루프 0회, T5만 환경 요인 미검증→사용자 스모크 이관) | claude-plugins/ | 첫 빌드 실행 |
| 2026-07-05 | 리뷰 minor 3건 수정(네트워크 catch 확대·metavar 통일·abs_url 보정) + --parent metavar 잔여 통일, QA 재검증 PASS | claude-plugins/bin/outline | 코드 리뷰 후속 (04_review.md, 05_fix_and_reverify.md) |
| 2026-07-05 | README 루트 단일화 — 플러그인 README 삭제, 셋업·permission 내용 루트로 통합. 스킬 명세도 동기화(재빌드 시 복원 금지) | claude-plugins/README.md, skills/plugin-packaging | 사용자 결정 |
| 2026-07-05 | 회사 정보 비내장(전면 스크럽) — 위키 URL 기본값 제거(OUTLINE_URL 필수·T7 신설), 마켓플레이스명 fitpet→team, owner/author 중립화, 문서 스크럽. 스킬·QA 매트릭스 동기화 | skills 2종, agents/plugin-qa, claude-plugins/ 전체 | 사용자 결정 (CONTEXT.md의 URL 내장 결정 무효화) |
| 2026-07-05 | 스크럽 통합 QA 재판정 PASS — 잔존 grep 6패턴 0건, B1~B6 정합, 부산물 정리 | claude-plugins/ | 배포 게이트 재통과 (07_scrub_and_final_qa.md) |
| 2026-07-06 | 마켓플레이스명 team→sanggi-wjg (설치 참조 `outline-wiki@sanggi-wjg`). 스킬 2종·plugin-qa B3·CLAUDE.md 목표 라인 동기화(목표 라인 잔존은 커밋 전 코드 리뷰에서 발견·수정), QA packaging 재검증 PASS (개인 계정명은 비내장 원칙과 무충돌 명시. CONTEXT.md의 fitpet 명명은 07-05 스크럽에서 이미 무효화) | marketplace.json, README.md, skills 2종, agents/plugin-qa, CLAUDE.md | 사용자 결정 — 개인 GitHub 계정 리포 배포 (08_rename_marketplace_sanggi-wjg.md) |
| 2026-07-06 | OUTLINE_URL 설정 안내 두 방법 병기 — 셸 프로필 export + Claude Code settings.json `env`(세션 한정 명시). CLI URL_HINT·README 셋업·스킬 2종(outline-cli L27, plugin-packaging 스냅샷) 동기화, QA cli+packaging+B4 PASS | bin/outline, README.md, skills 2종 | 사용자 결정 — settings.json env 방식 실검증 후 문서화 (09_outline_url_settings_env.md) |
