---
name: plugin-packaging
description: Claude Code 플러그인 패키징 명세 — 마켓플레이스 리포 구조, marketplace.json/plugin.json 작성, 배포용 outline-wiki 에이전트 정의, 팀원 셋업 README, permission 권장 규칙, 로컬 설치 검증. 플러그인 리포 골격 생성, 매니페스트 작성/수정, 배포 에이전트 정의 작성, README 갱신, /plugin install 준비·검증 작업 시 반드시 이 스킬을 사용할 것.
---

# 플러그인 패키징 명세

outline-wiki 플러그인을 사설 GitHub 마켓플레이스 리포로 패키징하는 명세. 최종 사용 경험: 팀원이 `/plugin marketplace add <사설 리포 git URL>` → `/plugin install outline-wiki@team`.

**회사 정보 비내장 (2026-07-05 사용자 결정)**: 산출물(매니페스트·에이전트 정의·README·CLI) 어디에도 회사 식별 정보를 넣지 않는다 — 위키 URL, 회사명, GitHub 조직 URL, 개인/회사 이메일 전부 금지. 마켓플레이스명은 중립적 `team`, owner는 `{"name": "outline-wiki maintainers"}`, README의 리포 주소는 `<사설 리포 git URL>` 플레이스홀더. 위키 주소는 `OUTLINE_URL` 환경변수로만 공급된다(README 셋업에 필수 단계로 포함).

## 리포 구조

```
claude-plugins/                        # 마켓플레이스 리포 루트 (사설)
├── .claude-plugin/
│   └── marketplace.json               # name: "team", 플러그인 목록 (상대 경로 source)
├── plugins/
│   └── outline-wiki/
│       ├── .claude-plugin/
│       │   └── plugin.json            # name, description, version
│       ├── agents/
│       │   └── outline-wiki.md        # 배포용 에이전트 정의
│       └── bin/
│           └── outline                # Python CLI (cli-developer 담당 — 내용 수정 금지)
└── README.md                          # 마켓플레이스 안내 + 팀원 셋업 (단일 README)
```

README는 **루트 하나만** 둔다 (2026-07-05 사용자 결정). 플러그인 디렉토리 안에 README를 만들지 않는다 — 재빌드 시 복원 금지.

## 매니페스트 작성

**스키마는 기억으로 쓰지 않는다.** 작성 전 반드시 공식 문서를 WebFetch로 확인해 대조한다:

- https://code.claude.com/docs/en/plugin-marketplaces.md — marketplace.json 스키마, 동일 리포 내 상대 경로 source 표기
- https://code.claude.com/docs/en/plugins.md — plugin.json 스키마, agents/·bin/ 번들 규칙

문서 확인 불가 시 아래 스냅샷(2026-07-05 설계 시점)으로 작성하되 보고에 "공식 문서 미대조" 플래그를 남긴다:

- `marketplace.json`: `name: "team"` (→ 설치 명령의 `@team`이 됨), **`owner` 필수** — `{ "name": "outline-wiki maintainers" }` (이메일 등 개인 정보 미기재), 최상위 `description` 권장(검증기 경고 회피), `plugins` 배열에 `{ "name": "outline-wiki", "source": "./plugins/outline-wiki", "description": ... }` — description에 회사 URL 금지
- `plugin.json`: `{ "name": "outline-wiki", "description": ..., "version": "0.1.0" }` — `name`은 marketplace.json 항목·디렉토리명과 일치해야 한다
- `version`은 plugin.json에만 둔다 — marketplace entry와 중복 시 조용히 plugin.json이 우선하므로 (공식 문서 경고) 두 곳에 두면 drift만 생긴다

검증: 각 JSON에 `python3 -m json.tool`, source 경로 실존 여부.

## 배포용 에이전트 정의 (`agents/outline-wiki.md`)

### frontmatter — 아래 그대로 사용

```yaml
---
name: outline-wiki
description: 사내 Outline 위키의 문서 검색·조회·작성·수정·관리를 담당한다.
  개발 중 비즈니스 정책·기획 배경·스펙·가이드 등 위키에 있을 법한 정보가 불확실할 때
  proactively 사용해 검색·인용한다. 문서 생성·수정·이동·아카이브·삭제는 사용자가
  명시적으로 요청한 경우에만 수행한다.
tools: Bash, Read, Write
---
```

- `model` 필드는 **생략** — 생략이 세션 모델 상속(inherit)이며 공식 기본값이다. 팀원마다 쓰는 모델이 달라 고정하지 않는다.
- description의 "proactively" 문구는 공식 자동 위임 메커니즘이므로 유지한다.
- `${CLAUDE_PLUGIN_ROOT}`는 hooks/MCP 설정에서만 치환되고 **에이전트 마크다운 본문에서는 동작하지 않는다**. 본문에서 경로 참조 금지 — 플러그인 `bin/`이 PATH에 자동 추가되므로 `outline <subcommand>`로만 호출하게 한다.

