---
name: plugin-packaging
description: Claude Code 플러그인 패키징 명세 — 마켓플레이스 리포 구조, marketplace.json/plugin.json 작성, 배포용 outline-wiki 에이전트 정의, 팀원 셋업 README, permission 권장 규칙, 로컬 설치 검증. 플러그인 리포 골격 생성, 매니페스트 작성/수정, 배포 에이전트 정의 작성, README 갱신, /plugin install 준비·검증 작업 시 반드시 이 스킬을 사용할 것.
---

# 플러그인 패키징 명세

outline-wiki 플러그인을 사설 GitHub 마켓플레이스 리포로 패키징하는 명세. 최종 사용 경험: 팀원이 `/plugin marketplace add <사설 리포 git URL>` → `/plugin install outline-wiki@sanggi-wjg`.

**회사 정보 비내장 (2026-07-05 사용자 결정)**: 산출물(매니페스트·에이전트 정의·README·CLI) 어디에도 회사 식별 정보를 넣지 않는다 — 위키 URL, 회사명, GitHub 조직 URL, 개인/회사 이메일 전부 금지. 마켓플레이스명은 `sanggi-wjg`(배포자 개인 GitHub 계정명, 2026-07-06 사용자 결정 — 회사 식별 정보가 아니므로 비내장 원칙과 무충돌), owner는 `{"name": "outline-wiki maintainers"}`, README의 리포 주소는 `<사설 리포 git URL>` 플레이스홀더. 위키 주소는 `OUTLINE_URL` 환경변수로만 공급된다(README 셋업에 필수 단계로 포함 — 셸 프로필 export 또는 Claude Code `~/.claude/settings.json`의 `env`, 두 방법 병기).

## 리포 구조

```
claude-plugins/                        # 마켓플레이스 리포 루트 (사설)
├── .claude-plugin/
│   └── marketplace.json               # name: "sanggi-wjg", 플러그인 목록 (상대 경로 source)
├── plugins/
│   └── outline-wiki/
│       ├── .claude-plugin/
│       │   └── plugin.json            # name, description, version
│       ├── agents/
│       │   └── outline-wiki.md        # 배포용 에이전트 정의
│       ├── skills/
│       │   └── outline/
│       │       └── SKILL.md           # 배포용 스킬 정의 → /outline-wiki:outline (2026-09-15 추가)
│       └── bin/
│           └── outline                # Python CLI (cli-developer 담당 — 내용 수정 금지)
└── (README 없음 — 아래 참고)
```

README는 **하나만** 둔다 (2026-07-05 사용자 결정). 현재 단일 README의 실제 위치는 프로젝트 루트 `README.md`(claude-plugins/ 상위)이며, `claude-plugins/` 안이나 플러그인 디렉토리 안에 README를 만들지 않는다 — 재빌드 시 복원 금지. `claude-plugins/`를 독립 리포로 push할 때 README 동봉이 필요하면 그 시점에 이동을 결정한다.

## 매니페스트 작성

**스키마는 기억으로 쓰지 않는다.** 작성 전 반드시 공식 문서를 WebFetch로 확인해 대조한다:

- https://code.claude.com/docs/en/plugin-marketplaces.md — marketplace.json 스키마, 동일 리포 내 상대 경로 source 표기
- https://code.claude.com/docs/en/plugins.md — plugin.json 스키마, agents/·skills/·bin/ 번들 규칙
- https://code.claude.com/docs/en/skills.md — SKILL.md frontmatter 필드(name·description·argument-hint·allowed-tools 등), 플러그인 스킬 네임스페이스 규칙

문서 확인 불가 시 아래 스냅샷(2026-07-05 설계 시점)으로 작성하되 보고에 "공식 문서 미대조" 플래그를 남긴다:

