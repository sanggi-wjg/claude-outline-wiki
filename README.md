# claude-plugins (sanggi-wjg 마켓플레이스)

사내 Claude Code 플러그인을 배포하는 사설 마켓플레이스 리포다. 마켓플레이스 이름은 `sanggi-wjg`이며, 설치 시 `<plugin>@sanggi-wjg` 형태로 참조한다.

## 제공 플러그인

| 플러그인 | 설명 |
| :--- | :--- |
| `outline-wiki` | 사내 Outline 위키의 문서 검색·조회·작성·수정·관리를 담당하는 에이전트 + `outline` CLI |

## outline-wiki 셋업

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

플러그인이 활성화되면 `bin/`의 `outline` 실행 파일이 Bash 도구의 `PATH`에 자동으로 추가된다. 에이전트는 `outline <subcommand>` 형태로만 CLI를 호출한다.

### 환경별 설정

- **위키 주소(`OUTLINE_URL`)**: 기본값이 없다. 셋업 2단계처럼 반드시 설정해야 하며, 두 가지 방법이 있다.
  - **셸 프로필 `export`**: 터미널과 Claude Code 세션 모두에 적용된다.
  - **`~/.claude/settings.json`의 `env`**: `{ "env": { "OUTLINE_URL": "https://<위키 도메인>" } }` 형태로 추가한다. settings.json `env`는 Claude Code 세션(Bash 도구)에만 주입되므로, 터미널에서 `outline`을 직접 실행하려면 셸 프로필 `export`를 병행해야 한다. 둘 다 설정해도 무방하다.
- **macOS가 아닌 경우**: 키체인 대신 `OUTLINE_API_TOKEN` 환경변수로 토큰을 지정한다.
- **자동 업데이트**: 이 마켓플레이스는 사설 리포이므로, 백그라운드 자동 업데이트를 원하면 `GH_TOKEN`(또는 `GITHUB_TOKEN`) 환경변수에 리포 read 권한이 있는 토큰을 설정한다. 수동 설치·업데이트는 기존 git 자격증명(`gh auth login` 등)을 그대로 사용한다.

### permission 권장 규칙

"파괴적 작업은 사용자가 명시적으로 요청한 경우에만" 정책을 permission 레이어에서도 보존하기 위해, **읽기 계열만 무프롬프트로 허용하고 쓰기 계열은 프롬프트를 유지**한다. 쓰기 계열(`create`·`update`·`move`·`archive`·`delete`·`restore`)은 절대 `allow`에 넣지 않는다.

`.claude/settings.json`(또는 프로젝트/사용자 settings)에 아래를 추가한다:

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

### 설치 후 확인

새 세션에서:

- `/context`의 Custom Agents에 `outline-wiki`가 보이는지 확인한다.
- `which outline`으로 CLI가 `PATH`에 등록됐는지 확인한다.
- `outline doctor`로 토큰·연결 상태를 진단한다.

## 리포 구조

```
claude-plugins/
├── .claude-plugin/
│   └── marketplace.json        # name: "sanggi-wjg", 플러그인 목록
├── plugins/
│   └── outline-wiki/
│       ├── .claude-plugin/
│       │   └── plugin.json     # name, description, version
│       ├── agents/
│       │   └── outline-wiki.md # 배포용 에이전트 정의
│       └── bin/
│           └── outline         # Python CLI
└── README.md                   # 마켓플레이스 안내 + 셋업 (이 파일)
```

## 로컬 검증

배포 전 로컬에서 마켓플레이스 유효성을 확인한다:

```bash
claude plugin validate .
```