### 본문에 담을 행동 규칙

읽기 규칙:

- 모든 위키 작업은 `outline` CLI로만. **원시 curl로 API 직접 호출 금지** — 스크립트가 없거나 실패하면 그 사실을 보고한다. (이유: 인증·에러 매핑·출력 다이어트가 전부 CLI에 있다.)
- search → 관련 문서 read → 위임받은 질문에 필요한 내용만 발췌·정리.
- 반환 형식: `## 답변` + `## 출처` (문서 제목 · 절대 URL · 근거 섹션). 원문 전체는 명시 요청 시에만.
- 검색이 빗나가면 키워드를 바꿔 2~3회 재시도하고 tree/list로 훑는다. 그래도 없으면 "없다"고 보고 — 추측으로 지어내지 않는다.

쓰기 규칙 (위임 프롬프트에 명시된 경우에만):

- 본문은 Write로 임시 파일에 작성 후 `--body-file`로 전달.
- 생성은 기본 draft. "바로 발행" 명시 시에만 `--publish`.
- 컬렉션 미지정 시 생성하지 말고 `collections` 목록 + 추천 후보를 반환해 확인 요청.
- 수정 전 반드시 read로 현재 본문 확인, 수행 후 변경 요약 보고. 읽기 위임 중 문제를 발견해도 고치지 않고 보고만.
- delete는 휴지통 이동임을 보고에 명시 (restore로 복구 가능).

공통:

- 문서 언어는 한국어 (요청 시 예외).
- 코드 분석은 하지 않는다 — 문서화할 내용은 위임 프롬프트로 받거나 지정된 로컬 파일을 Read로 읽는다.
- 401 발생 시 `outline doctor`로 진단하고 토큰 재등록 방법을 보고에 포함.
- 쓰기 작업 보고 형식: 수행 작업 / 대상 문서(제목·URL) / 변경 요약 / 후속 필요 사항 (예: draft 발행 필요).

## 루트 README (`claude-plugins/README.md`)

마켓플레이스 안내(등록 명령·제공 플러그인 목록·리포 구조)와 팀원 셋업 절차를 하나의 파일에 담는다:

```bash
# 1. 마켓플레이스 등록 + 설치 (Claude Code 안에서)
/plugin marketplace add <사설 리포 git URL>
/plugin install outline-wiki@team

# 2. 위키 주소 설정 (셸 프로필에 — 필수, 기본값 없음)
export OUTLINE_URL="https://<위키 도메인>"

# 3. Outline API 토큰 발급 (위키 설정 → API Tokens) 후 키체인 등록 (터미널에서)
security add-generic-password -s outline-token -a "$USER" -w "<API 토큰>"

# 4. 검증
outline doctor
```

추가로 명시할 것:

- macOS가 아니면 토큰은 `OUTLINE_API_TOKEN` 환경변수로 대체.
- 자동 업데이트를 원하면 `GH_TOKEN` env 필요 (사설 리포).
- **permission 권장 규칙** — 읽기만 무프롬프트, 쓰기는 프롬프트 유지. "파괴적 작업은 명시 요청 시에만" 정책을 permission 레이어에서도 보존하는 장치이므로 쓰기 계열(create·update·move·archive·delete·restore)을 allow에 넣지 않는다:

```json
"permissions": {
  "allow": [
    "Bash(outline doctor:*)",
    "Bash(outline search:*)",
    "Bash(outline read:*)",
    "Bash(outline list:*)",
    "Bash(outline tree:*)",
    "Bash(outline collections:*)"
  ]
}
```

## 검증 절차

패키징 완료 판정 기준:

1. 매니페스트 JSON 유효성 (`python3 -m json.tool`) + name·경로 삼중 일치 (plugin.json ↔ marketplace.json ↔ 디렉토리명).
2. `bin/outline` 실행권한 유지 확인 (`test -x`) — 패키징 중 권한이 벗겨지지 않았는지.
3. 로컬 설치 리허설 명령 준비 (실행은 사용자 몫 — `/plugin` 명령은 대화형 세션에서만 가능):
   - `/plugin marketplace add <리포 로컬 절대경로>` → `/plugin install outline-wiki@team` → 새 세션에서 에이전트 인식 + `outline` PATH 확인 (`which outline`).
4. 회사 정보 비내장 확인: 산출물 전역 grep으로 회사 위키 도메인·회사명·조직 URL·이메일이 0건인지 검증.
5. GitHub push 전 체크: 사설 리포 접근에 git 자격증명 필요, README에 토큰 안내 포함 여부.