- `marketplace.json`: `name: "sanggi-wjg"` (→ 설치 명령의 `@sanggi-wjg`가 됨), **`owner` 필수** — `{ "name": "outline-wiki maintainers" }` (이메일 등 개인 정보 미기재), 최상위 `description` 권장(검증기 경고 회피), `plugins` 배열에 `{ "name": "outline-wiki", "source": "./plugins/outline-wiki", "description": ... }` — description에 회사 URL 금지
- `plugin.json`: `{ "name": "outline-wiki", "description": ..., "version": "0.3.0" }` — `name`은 marketplace.json 항목·디렉토리명과 일치해야 한다. `name`은 스킬 네임스페이스도 된다(`/outline-wiki:outline`). (version 0.2.0 = 2026-07-07 리비전 기능·왕복 검증, **0.3.0 = 2026-09-15 스킬 번들 추가**)
- marketplace.json 플러그인 항목·plugin.json의 `description`은 제공 컴포넌트를 반영한다: "…담당하는 에이전트·스킬과 outline CLI를 제공한다."
- `version`은 plugin.json에만 둔다 — marketplace entry와 중복 시 조용히 plugin.json이 우선하므로 (공식 문서 경고) 두 곳에 두면 drift만 생긴다

검증: 각 JSON에 `python3 -m json.tool`, source 경로 실존 여부, `claude plugin validate <리포 루트>` 및 `claude plugin validate <플러그인 디렉토리>` 통과.

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
- 기존 본문을 유지한 채 덧붙일 때는 `update --append` 우선 (기존 본문을 재변환하지 않아 왕복 손실이 없다). 전체 교체는 기존 내용을 고칠 때만.
- `update`가 exit 3으로 끝나면: 저장은 됐지만 왕복 검증 불일치. 임의 재시도 금지. `outline revisions`로 **이번 저장 직전** 리비전을 식별하고(최상단 최신 항목은 방금 저장분일 수 있음), revert는 본문과 함께 **제목도** 리비전 시점으로 되돌린다는 점을 명시한 뒤 사용자 확인을 받아 `outline revert <doc> <revisionId>` 실행.
- `update`가 exit 1로 끝나면: 재시도 전에 `outline read`로 저장 반영 여부를 먼저 확인 (이중 수정·리비전 중복 방지).
- delete는 휴지통 이동임을 보고에 명시 (restore로 복구 가능).

공통:

- 문서 언어는 한국어 (요청 시 예외).
- 코드 분석은 하지 않는다 — 문서화할 내용은 위임 프롬프트로 받거나 지정된 로컬 파일을 Read로 읽는다.
- 401 발생 시 `outline doctor`로 진단하고 토큰 재등록 방법을 보고에 포함.
- 쓰기 작업 보고 형식: 수행 작업 / 대상 문서(제목·URL) / 변경 요약 / 후속 필요 사항 (예: draft 발행 필요).

## 배포용 스킬 정의 (`skills/outline/SKILL.md`) — 2026-09-15 추가

플러그인 스킬은 `skills/<name>/SKILL.md`로 번들되고, 설치 후 `/outline-wiki:outline`으로 호출된다(plugin.json `name`이 네임스페이스, 폴더명/`name`이 스킬명). `.claude-plugin/` 안에 두지 않는다 — 플러그인 루트 직속.

**역할 분담** — 에이전트와 스킬은 같은 규칙을 두 진입점으로 제공한다:

| 컴포넌트 | 진입 | 용도 |
|---|---|---|
| `agents/outline-wiki.md` | 메인 에이전트의 proactive 위임 / `@outline-wiki` | 격리 컨텍스트에서 검색·발췌 (원본 규칙) |
| `skills/outline/SKILL.md` | `/outline-wiki:outline [요청]` 수동 호출 + Claude 자동 로드 | 메인 세션이 `outline` CLI를 직접 쓸 때의 사용법·안전 규칙 참조 (요약본) |

스킬의 행동 규칙은 에이전트 정의의 규칙을 **완화 없이** 요약한 것이어야 한다(안전장치 동일). 규칙 원본은 에이전트 정의이며, 에이전트 규칙이 바뀌면 스킬도 같이 갱신한다.

### frontmatter — 아래 그대로 사용

