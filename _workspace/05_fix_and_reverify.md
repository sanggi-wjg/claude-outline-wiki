# 리뷰 지적 수정 + QA 재검증 — 2026-07-05

## 수정 (cli-developer, fix 모드)

리뷰(04_review.md) minor 3건 → 전부 해소:

1. call() 네트워크 catch를 (URLError, http.client.HTTPException, TimeoutError, OSError)로 확대 — 연결 리셋 계열도 "네트워크 오류" 안내 + exit 1. import http.client 추가.
2. update/move/archive/delete/restore의 doc metavar → `docId|URL` 통일.
3. abs_url에 "/" 미시작 rel 보정.

## QA 재검증 (scope: cli): PASS

- minor 3건 해소를 모듈 직접 로드 + 예외/입력 주입으로 재현 확인 (ConnectionResetError·RemoteDisconnected·BrokenPipeError·IncompleteRead·bare OSError 등 전부 흡수, 트레이스백 소멸)
- 회귀 중점: (a) HTTPError(OSError 서브클래스)가 광역 catch에 삼켜지지 않음 — HTTPError 절이 앞줄이라 401/403/404/429 매핑 유지, 주입으로 확인 (b) try 블록 내 비네트워크 OSError 발생원 없음 — 오분류 경로 없음 (c) T1~T6 재실행: PASS 5 / 미검증 1(T5 환경 요인, 사용자 스모크 이관 유지)
- 정적·해석 항목 유지: http.client는 표준 라이브러리로 적합, curl 없음, 토큰 미노출, no-op 가드 4건 실동작

## 잔여 관찰 처리

QA 관찰: --parent 옵션 metavar(create·move 2곳)도 URL 허용인데 "docId"로 잔존 (사전 존재, 이번 수정 범위 밖). → help 문자열 2곳이라 오케스트레이터가 직접 수정 (에이전트 왕복 불비례). 수정 후 py_compile OK + create/move --help에서 `docId|URL` 표시 확인, 잔여 `metavar="docId"` 0건, __pycache__ 정리.
