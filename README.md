# orca-parallel-teams

Orca ADE 에서 병렬 작업을 3계층(메인 세션 = 창구 → 트랙 워커 → 워커 안 이름 있는 agent team)으로
운영하는 절차를 담은 Claude Code 플러그인입니다. 스킬 하나(`orca-parallel-teams`)가 들어 있고, 프로젝트와 무관합니다.

> Paseo 용 판은 별도 저장소 [wowjangpro/paseo-parallel-teams](https://github.com/wowjangpro/paseo-parallel-teams) 에 있습니다(시험 운영 전 · 미검증).
> Orca 를 쓰면 이 저장소, Paseo 를 쓰면 그쪽입니다. 원칙 · 역할 · 기준 · 흐름은 같고, 도구 명령만 다릅니다.

## 무엇을 하나

- 창구(메인 세션)는 트랙을 나누고 워커를 띄우고 결재 · 머지만 합니다. 조사 · 검증 · 진단은 맡깁니다.
- 워커는 트랙 worktree 의 독립 Claude 세션이며, 역할 팀원(plan-architect · plan reviewer · analyzer · dev · tester · reviewer · security)을
  agent team 으로 구성해 팀원끼리 직접 주고받게 합니다.
- 코드를 바꾸는 트랙은 리뷰 다음에 보안 검수(security)를 반드시 거칩니다. 공통 점검 항목은 스킬 폴더의 `security-checklist.md` 에 있습니다.

## 상태

작성자 환경(Orca 1.4.205 · macOS)에서 읽기 전용 프로브 3회로 실측한 절차입니다. 스킬 안의 시간 · 횟수 · 동작 수치는
그 환경에서 잰 값이므로 처음 쓰는 환경에서는 아래 「처음 쓰기 전에」 의 시험을 한 번 돌려 다시 확인합니다.

## 필요한 것

- Claude Code (한 줄 설치 명령은 v2.1.292 이상. 그 밖의 명령은 `claude plugin --help` 로 확인)
- Orca ADE 와 `orca` 명령 — `orca status` 의 `runtimeReachable` 이 true 여야 합니다.
- agent team 실험 기능 켜기 — 환경변수 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 을 셸 환경이나 Claude Code `settings.json` 의 `env` 에 둡니다.
  팀원 생성은 대화형 세션에서만 되고, `-p` 비대화형 모드에서는 되지 않습니다(공식 문서).
- 무인 워커가 권한 확인 화면에서 멈추지 않도록 Orca 설정의 claude 기본 인자를 정해 둡니다(스킬 §0 3번).

## 설치

터미널에서:

- `claude plugin marketplace add wowjangpro/orca-parallel-teams`
- `claude plugin install orca-parallel-teams@orca-parallel-teams`

한 줄로 하려면(마켓플레이스 추가와 설치를 함께): `claude plugin install orca-parallel-teams --marketplace wowjangpro/orca-parallel-teams`

Claude Code 세션 안에서는 `/plugin marketplace add wowjangpro/orca-parallel-teams` 다음에
`/plugin install orca-parallel-teams@orca-parallel-teams` 입니다(설치 범위를 고르는 화면이 열립니다).
설치 범위는 기본이 사용자 전체(`--scope user`)이고, 저장소 하나에만 켜려면 `--scope project` 또는 `--scope local` 을 붙입니다.

GitHub 가 아닌 곳이나 로컬에서도 추가할 수 있습니다. `marketplace add` 에 전체 git 주소(예: `https://호스트/경로/저장소.git`, 특정 브랜치나 태그는 끝에 `#이름`)나
폴더 경로(상대경로는 `./` 로 시작)를 주면 됩니다. 비공개 저장소는 쓰는 PC 의 git 인증이 이미 되어 있어야 합니다.

설치하면 새 세션부터(또는 `/reload-plugins` 후) 스킬이 잡힙니다. 스킬 이름은 플러그인 이름이 앞에 붙은
`/orca-parallel-teams:orca-parallel-teams` 입니다. 설명이 맞는 상황에서는 Claude 가 스스로 불러오기도 합니다.

## 업데이트 · 제거

- 업데이트: `claude plugin marketplace update orca-parallel-teams` 다음에 `claude plugin update orca-parallel-teams@orca-parallel-teams`.
  새 사본은 `.claude-plugin/plugin.json` 의 `version` 이 바뀔 때만 받습니다. 이 마켓플레이스처럼 Anthropic 공식이 아닌 마켓플레이스는 자동 업데이트가 기본 꺼져 있으며,
  `/plugin` 의 Marketplaces 탭에서 켤 수 있습니다. 적용은 다음 시작 때 또는 `/reload-plugins` 입니다.
- 잠시 끄기: `claude plugin disable orca-parallel-teams` (켜 두면 스킬 설명이 매 세션 컨텍스트를 조금 차지합니다).
- 제거: `claude plugin uninstall orca-parallel-teams@orca-parallel-teams`. 마켓플레이스까지 지우려면 `claude plugin marketplace remove orca-parallel-teams`
  (그 마켓플레이스에서 설치한 플러그인도 함께 제거됩니다).

## 처음 쓰기 전에 — 읽기 전용 시험 1회

실제 작업에 쓰기 전에 파일을 바꾸지 않는 시험(스킬 §0 5번)을 한 번 돌립니다.

1. 워커 1명을 띄우고, 워커가 이름 있는 팀원 2명을 `model: sonnet` 으로 만들게 합니다.
2. 팀원끼리 직접 메시지를 한 번 주고받는지 봅니다.
3. 창구와 워커 사이의 `status` · `ask` · `worker_done` 교신이 실제로 도는지 봅니다.

「된다/안 된다」는 환경마다 다르므로 Orca 버전이 바뀐 뒤에도 같은 시험을 다시 합니다.

## 프로젝트별 보안 체크리스트

보안 검수는 두 층입니다.

- 공통: 스킬 폴더의 `security-checklist.md` (등급 기준 · 공통 점검 항목). 근거 문서(법령 · 인증기준)는 국내 서비스 기준이고 조문 대조 전입니다.
- 프로젝트별: `{프로젝트 루트}/docs/security-review/checklist.md` — 프로젝트 루트는 그 프로젝트의 최상위 `CLAUDE.md` 가 있는 디렉터리입니다.
  스택별 항목 · 등급 보정 · 보고서 경로 · 금지 동작을 여기에 둡니다. 겹치거나 다르면 프로젝트 쪽이 우선하고, 없으면 공통 기준으로만 검수합니다.

## 알아 둘 점

- 공통 체크리스트의 설치 위치는 플러그인 캐시 안이고 버전마다 경로가 바뀝니다. 창구가 워커 명세에 절대경로를 적을 때마다 현재 위치를 확인합니다
  (`claude plugin list --json` 의 `installPath` 가 플러그인 설치 위치입니다).
- 같은 스킬을 `~/.claude/skills/` 에 직접 복사해 둔 것이 있으면 중복되므로 한쪽만 씁니다. `skills/orca-parallel-teams/README.md` 는 그 직접 복사 방식의 안내입니다.
- 브라우저 검증 도구는 사용자 · 프로젝트가 지정한 것을 씁니다. 사용자 전역 `CLAUDE.md` 에 지정이 있는지 먼저 확인하세요.

## 저장소 구성과 로컬 확인

- `.claude-plugin/marketplace.json` — 마켓플레이스(이 저장소 자신이 플러그인 하나를 가리킵니다)
- `.claude-plugin/plugin.json` — 플러그인 매니페스트
- `skills/orca-parallel-teams/` — `SKILL.md` · `README.md` · `security-checklist.md`

저장소를 내려받은 폴더에서 `claude plugin validate .` 로 검증하고, 설치 없이 한 세션만 써 보려면 `claude --plugin-dir .` 입니다.

라이선스: 미정.