```yaml
---
name: outline
description: Outline 위키 문서를 outline CLI로 검색·조회·작성·수정·관리한다. "위키에서 찾아줘", "위키 문서 읽어줘/만들어줘/수정해줘", outline 명령 사용법, 위키 컬렉션·리비전 확인 등 Outline 위키 관련 요청 시 사용.
argument-hint: [위키 작업 요청]
allowed-tools: Bash(outline doctor:*) Bash(outline search:*) Bash(outline read:*) Bash(outline list:*) Bash(outline tree:*) Bash(outline collections:*) Bash(outline revisions:*)
---
```

- 첫 줄이 반드시 `---` (앞에 빈 줄·BOM 금지 — 아니면 전체가 본문으로 취급된다).
- `allowed-tools`는 **읽기 7종만** — README permission allow와 동일 집합. 쓰기 계열(create·update·move·archive·delete·restore·revert)은 절대 넣지 않는다 (호출 턴 한정 사전 승인이라도 "쓰기는 프롬프트 유지" 정책 위반).
- `disable-model-invocation`·`user-invocable` 미설정(기본값: 사용자·Claude 모두 호출 가능). `context: fork`·`agent` 미사용 — 스킬은 메인 세션 지식용이고, 격리 실행은 에이전트가 담당한다.
- `model` 미설정(세션 모델 상속). `${CLAUDE_PLUGIN_ROOT}`는 스킬 본문에서 치환되지만 사용하지 않는다 — bin/이 PATH에 등록되므로 `outline <subcommand>`로만 호출.
- description은 자동 호출 판단 근거이므로 트리거 문구를 포함하되 1,536자 이내.

### 본문 구성 (순서대로)

1. **인자 처리**: `$ARGUMENTS`가 있으면 그 요청을 아래 규칙대로 수행한다. 인자 없이 호출되면 `outline doctor`로 연결 상태를 확인하고 사용 가능한 명령 요약을 보여준다.
2. **위임 기준**: 여러 문서를 읽어 종합해야 하거나 검색 범위가 넓으면 `outline-wiki` 서브에이전트에 위임한다(컨텍스트 절약). 단일 명령·소규모 작업은 메인 세션에서 직접 실행한다.
3. **CLI 참조 표**: `outline --help`의 14개 서브커맨드 전부, 실제 시그니처와 일치하게 (search·read·list·tree·collections·doctor·create·update·move·archive·delete·restore·revisions·revert). 플래그명·positional 순서를 `bin/outline --help`/각 서브커맨드 `--help`에서 그대로 옮긴다 — 기억으로 쓰지 않는다.
4. **읽기 규칙**: search → read → 필요한 내용만 발췌, 출처(제목·절대 URL·근거 섹션) 표기. 원시 curl 금지. 검색 실패 시 키워드 변경 2~3회 + tree/list, 그래도 없으면 "없다"고 보고.
5. **쓰기 규칙 (사용자 명시 요청 시에만)**: 본문은 임시 파일 → `--body-file`. 기본 draft(`--publish`는 "바로 발행" 명시 시만). 컬렉션 미지정 시 생성하지 말고 `collections` 후보 확인. 수정 전 `read`, `--append` 우선. `update` exit 3 → 재시도 금지·`revisions`로 직전 리비전 식별·revert는 제목까지 되돌림을 명시 후 사용자 확인. exit 1 → 재시도 전 `read`로 반영 여부 확인. delete는 휴지통 이동(restore 가능).
6. **에러·셋업**: exit 2(토큰/`OUTLINE_URL` 미설정)는 stderr 안내를 그대로 전달. 401은 `outline doctor`로 진단 후 토큰 재등록 안내.
7. 문서 언어 한국어 기본, 쓰기 보고 형식(수행 작업/대상 문서/변경 요약/후속 필요 사항).

## 루트 README (프로젝트 루트 `README.md` — claude-plugins/ 상위)

마켓플레이스 안내(등록 명령·제공 플러그인 목록·리포 구조)와 팀원 셋업 절차를 하나의 파일에 담는다:

