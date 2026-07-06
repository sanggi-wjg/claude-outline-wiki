# 구현 리뷰 — 2026-07-05 (오케스트레이터 직접 수행, 코드 정확성 중심)

QA 3 scope(스펙 준수·경계면)와 별도로, 코드 자체의 런타임 결함·엣지 케이스를 검토. 산출물 전체(bin/outline 453줄, 에이전트 정의, 매니페스트 2종, README 2종)를 정독.

**판정: 승인. blocker/major 0건, minor 3건, 참고 3건.** 수정은 사용자 지시 대기.

## minor

1. bin/outline:110-118 — call()이 URLError/TimeoutError만 잡음. 연결 중 리셋(ConnectionResetError, http.client.RemoteDisconnected 등 OSError 계열)은 미포착 → 원시 트레이스백 노출 (exit code는 우연히 1 유지, stderr 계약만 깨짐).
2. bin/outline:384,391,397,401,405 — update/move/archive/delete/restore의 metavar가 "docId" (read만 "docId|URL"). 실제로는 전부 URL 허용 — help 표면 비일관.
3. bin/outline:140-146 — abs_url이 상대 경로가 "/"로 시작한다고 가정. Outline 응답이 실제로 항상 "/doc/..."라 현재는 무해.

## 참고

- git push 시 bin/outline이 mode 100755로 커밋되는지 확인 필요 (클론 후 실행권한 보존 여부 — 설치 리허설에 which outline + 실행 확인 포함됨).
- Windows 팀원은 shebang 실행 불가 가능성 — README는 macOS/비macOS(env)만 다룸. v1 범위상 수용.
- create가 --collection 없이 draft 생성을 허용 (documents.create의 collectionId 선택 가정). 위키 인스턴스 버전이 이를 거부하면 400+message로 명확히 실패 — 스모크에서 무컬렉션 draft 생성 1회 확인 권장. (컬렉션 확인 책임은 설계상 CLI가 아닌 에이전트 규칙에 있음.)

## 확인된 강점

- 토큰이 프로세스 인자·로그에 노출되는 경로 없음 (env/키체인 stdout → 헤더 직행)
- title-only update가 text를 재전송하지 않음 → lost-update 완화 설계와 정합
- usage 검증(validate)이 토큰 해석·API 호출보다 앞서 실행 — 부작용 전 거부 순서 올바름
- no-op 가드·permanent 미노출·읽기/쓰기 permission 분리 등 안전장치 일관 적용
