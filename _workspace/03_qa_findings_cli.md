# QA Findings — scope: cli (2026-07-05)

**결과: PASS — 발견사항 0건** (blocker 0 / major 0 / minor 0). QA가 빌더 자가 보고와 독립적으로 T1~T6을 재현, 결과 일치.

## 수행 항목

- T1 py_compile PASS · T2 실행권한+shebang PASS · T3 --help 서브커맨드 12개 PASS · T4 토큰 없음(PATH 심) exit 2 PASS · T6 --publish 가드 exit 2 PASS
- T5: **미검증(환경)** — 샌드박스 CA 번들 부재(`CERTIFICATE_VERIFY_FAILED`) + 엣지 403 차단으로 실서버 미도달. 사용자 스모크로 이관.
- T5 대체 검증: HTTPError 주입으로 401/403/404/429/500/URLError 전 항목이 스킬 에러 매핑 표와 문자 단위 일치 (exit 1, 401만 키체인 재등록 안내 포함)
- 정적 검토: 표준 라이브러리만 / curl 없음 / 토큰 프로세스 인자 미노출(security stdout 수신) / timeout 30초 / delete permanent 미노출 / v1 제외 목록 미구현 / 채택 해석 4건(no-op 가드 등) 실동작 확인

## Integration QA 대조용 원본 사실 (CLI가 내는 실제 신호)

- **B4**: 키체인 서비스명 `outline-token` (안내·조회 일치), env `OUTLINE_API_TOKEN`(빈 문자열=미설정)/`OUTLINE_URL`(기본 `https://wiki.fitpet.kr`), 해석 순서 env→키체인, 미설정 안내 문구에 키체인+env+발급 경로 포함
- **B6**: exit 2 = 토큰 미설정·usage 오류(publish/update/move 가드, body-file 읽기 실패) / exit 1 = API HTTP·네트워크·JSON 파싱 오류. 401 stderr = "토큰이 유효하지 않거나 만료" + 키체인 재등록 안내 + API message 병기 — `doctor` 재실행을 문자적으로 지시하지 않음 (CLI 스펙대로). 배포 에이전트의 "401 → doctor 진단" 규칙과의 정합은 integration에서 판정.

## 담당 빌더 조치 필요 사항

없음.
