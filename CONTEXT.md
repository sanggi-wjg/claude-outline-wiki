# Outline Wiki 플러그인 — 구현 컨텍스트

> 2026-07-05, fitpetmall-backend-v4 세션에서 인터뷰·검증을 거쳐 확정한 설계 핸드오프 문서.
> 이 문서만으로 구현이 가능하도록 모든 결정 사항과 검증된 사실을 담았다.

## 1. 목표

회사 Outline 위키(https://wiki.fitpet.kr)의 문서 **검색·조회·작성·수정·관리**를 수행하는
Claude Code 서브에이전트를 **플러그인**으로 만들어, 사설 GitHub 리포를 통해 팀원들이
`/plugin install`로 설치해 쓸 수 있게 한다.

## 2. 확정된 설계 결정 (인터뷰 결과)

| 분기 | 결정 |
|---|---|
| 용도 | 읽기 참조 + 문서 작성/갱신 + 위키 관리 + 대화형 범용 (전체) |
| API 통합 | 자체 CLI 래퍼 스크립트 (에이전트는 Bash로 호출, 본문은 파일로 전달) |
| 배포 형태 | **Claude Code 플러그인** (agents/ + bin/ 번들, 사설 GitHub 리포) |
| 토큰 | macOS 키체인(`security`, 서비스명 `outline-token`) 기본 + `OUTLINE_API_TOKEN` env 오버라이드 |
| 베이스 URL | `https://wiki.fitpet.kr`를 스크립트 기본값으로 내장 + `OUTLINE_URL` env 오버라이드 |
| 안전장치 | 영구삭제 미노출. 삭제·아카이브·이동·기존 문서 수정은 사용자 명시 요청 시에만 |
| 출력 계약 | 질문 맞춤 답변 + 출처(제목·절대 URL·근거 섹션). 전문 반환은 명시 요청 시에만 |
| 호출 정책 | 읽기는 프로액티브(메인 에이전트가 정책·스펙 불확실 시 자동 위임), 쓰기는 명시적 |
| 생성 기본값 | draft(`publish=false`) + 컬렉션 미지정 시 생성하지 않고 후보 반환·확인 요청 |
| 모델 | `model` 필드 생략 → 세션 모델 상속(inherit이 공식 기본값) |
| 스크립트 구현 | Python 3 표준 라이브러리 단일 파일 (의존성 없음, urllib 사용 → 토큰이 프로세스 인자에 노출되지 않음) |

## 3. 리포 구조 (제안)

```
claude-plugins/                        # 마켓플레이스 리포 (사설)
├── .claude-plugin/
│   └── marketplace.json               # name: "fitpet", 플러그인 목록 (상대 경로 source)
├── plugins/
│   └── outline-wiki/
│       ├── .claude-plugin/
│       │   └── plugin.json            # name, description, version
│       ├── agents/
│       │   └── outline-wiki.md        # 에이전트 정의 (§5)
│       ├── bin/
│       │   └── outline                # Python CLI, chmod +x (§4)
│       └── README.md                  # 팀원 셋업 안내 (§6)
└── README.md
```

- 플러그인 `bin/`은 **Bash PATH에 자동 추가**됨 → 에이전트는 그냥 `outline <subcommand>`로 호출.
- `${CLAUDE_PLUGIN_ROOT}`는 hooks/MCP 설정에서만 치환되고 **에이전트 마크다운 본문에서는 동작하지 않음** → bin/ PATH 방식이 정답.
- marketplace.json의 정확한 스키마(동일 리포 내 상대 경로 source 등)는 구현 시
  https://code.claude.com/docs/en/plugin-marketplaces.md 에서 확인할 것.
- 마켓플레이스명 `fitpet` → 설치 명령이 `/plugin install outline-wiki@fitpet`이 됨.

## 4. CLI 스크립트 사양 (`bin/outline`)

### 서브커맨드

```
doctor                                                # auth.info — 토큰·계정 확인 (셋업 검증용)
search <query> [--collection <id>] [--limit N=10]     # documents.search
read   <docId|URL>                                    # documents.info — 메타 헤더 + 마크다운 본문
list   [--collection <id>] [--limit N=25]             # documents.list (updatedAt DESC)
tree   <collectionId>                                 # collections.documents — 들여쓰기 트리 + id
collections                                           # collections.list — id, 이름
create --title <t> [--collection <id>] [--parent <docId>] --body-file <f> [--publish]
update <docId> [--body-file <f>] [--title <t>] [--append]
move   <docId> [--collection <id>] [--parent <docId>]
archive <docId>
delete  <docId>                                       # 휴지통 이동만. permanent 미노출
restore <docId>
```

### 동작 규칙

- **인증 해석 순서**: `OUTLINE_API_TOKEN` env → `security find-generic-password -s outline-token -w`.
  둘 다 없으면 등록 방법을 stderr로 안내하고 exit 2.
- **URL 해석**: `OUTLINE_URL` env → 기본값 `https://wiki.fitpet.kr`.
- **API 형식**: 전부 `POST {base}/api/{endpoint}`, JSON body, `Authorization: Bearer`, timeout 30초.
- **에러 매핑** (stderr + non-zero exit, API 응답의 message 포함):
  - 401 → "토큰이 유효하지 않거나 만료. 키체인 outline-token 재등록 안내"
  - 403 → "권한 부족 (문서/컬렉션 권한 또는 API 키 스코프 확인)"
  - 404 → "문서/컬렉션 ID 확인"
  - 429 → "rate limit — 잠시 후 재시도"
- **컨텍스트 다이어트**: search 출력은 id·제목·절대 URL·매칭 스니펫(HTML 태그 제거, ~200자)만.
  read는 본문 앞에 메타데이터 한 줄 헤더(제목·id·collectionId·updatedAt·절대 URL).
  절대 URL 조합(응답의 상대 경로 + base URL)은 스크립트가 책임진다.
- **docId 인자**: 전체 URL도 허용 — 마지막 path segment 추출 (Outline API는 UUID와 urlId 모두 허용).
- **create**: 기본 `publish=false`. `--publish` 시 `--collection` 필수 검증.
- **update**: `--append`는 `--body-file` 필수 (documents.update의 append 파라미터 사용).
- **본문 전달**: 항상 `--body-file <파일>` — JSON 이스케이핑 문제 원천 차단.

## 5. 에이전트 정의 사양 (`agents/outline-wiki.md`)

### frontmatter

```yaml
name: outline-wiki
description: 회사 Outline 위키(https://wiki.fitpet.kr)의 문서 검색·조회·작성·수정·관리를 담당한다.
  개발 중 비즈니스 정책·기획 배경·스펙·가이드 등 위키에 있을 법한 정보가 불확실할 때
  proactively 사용해 검색·인용한다. 문서 생성·수정·이동·아카이브·삭제는 사용자가
  명시적으로 요청한 경우에만 수행한다.
tools: Bash, Read, Write
# model 생략 → 세션 모델 상속
```

### 본문에 담을 행동 규칙

- 모든 위키 작업은 `outline` CLI로만 수행. **원시 curl로 API 직접 호출 금지** (스크립트가 없거나 실패하면 그 사실을 보고).
- **읽기**: search → 관련 문서 read → 위임받은 질문에 필요한 내용만 발췌·정리.
  반환 형식은 `## 답변` + `## 출처`(문서 제목, 절대 URL, 근거 섹션). 원문 전체는 명시 요청 시에만.
  검색이 빗나가면 키워드를 바꿔 2~3회 재시도하고 tree/list로 훑는다. 그래도 없으면 "없다"고 보고 — 추측으로 지어내지 않는다.
- **쓰기** (위임 프롬프트에 명시된 경우에만):
  - 본문은 Write 도구로 임시 파일에 작성 후 `--body-file`로 전달.
  - 생성은 기본 draft. "바로 발행" 명시 시에만 `--publish`.
  - 컬렉션 미지정 시 생성하지 말고 `collections` 목록 + 추천 후보를 반환해 확인 요청.
  - 수정 전 반드시 read로 현재 본문 확인, 수행 후 변경 요약 보고. 읽기 위임 중 문제를 발견해도 고치지 않고 보고만.
  - delete는 휴지통 이동임을 보고에 명시 (restore로 복구 가능).
- 문서 언어는 한국어 (요청 시 예외).
- 코드 분석은 하지 않는다 — 문서화할 내용은 위임 프롬프트로 받거나 지정된 로컬 파일을 Read로 읽는다.
- 401 발생 시 `outline doctor`로 진단하고 토큰 재등록 방법을 보고에 포함.
- 쓰기 작업 보고 형식: 수행 작업 / 대상 문서(제목·URL) / 변경 요약 / 후속 필요 사항(예: draft 발행 필요).

## 6. 팀원 셋업 절차 (플러그인 README에 넣을 내용)

```bash
# 1. 마켓플레이스 등록 + 설치 (Claude Code 안에서)
/plugin marketplace add https://github.com/FitpetKorea/claude-plugins.git
/plugin install outline-wiki@fitpet

# 2. Outline API 토큰 발급 (위키 설정 → API Tokens) 후 키체인 등록 (터미널에서)
security add-generic-password -s outline-token -a "$USER" -w "<API 토큰>"

# 3. 검증
outline doctor
```

- macOS가 아니면 `OUTLINE_API_TOKEN` 환경변수로 대체.
- 자동 업데이트를 원하면 `GH_TOKEN` env 필요 (사설 리포).
- **permission 권장 규칙** (읽기만 무프롬프트, 쓰기는 프롬프트 유지 — "파괴적 작업은 명시 요청 시에만" 정책을 permission 레이어에서도 보존):
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

## 7. 구현·검증 순서

1. 리포 골격 + marketplace.json/plugin.json 작성 → 검증: 스키마를 공식 문서와 대조
2. `bin/outline` 구현 → 검증: `outline --help`, 토큰 없이 `doctor`(안내 메시지 + exit 2), 더미 토큰으로 `doctor`(401 매핑 확인)
3. `agents/outline-wiki.md` 작성
4. **로컬 마켓플레이스로 설치 검증**: `/plugin marketplace add <로컬 경로>` → `/plugin install` → 에이전트 인식·`outline` PATH 확인
5. 실토큰 등록 후 스모크: `doctor` → `collections` → `search` → `read` → draft `create`/`delete`/`restore` 왕복
6. GitHub 사설 리포 push → 팀원 1명 설치 리허설

## 8. 검증된 사실 (2026-07-05 기준, 재검증 불필요)

- `https://wiki.fitpet.kr` HTTP 200, `/api/auth.info`가 표준 Outline 401 JSON 반환 → API 활성화된 정상 인스턴스
- Claude Code 샌드박스 Bash에서 `security`(키체인) 접근 가능 (실측)
- macOS python3 = 3.14.0
- 플러그인은 `agents/`(서브에이전트)와 `bin/`(PATH 자동 추가) 번들 지원 — https://code.claude.com/docs/en/plugins.md
- `${CLAUDE_PLUGIN_ROOT}`는 hooks/MCP 설정 한정, 마크다운 본문 미치환 — https://code.claude.com/docs/en/plugin-marketplaces.md
- 사설 GitHub 리포 마켓플레이스 지원 (git 자격증명 사용, auto-update는 `GH_TOKEN`/`GITHUB_TOKEN`)
- 서브에이전트 `model` 생략 = inherit, description의 "proactively" 문구는 공식 자동 위임 메커니즘 — https://code.claude.com/docs/en/sub-agents.md

## 9. 알려진 한계 / 의도적 제외 (v1)

- **lost-update**: documents.update는 전체 본문 교체이고 revision 충돌 감지 없음 → "수정 전 read + 변경 요약 보고"로 완화
- **첨부파일/이미지 업로드 미지원** (attachments.create는 multipart라 v1 제외)
- **대용량 문서 read**는 서브에이전트 컨텍스트를 소모 — 문제 되면 `--outfile` 옵션 추가 (v1 제외)
- dry-run/diff 프리뷰, 로컬 감사 로그, 자동 페이지네이션, 캐싱 — 과설계로 판단해 제외 (Outline 자체의 리비전 히스토리·휴지통이 감사 추적 제공)
- API 키 스코프: 최신 Outline은 키에 스코프 지정 가능 — 키 발급 시 지원 여부 확인 권장 (쓰기 용도 포함이므로 전체 스코프 사용 예정)
