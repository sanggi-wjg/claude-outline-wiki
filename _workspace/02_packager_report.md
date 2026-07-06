# 02 Packager Report

- 최신 mode: fix (회사 정보 비내장 결정 반영)
- repo_root: /Users/raynor/vscode_workspace/claude-outline-wiki/claude-plugins
- 작성 명세: .claude/skills/plugin-packaging/SKILL.md (fix 전 재독 — 회사 정보 비내장 규칙, 마켓플레이스명 team, OUTLINE_URL 필수 셋업 반영본)
- 일시: 2026-07-05

## fix 라운드: 회사 식별 정보 스크럽

### 수정한 파일 (절대 경로)

- /Users/raynor/vscode_workspace/claude-outline-wiki/claude-plugins/.claude-plugin/marketplace.json
- /Users/raynor/vscode_workspace/claude-outline-wiki/claude-plugins/plugins/outline-wiki/.claude-plugin/plugin.json
- /Users/raynor/vscode_workspace/claude-outline-wiki/claude-plugins/plugins/outline-wiki/agents/outline-wiki.md
- /Users/raynor/vscode_workspace/claude-outline-wiki/claude-plugins/README.md

### 변경 요약

1. marketplace.json
   - name: "fitpet" → "team"
   - owner: { name: "Fitpet Korea", email: "dev@fitpet.co.kr" } → { name: "outline-wiki maintainers" } (이메일 제거)
   - 최상위 description: "Fitpet 사내 …" → "outline-wiki 플러그인 마켓플레이스" (회사명 제거)
   - 플러그인 항목 description: 위키 URL(https://wiki.fitpet.kr)·"회사" 제거 → "사내 Outline 위키의 …"
2. plugin.json
   - description: 위키 URL 제거 → "사내 Outline 위키의 …"
   - author: { name: "Fitpet Korea", email: … } → { name: "outline-wiki maintainers" } (이메일 제거)
   - version 0.1.0 유지
3. agents/outline-wiki.md
   - frontmatter description: 스킬 갱신본 "사내 Outline 위키의 문서 검색…"으로 교체(위키 URL 제거, proactively 문구·tools 유지)
   - 본문 첫 단락: 위키 URL 제거, "위키 주소는 OUTLINE_URL 환경변수로 공급된다" 문구로 대체
   - model 필드 여전히 없음(inherit), 본문 규칙 문구는 그대로
4. README.md (루트 단일)
   - 리포 URL → `<사설 리포 git URL>` 플레이스홀더
   - 설치 명령 @fitpet → @team, 제목/문구의 "Fitpet 사내" 표기 제거
   - OUTLINE_URL을 셋업 2단계(필수, 기본값 없음)로 삽입 → 셋업 4단계 구성
   - 환경별 설정: OUTLINE_URL을 "오버라이드"가 아닌 "필수(기본값 없음)"로 재기술
   - 리포 구조 트리의 name을 "team"으로 갱신
   - permission 블록·키체인 명령·리포 구조 트리 유지(회사 정보 아님)

### 유지/미복원 (지시대로)

- plugins/outline-wiki/README.md: 삭제 상태 유지. 재빌드 시 복원 금지 규칙 준수 — 생성하지 않음.
- plugins/outline-wiki/bin/outline: cli-developer가 스크럽 중. 건드리지 않음. test -x PASS(실행권한 유지, -rwxr-xr-x).

## 검증 결과

### JSON 유효성 (python3 -m json.tool)
- marketplace.json : VALID
- plugin.json      : VALID

### name·경로 삼중 일치
- marketplace.name = "team" (→ @team)
- entry.name == plugin.json.name == 디렉토리명 == "outline-wiki" (일치)
- source "./plugins/outline-wiki" 실존
- owner.email 없음 / author.email 없음 (확인)

### 회사 정보 스크럽 grep (담당 산출물 4종, 파일별 명시 grep)
주의: 셸이 zsh라 unquoted 변수의 word-split이 없어 초기 일괄 grep이 무효(단일 비존재 경로로 전달됨)였다. 파일별 명시 경로로 재검증함.
- 대상: marketplace.json / plugin.json / agents/outline-wiki.md / README.md
- 패턴 fitpet · co.kr · wiki.fitpet · FitpetKorea · @fitpet : 전부 0건
- grep -ic "fitpet" 파일별: 모두 0
- 결과: 회사 마커 0건 (clean)

### 부가 확인
- agent frontmatter model 필드 없음(inherit) — 확인
- README @team 2곳, OUTLINE_URL 2곳, `<사설 리포 git URL>` 플레이스홀더 1곳 존재
- 플러그인 README 미존재(복원 안 함)

## 공식 문서 스키마 대조
- build 라운드에서 두 공식 문서(plugin-marketplaces.md, plugins.md) WebFetch 대조 완료. fix는 스키마 필드 구조 변경이 아닌 값 스크럽이라 재대조 불필요.
- owner 필수 필드 유지(공식 문서 규정). 스킬 갱신본도 "owner 필수"를 반영함 → 이전 상충 사항 해소됨.

## 미확정 / 후속
- 로컬 설치 리허설(/plugin marketplace add <로컬 경로> → install @team), `claude plugin validate .`는 대화형/사용자 몫 — 미실행, 명령은 README에 준비됨.
- 실제 배포 시 `<사설 리포 git URL>`·`OUTLINE_URL` 값은 사용자가 각자 환경에서 공급.
