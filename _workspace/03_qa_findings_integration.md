# QA Findings — scope: integration (2026-07-05)

**결과: PASS — 발견사항 0건** (blocker 0 / major 0 / minor 0). QA가 CLI 표면을 `--help`로 독립 재현 후 경계면 B1~B6 양쪽 파일을 동시에 열어 교차 대조.

## 항목별 판정

- **B1** CLI 실제 표면 ↔ 배포 에이전트 CLI 지시: PASS — 지시된 명령·플래그(search/read/tree/list/collections/doctor/--body-file/--publish/delete/restore) 전부 실존·시그니처 일치. 존재하지 않는 명령 지시 0건.
- **B2** CLI 12 서브커맨드 ↔ README permission allow: PASS — allow = 읽기 6종 1:1, 쓰기 6종 누수 0건 (README가 쓰기 계열 never-allow 명시).
- **B3** 매니페스트 삼중 일치: PASS (packaging 이후 drift 없음).
- **B4** 키체인·env 문자열: PASS — 등록 명령이 CLI 힌트와 README에서 바이트 동일, 서비스명 `outline-token` 등록/조회 짝 성립, env 이름 문자 일치.
- **B5** frontmatter tools ↔ 본문 요구 도구: PASS (재확인).
- **B6** exit code·401 계약 ↔ 에이전트 에러 대응 규칙: PASS — 선행 QA 플래그 항목 판정: 에이전트의 "401 → doctor 진단 + 재등록 방법 보고" 규칙은 CLI 출력에 대한 주장이 아니라 에이전트 자신의 절차이며, doctor(auth.info)는 401/403/404 구분 진단으로 유효하고 재등록 방법은 CLI의 401 stderr(REGISTER_HINT)가 그대로 제공 — 모순 없이 합성됨.

정보성 관찰(조치 불요): CLI 401 stderr는 리터럴 "401" 대신 고유 한국어 안내를 방출 — LLM 에이전트가 의미로 식별 가능, 설계 의도와 일치.

## 담당 빌더 조치 필요 사항

없음. 수정 루프 0회로 전 scope 마감.
