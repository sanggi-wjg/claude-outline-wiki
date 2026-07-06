---
name: outline-plugin-builder
description: Outline 위키 플러그인 빌드 하네스 오케스트레이터. cli-developer·plugin-packager·plugin-qa 에이전트를 조율해 claude-plugins 마켓플레이스 리포(bin/outline CLI + 매니페스트 + 배포 에이전트 + README)를 구현·검증한다. "플러그인 빌드/만들어/구현해", "CLI 구현/수정/고쳐줘", "매니페스트/README/에이전트 정의 수정", "QA 돌려줘", "플러그인 검증/재검증", "다시 실행/재빌드/업데이트/보완", "이전 결과 개선", "설치 준비/배포 준비" 등 outline-wiki 플러그인 산출물 관련 모든 작업 요청 시 반드시 이 스킬을 사용할 것.
---

# Outline Plugin Builder — 오케스트레이터

CONTEXT.md 설계대로 outline-wiki 플러그인을 구현·검증하는 빌드 하네스. 최종 산출물은 마켓플레이스 리포 `{프로젝트 루트}/claude-plugins/` (사용자가 다른 경로를 지정하면 그 경로).

## 실행 모드: 하이브리드

| Phase | 모드 | 이유 |
|-------|------|------|
| Phase 2 (병렬 빌드) | 서브 에이전트 | CLI와 패키징은 산출물 파일이 완전히 분리되어 통신 불필요 |
| Phase 3 (증분 QA + 수정 루프) | 서브 에이전트 + SendMessage 릴레이 | QA 발견사항을 오케스트레이터가 살아있는 빌더 에이전트에 SendMessage로 전달해 수정 — 생성-검증 피드백 루프 |

## 에이전트 구성

| 에이전트 | subagent_type | 스킬 | 산출물 | 보고서 |
|---------|--------------|------|--------|--------|
| cli-developer | `cli-developer` | outline-cli | `claude-plugins/plugins/outline-wiki/bin/outline` | `_workspace/01_cli_selftest.md` |
| plugin-packager | `plugin-packager` | plugin-packaging | 매니페스트 2종 + `agents/outline-wiki.md` + 루트 README | `_workspace/02_packager_report.md` |
| plugin-qa | `plugin-qa` | (교차 매트릭스는 에이전트 정의에 내장) | 없음 (검증만) | `_workspace/03_qa_findings_{scope}.md` |

모든 Agent 호출에 `model: "opus"`를 명시한다.

## 워크플로우

### Phase 0: 컨텍스트 확인 (초기/후속/부분 판별)

1. `_workspace/`와 `claude-plugins/` 존재 여부 확인.
2. 실행 모드 결정:
   - **둘 다 미존재** → 초기 실행. Phase 1로.
   - **존재 + 부분 수정 요청** (예: "에러 메시지만 고쳐줘", "README 갱신") → 부분 재실행. 해당 빌더만 `mode: fix`로 호출하고 → Phase 3의 해당 scope QA → 완료. 이전 산출물 경로와 피드백을 프롬프트에 포함한다.
   - **존재 + 전면 재작업 요청** → 기존 `_workspace/`를 `_workspace_{YYYYMMDD_HHMMSS}/`로 이동 후 Phase 1부터. `claude-plugins/`는 삭제하지 않고 그 위에 수정한다 (git 이력·사용자 수동 변경 보존).

### Phase 1: 준비

1. `_workspace/` 생성. 리포 루트 경로 확정 (기본 `{프로젝트 루트}/claude-plugins`).
2. CONTEXT.md가 갱신되었는지 확인 (스킬 명세와 어긋나는 새 결정이 있으면 스킬을 먼저 갱신하고 CLAUDE.md 변경 이력에 기록).

### Phase 2: 병렬 빌드

**실행 모드:** 서브 에이전트 — 단일 메시지에서 두 Agent를 동시 호출 (`run_in_background: true`, `model: "opus"`).

| 에이전트 | 프롬프트에 담을 것 |
|---------|------------------|
| cli-developer | `mode: build`, `target`: `{repo}/plugins/outline-wiki/bin/outline` 절대 경로, `workspace` 절대 경로 |
| plugin-packager | `mode: build`, `repo_root` 절대 경로, `workspace` 절대 경로. **bin/outline 내용은 cli-developer 담당이므로 건드리지 말 것** 명시 |

### Phase 3: 증분 QA + 수정 루프

**실행 모드:** 서브 에이전트 + SendMessage 릴레이. 전체 완성을 기다리지 않는다 — 각 빌더가 끝나는 즉시 해당 scope QA를 시작한다 (결함 조기 발견이 수정 비용을 줄인다).