```bash
# 1. 마켓플레이스 등록 + 설치 (Claude Code 안에서)
/plugin marketplace add <사설 리포 git URL>
/plugin install outline-wiki@sanggi-wjg

# 2. 위키 주소 설정 (필수, 기본값 없음) — 아래 a·b 중 하나
#    a) 셸 프로필: 터미널과 Claude Code 모두 적용
export OUTLINE_URL="https://<위키 도메인>"
#    b) Claude Code 사용자 설정(~/.claude/settings.json)에 env 추가: Claude Code 세션에만 적용
#       { "env": { "OUTLINE_URL": "https://<위키 도메인>" } }

# 3. Outline API 토큰 발급 (위키 설정 → API Tokens) 후 키체인 등록 (터미널에서)
security add-generic-password -s outline-token -a "$USER" -w "<API 토큰>"

# 4. 검증
outline doctor
```

추가로 명시할 것:

- `OUTLINE_URL` 두 방법의 적용 범위 차이: settings.json `env`는 Claude Code 세션(Bash 도구)에만 주입된다 — 터미널에서 `outline`을 직접 실행하려면 셸 프로필 export 병행 (둘 다 설정해도 무방).
- macOS가 아니면 토큰은 `OUTLINE_API_TOKEN` 환경변수로 대체.
- 자동 업데이트를 원하면 `GH_TOKEN` env 필요 (사설 리포).
- **제공 컴포넌트·사용법** (2026-09-15): 제공 플러그인 표에 에이전트 + 스킬 + CLI를 명시하고, 사용법 절에 두 진입점을 안내한다 — ① 서브에이전트 `outline-wiki`(proactive 위임 또는 `@outline-wiki`) ② 스킬 `/outline-wiki:outline [요청]`(메인 세션에서 직접, 인자 없이 호출하면 doctor + 명령 요약). 리포 구조도에 `skills/outline/SKILL.md`를 포함한다.
- 설치 후 확인 항목에 `/help`의 Custom commands 탭(또는 `/` 메뉴)에 `/outline-wiki:outline`이 보이는지 추가한다.
- **permission 권장 규칙** — 읽기만 무프롬프트, 쓰기는 프롬프트 유지. "파괴적 작업은 명시 요청 시에만" 정책을 permission 레이어에서도 보존하는 장치이므로 쓰기 계열(create·update·move·archive·delete·restore·**revert**)을 allow에 넣지 않는다 (revert는 문서 본문·제목을 통째로 되돌리는 파괴적 쓰기다):

```json
"permissions": {
  "allow": [
    "Bash(outline doctor:*)",
    "Bash(outline search:*)",
    "Bash(outline read:*)",
    "Bash(outline list:*)",
    "Bash(outline tree:*)",
    "Bash(outline collections:*)",
    "Bash(outline revisions:*)"
  ]
}
```

## 검증 절차

패키징 완료 판정 기준:

1. 매니페스트 JSON 유효성 (`python3 -m json.tool`) + name·경로 삼중 일치 (plugin.json ↔ marketplace.json ↔ 디렉토리명) + `claude plugin validate` (리포 루트·플러그인 디렉토리 각각) 통과.
2. `bin/outline` 실행권한 유지 확인 (`test -x`) — 패키징 중 권한이 벗겨지지 않았는지.
3. 스킬 정의: `skills/outline/SKILL.md` 실존, 첫 줄 `---`, frontmatter `name: outline`, `allowed-tools`가 읽기 7종만, CLI 참조 표의 명령·플래그가 `bin/outline --help`와 전부 일치.
4. 로컬 설치 리허설 명령 준비 (실행은 사용자 몫 — `/plugin` 명령은 대화형 세션에서만 가능):
   - `/plugin marketplace add <리포 로컬 절대경로>` → `/plugin install outline-wiki@sanggi-wjg` → 새 세션에서 에이전트 인식 + `/outline-wiki:outline` 노출 + `outline` PATH 확인 (`which outline`). 설치 없이 확인하려면 `claude --plugin-dir <플러그인 디렉토리>`.
5. 회사 정보 비내장 확인: 산출물 전역 grep으로 회사 위키 도메인·회사명·조직 URL·이메일이 0건인지 검증.
6. GitHub push 전 체크: 사설 리포 접근에 git 자격증명 필요, README에 토큰 안내 포함 여부.
