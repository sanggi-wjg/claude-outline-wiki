# QA 판정 기록 — 2026-07-07 코드 리뷰 반영 수정 (오케스트레이터 기록)

리뷰(xhigh, 10-angle 파인더 → 후보별 검증 → 스위프)에서 확정된 CONFIRMED 8건·PLAUSIBLE 4건·하네스 동기화 5건을 반영한 뒤, scope 3종을 순차 재검증했다. 세 scope 모두 발견사항 0으로 PASS — 수정 루프 0회.

## scope: packaging — PASS

- README permission: 읽기 7개(doctor·search·read·list·tree·collections·revisions) allow 1:1, 쓰기 7종(revert 포함) 누출 0건, revert 파괴적 쓰기 명시.
- agents/outline-wiki.md: exit 3 규칙 3요소(직전 리비전 식별·제목 동반 회귀 경고·사용자 확인), exit 1 규칙(재시도 전 read 확인), --append 우선 규칙 유지, frontmatter 무변경.
- plugin.json 0.2.0, 매니페스트 JSON 유효, name 삼중 일치, 실행권한 유지, 회사 정보 grep 0건.
- 공식 문서(plugins.md·plugin-marketplaces.md) 스키마 대조 완료 — 상충 없음.
- minor 1건(패키징 스킬 구조도의 README 위치 문구 드리프트, 2026-07-05 결정 잔여)은 오케스트레이터가 스킬 문구를 실제 위치(프로젝트 루트 README.md)로 정합화하여 해소.

## scope: cli — PASS

- T1~T9 전부 독립 재현 PASS (T5는 환경 OUTLINE_URL로 더미 토큰 401 경로만 — 비내장 원칙 유지).
- 리뷰 회귀 케이스: 표 직후 헤딩/인용 비병합, 구분자 행 없는 파이프 줄(들여쓰기 코드·`|x|` 문단) 비병합, ``` 안 ~~~ 무시, 정상 다중 줄 표 병합 유지 — 전부 통과.
- 왕복 검증: documents.info 재호출 제거 확인(grep), update 응답 data.text 비교, 경고문 docId 응답 UUID 사용, --append 미정규화·검증 제외, create 정규화, revisions 비-UUID 해석 — mock으로 전부 재현.
- 기존 계약 무퇴행: no-op 가드, exit 0/1/2/3 체계, list/read 출력 형식, 토큰→URL 확인 순서.

## scope: integration — PASS (B1~B6)

- B1: 에이전트 md 지시 명령 전부 파서 시그니처 일치 (revert <doc> <revisionId> positional 순서 포함).
- B2: 읽기 7 allow 1:1, 쓰기 7 누출 0.
- B3: name 삼중 일치, version 0.2.0, 설치 참조 outline-wiki@sanggi-wjg 성립.
- B4: outline-token·OUTLINE_API_TOKEN·OUTLINE_URL 두 방법 병기 문자 단위 일치.
- B5: tools(Bash·Read·Write)가 본문 요구 도구 전부 커버.
- B6: exit 3(저장됨+불일치)·exit 1 대응·401→doctor 신호 전부 CLI 실신호와 일치.
- T5(실토큰 스모크)만 사용자 이관 — 설계상 예외.

## 빌더 보고서

- CLI 셀프 테스트: `_workspace/10_cli_selftest_reviewfix.md`
- 패키저: `_workspace/11_packager_reviewfix.md`