1. cli-developer 완료 → plugin-qa를 `scope: cli`로 호출.
2. plugin-packager 완료 → plugin-qa를 `scope: packaging`으로 호출 (별도 호출·순차여도 무방).
3. 둘 다 완료 + scope별 blocker 해소 후 → plugin-qa를 `scope: integration`으로 호출 (경계면 매트릭스 B1~B6).
4. **수정 루프**: QA 발견사항의 담당 빌더에게 SendMessage로 지적사항(증상·기대 동작·근거)을 전달해 `fix` 수행 → 해당 scope 재QA. 빌더 에이전트가 이미 종료되어 SendMessage가 불가하면 같은 subagent_type으로 새로 호출하되 `mode: fix`와 이전 보고서 경로를 프롬프트에 담는다.
5. 루프 상한 **2회**. 2회 후에도 blocker가 남으면 중단하고 미해결 항목을 사용자에게 보고한다 (같은 지적이 2회 반복되면 스킬 명세 자체의 문제일 가능성 — Phase 5 진화 대상).

### Phase 4: 사용자 핸드오프

`/plugin` 명령과 실토큰 스모크는 대화형·자격증명 필요 작업이라 에이전트가 대신할 수 없다. 아래를 **그대로 복사해 실행할 수 있는 형태**로 정리해 최종 보고에 포함한다:

1. 로컬 설치 리허설: `/plugin marketplace add {repo 절대경로}` → `/plugin install outline-wiki@sanggi-wjg` → 새 세션에서 `which outline` + 에이전트 인식 확인
2. 실토큰 등록(`security add-generic-password -s outline-token -a "$USER" -w "<토큰>"`) 후 스모크: `doctor` → `collections` → `search` → `read` → draft `create`/`delete`/`restore` 왕복
3. GitHub 사설 리포 생성·push 명령 + 팀원 1명 설치 리허설 안내

### Phase 5: 마무리

1. `_workspace/` 보존 (감사 추적용).
2. CLAUDE.md 변경 이력 갱신 (산출물·하네스에 변경이 있었던 경우).
3. 결과 요약 보고: 산출물 경로 / QA 최종 상태 (PASS·미해결 목록) / 사용자 핸드오프 절차.
4. 피드백 기회 제공: "결과나 워크플로우에서 개선할 부분이 있나요?" — 반복되는 피드백은 스킬·에이전트 정의에 반영한다.

## 데이터 전달 프로토콜

- **파일 기반 (산출물·보고서)**: 산출물은 리포 경로에 직접, 보고서는 `_workspace/{NN}_{agent}_{artifact}.md`. 절대 경로만 사용.
- **반환값 기반 (요약)**: 각 에이전트의 반환 메시지로 PASS/FAIL 요약과 파일 목록 수집.
- **SendMessage (수정 루프)**: QA 발견사항을 오케스트레이터가 담당 빌더에 릴레이. QA와 빌더는 서로 직접 통신하지 않는다 (수정 책임 소재를 오케스트레이터가 추적).

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| 빌더 1개 실패 | 1회 재시도. 재실패 시 나머지 산출물로 진행하고 최종 보고에 누락 명시 |
| QA 자체 실패 | 1회 재시도. 재실패 시 "미검증" 상태로 보고 — 검증 없이 PASS로 표기하지 않는다 |
| 공식 문서 WebFetch 실패 | 스킬 스냅샷으로 진행 + 최종 보고에 "스키마 미대조 — 로컬 설치 리허설에서 확인 필요" 명시 |
| 네트워크 필요 테스트(T5) 실패 | "미검증" 표기, 사용자 핸드오프의 스모크 테스트로 이관 |
| 스킬 명세 ↔ 공식 문서 상충 | 공식 문서 우선. 스킬 갱신 + CLAUDE.md 변경 이력 기록 |
| 수정 루프 2회 초과 | 중단, 미해결 blocker를 사용자에게 보고 (무한 루프 방지) |

## 테스트 시나리오

### 정상 흐름 (초기 빌드)

1. "플러그인 빌드해줘" → Phase 0: 산출물 없음 → 초기 실행
2. Phase 2: cli-developer와 plugin-packager 병렬 호출 (background)
3. cli-developer 완료(셀프 테스트 T1~T6 PASS) → QA `scope: cli` PASS
4. plugin-packager 완료(JSON 유효·스키마 대조 완료) → QA `scope: packaging` PASS
5. QA `scope: integration` — B1~B6 교차 비교 PASS
6. 최종 보고: `claude-plugins/` 경로 + 사용자 핸드오프 절차 (로컬 설치 리허설 → 실토큰 스모크 → push)

### 에러 흐름 (경계면 결함)

1. QA `scope: integration`에서 B2 위반 발견 — README permission allow에 `Bash(outline update:*)`가 포함됨 (blocker: 쓰기 무프롬프트는 안전장치 위반)
2. 오케스트레이터가 plugin-packager에 SendMessage로 지적사항 릴레이 → `fix` 수행
3. QA `scope: packaging` + B2 재검증 → PASS
4. 최종 보고에 수정 이력 1건 포함

### 에러 흐름 (부분 재실행)

1. "401 에러 메시지 문구 바꿔줘" → Phase 0: 산출물 존재 + 부분 수정 → cli-developer만 `mode: fix` 호출
2. cli-developer가 전체 셀프 테스트 재실행 → QA `scope: cli` + B6 재검증 → 완료 보고
