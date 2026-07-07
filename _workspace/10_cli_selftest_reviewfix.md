# CLI 셀프 테스트 — 2026-07-07 코드 리뷰 fix 재검증

- 대상: `claude-plugins/plugins/outline-wiki/bin/outline`
- 명세: `.claude/skills/outline-cli/SKILL.md` (본문 정규화·왕복 검증 섹션, revisions/revert, exit 3, T8·T9)
- 모드: fix — 리뷰 지적 6건 반영 후 전체 매트릭스 재실행
- 요약: **PASS 9 / FAIL 0 / 미검증 0** (T1~T9 전부 PASS). T5 는 실 인스턴스 접근 가능하여 검증 완료.

## 수정 항목 → 반영 위치

| # | 리뷰 지적 | 반영 |
|---|---|---|
| 1 | normalize_tables 오병합 3종 | 구분자 행 lookahead 진입 가드, 헤딩/인용 병합 제외·표 종료, 펜스 문자·길이 매칭 추적(`_classify_fences`/`_opening_fence`/`_closing_fence`/`_is_separator_row`) |
| 2 | canonical_text fence-aware 재작성 | 펜스/산문 구간 분할, 펜스 내부 rstrip-only 엄격, 산문만 느슨(ASCII 구두점 이스케이프만 해제·하드브레이크/`<br>`/`&nbsp;` 동치·`_`→`*`·마커 `[ \t]*` 무제한·ATX/setext/hr 제거·공백 제거), 구간 NUL 결합 |
| 3 | cmd_update 왕복 검증 | `documents.info` 재호출 제거 → `documents.update` 응답 `data.text` 직접 비교, 경고 docId 3곳 응답 UUID(`d.get("id", doc_id)`), exit 3 유지 |
| 4 | --append 미정규화 | append 시 read_body_file 원문 그대로 전송, 전체 교체만 normalize_tables |
| 5 | create 정규화 | create 본문에도 normalize_tables 적용(왕복 검증 없음) |
| 6 | revisions docId 해석 | UUID 아니면 `documents.info` 로 UUID 해석 후 revisions.list (`is_uuid`/`_UUID_RE`) |

## 매트릭스

| # | 명령 | 기대 | 실제 | 판정 |
|---|---|---|---|---|
| T1 | `python3 -m py_compile bin/outline` | exit 0 | exit 0 | PASS |
| T2 | `test -x bin/outline && head -1` | 실행권한 + shebang | executable: yes / `#!/usr/bin/env python3` | PASS |
| T3 | `outline --help` | exit 0, 전체 서브커맨드 | exit 0, 14개 서브커맨드 표시 | PASS |
| T4 | 토큰 없음 시뮬(security shim exit44 + 빈 env) `doctor` | exit 2, 키체인+env 안내 | exit 2, REGISTER_HINT 출력 | PASS |
| T5 | `OUTLINE_API_TOKEN=dummy OUTLINE_URL=<실인스턴스> doctor` | exit 1, 401 안내 | exit 1, "토큰이 유효하지 않거나 만료됨" + `API message: Unable to decode token` | PASS |
| T6 | `create --title t --publish --body-file /tmp/x` (--collection 없이) | usage 에러 exit 2 | exit 2, "--publish 지정 시 --collection 필요" | PASS |
| T7 | `OUTLINE_URL= doctor` | exit 2, OUTLINE_URL 안내 | exit 2, URL_HINT(두 방법 병기) 출력 | PASS |
| T8 | `revert doc-id` (revisionId 없이) | usage 에러 exit 2 | exit 2, "the following arguments are required: revisionId" | PASS |
| T9 | python3 오프라인 단위 8케이스 | ①~⑧ 전부 통과 | 아래 상세 전부 PASS | PASS |

### T5 비고
실 인스턴스(OUTLINE_URL env 로 공급됨)에 **더미 토큰**으로만 요청 — 인증·데이터 접근 없이 401 경로만 확인. 회사 정보 비내장 원칙 유지(코드/문서에 URL 미기재, env 로만 공급).

## T9 상세 (python3 실제 실행)

normalize_tables / canonical_text 를 `SourceFileLoader` 로 로드해 직접 호출.

```
[PASS] T9-①a 들여쓰기 파이프 줄 비병합        "    | id | name |\n    output line" → 무변형
[PASS] T9-①b |x| 문단 비병합                  "|x| denotes…\nplain…" → 무변형
[PASS] T9-②h 표 직후 헤딩 비병합              "…| 1 | 2 |\n## 다음 섹션" → 헤딩 미병합·표 종료
[PASS] T9-②q 표 직후 인용 비병합              "…| 1 | 2 |\n> 인용문" → 인용 미병합·표 종료
[PASS] T9-③ ``` 안 ~~~ 무시(무변형)           "```text\n~~~\n| a | b |\nplain\n~~~\n```" → 무변형
[PASS] T9-④ 정상 다중 줄 표 병합             "| Item | line one\nline two |" → "…line one<br>line two |"
[PASS] T9-④b 파이프 끝 행 공백 병합           "| Item |\ncontinued |" → "| Item | continued |"
[PASS] T9-⑤ setext↔ATX 동치                   ct("# Title") == ct("Title\n====")
[PASS] T9-⑥ 4칸 중첩 불릿 마커 교체 동치      ct("    - item") == ct("    * item")
[PASS] T9-⑥b 순서목록 1.↔1) 동치             ct("    1. item") == ct("    1) item")
[PASS] T9-⑦ C:\new↔C:\\new 동치               비구두점 백슬래시 유지로 오탐 방지
[PASS] T9-⑧ 펜스 내부 들여쓰기 변경 → 불일치  ct("```\n    x\n```") != ct("```\nx\n```")
[PASS] 추가 토글금지: 펜스 뒤 표 정상 병합
[PASS] 추가 <br>↔개행 / &nbsp;↔공백 / 하드브레이크 / 행말 2스페이스 동치
ALL T9 CASES PASS
```

리뷰 재현 케이스 4종(헤딩 비병합, 들여쓰기 파이프 비병합, ``` 안 ~~~ 무변형, 정상 다중 줄 표 병합) 모두 고쳐짐 확인.

## 추가 동작 검증 (오프라인 mock — `call` 대체, 네트워크 없음)

fix #3~#6 의 실제 디스패치 흐름을 mock 으로 검증:

```
[PASS] #3a 전체교체 동치 → exit 0, documents.info 재호출 없음, documents.update 1회만
[PASS] #3b 불일치 → exit 3, 경고에 응답 UUID 포함·입력 slug 미포함
[PASS] #4  append 원문 그대로 전송(미정규화), payload append=True, 검증 제외 exit 0
[PASS] #5  create 본문 정규화(다중 줄 표 병합되어 전송)
[PASS] #6a non-UUID(slug) → documents.info 선행 후 해석된 UUID 로 revisions.list
[PASS] #6b UUID → documents.info 생략
ALL MOCK CASES PASS
```

## 명세 상충
없음. 명세 지시대로 구현. 보수적 해석/판단 위임 사항 없음.
