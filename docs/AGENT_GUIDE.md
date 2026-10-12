# SSG-DEAL - AI 에이전트 공통 가이드

> 이 문서는 이 저장소에서 작업하는 **모든 AI 코딩 에이전트의 공통 진입 문서**입니다.
> Claude Code(`CLAUDE.md`), Codex(`AGENTS.md`) 등 어떤 도구를 쓰든 먼저 이 문서를 읽으세요.
> 각 도구의 진입 파일(`CLAUDE.md`, `AGENTS.md`)은 이 문서를 가리키도록 되어 있습니다.

SSG-DEAL은 **Kotlin + Spring Boot 기반 MSA(백엔드 전용)**를 Gradle 멀티모듈 모노레포로 관리하는 1인 포트폴리오 프로젝트입니다. 프론트엔드는 없으며, `gateway_service`가 외부에 노출되는 유일한 진입점입니다.

## 프로젝트 개요·구조

이 프로젝트의 상세 설명, **프로젝트 구조와 아키텍처**는 모두 [README.md](../README.md)에 정리되어 있습니다 (TODO: 루트 README.md 작성 필요 — [docs/template/README.md](template/README.md) 참고).
**작업 시작 전 반드시 README.md를 읽고 프로젝트 컨텍스트를 파악하세요.** README.md에는 다음이 담겨 있습니다.

- 어느 서비스 모듈이 어떤 책임을 갖는지 ([프로젝트 구조](../README.md#프로젝트-구조))
- 요청이 들어와 끝날 때까지의 흐름 ([실행 파이프라인](../README.md#실행-파이프라인--핵심-흐름) — Gateway → 각 서비스 경계 포함)
- 기능을 수정할 때 건드려야 할 서비스 ([주요 기능과 위치](../README.md#주요-기능과-위치))
- 코드 스타일과 자주 쓰는 라이브러리·패턴 ([코드 스타일](../README.md#코드-스타일))

## 팀 컨벤션 (docs/conventions/)

합의한 **기술 스택·규약·프로세스 결정**은 영역별 문서로 [`docs/conventions/`](conventions/)에 정리되어 있습니다. 작업 영역에 해당하는 규약을 먼저 확인하세요.

**진입점: [docs/conventions/README.md](conventions/README.md)** — 11개 영역(형상 관리·백엔드·DB·API·인증·인프라·로깅·코드 품질·문서·외부 API·배포 릴리즈)을 트리로 타고 들어갈 수 있습니다.

- 각 영역 문서의 `결정 사항` 표가 **그 영역의 단일 출처(SOT)**입니다. 다른 문서에 같은 결정을 중복 서술하지 말고 링크로 참조하세요.
- 아직 정해지지 않은 항목은 표에 `TODO`로 남아 있습니다. 해당 영역을 작업하기 전에 먼저 결정하고 이 표를 채우세요.
- 결정이 바뀌거나 새 영역이 생기면 해당 컨벤션 문서를 [`docs/template/conventions.md`](template/conventions.md) 형식으로 갱신/추가합니다(`doc-update` 스킬).

## 문서 동기화 (필수)

프로젝트에 **새로운 요소·구조·기능이 추가되거나 기존 동작이 바뀌면, 그 변경과 관련된 문서를 항상 함께 수정**하세요. 코드만 바꾸고 문서를 방치하면 다음 작업(사람·AI)이 낡은 정보를 신뢰하게 됩니다. 문서 갱신은 별도 작업이 아니라 **변경의 일부**로 취급합니다.

어떤 변경이 어떤 파일을 건드려야 하는지는 아래를 기준으로 판단하세요.

| 변경(행동) | 함께 수정할 문서/파일 |
|------|------|
| 서비스 모듈 추가/삭제/이동 (구조 변경) | `settings.gradle.kts` + [README.md](../README.md) `프로젝트 구조` 트리 |
| 요청/이벤트 흐름·서비스 간 계약(API·Kafka 토픽) 변경 | [README.md](../README.md) `실행 파이프라인` (필요 시 mermaid 포함) |
| 기능 추가 / 기능 담당 서비스 이동 | [README.md](../README.md) `주요 기능과 위치` 표 |
| 서비스 의존성 추가/제거 | 해당 `{service}_service/build.gradle.kts` + [README.md](../README.md) `패키지 의존성` |
| 코드 스타일·린팅 규칙 변경 | 해당 설정 파일 + [README.md](../README.md) `코드 스타일` |
| 실행 방법·환경변수·설정 변경 | [README.md](../README.md) `실행 방법` / [app-run-guide.md](app-run-guide.md) |
| 기술 스택·규약·프로세스 결정 변경/추가 | [`docs/conventions/`](conventions/)의 해당 영역 문서 (템플릿: `docs/template/conventions.md`) |
| 추가될·진행 중인 기능의 계획/설명/구현 | `docs/specs/{NNN}-{기능}/` (템플릿: `docs/template/specs/`, 아래 '기능 스펙' 참고) |

> 표에 없는 새로운 종류의 변경이라면, 가장 가까운 항목을 기준으로 삼되 **"이 변경을 모르는 사람이 무엇을 읽고 알아야 하는가?"**를 기준으로 갱신 대상을 정하세요.

## 작업 워크플로우 (기능 추가·리팩토링)

새로운 기능 추가나 리팩토링 등 작업을 **시작할 때**는 다음 순서를 지킵니다.

1. **문서 먼저 업데이트** — 구현보다 문서를 먼저 작성/갱신합니다.
   - 기능이면 [기능 스펙](#기능-스펙-docsspecs) 규약대로 `docs/specs/{NNN}-{기능}/`을 작성하고, 영향받는 README·AGENT_GUIDE 등을 [문서 동기화](#문서-동기화-필수) 표에 따라 갱신합니다.
   - `implementation.md`에는 먼저 **구현 규모(한 번에 구현 / 단위로 분할)**를 판단해 적고, 분할이 필요하면 구현 계획을 단위로 나눕니다(큰 작업은 단위로 쪼갠다).
   - 이 문서 작업은 항상 `doc-update` 스킬을 호출해 정합성·SOT·템플릿 검사를 거칩니다.
2. **(선택) 사용자 컨펌 대기** — 1인 프로젝트이므로 리뷰어는 없지만, 범위가 크거나 돌이키기 어려운 결정은 문서 업데이트안을 먼저 제시하고 본인 확인을 받은 뒤 구현을 시작하는 습관을 유지합니다.
3. **구현** — 문서(필요 시 컨펌)를 마친 뒤 구현을 시작합니다.
4. **계획이 바뀌면 문서부터** — 구현 도중 계획·범위·접근이 달라지면, 그 변경을 **코드에 반영하기 전에** 해당 spec 문서(`plan.md`·`implementation.md` 등)를 먼저 갱신합니다(`doc-update` 스킬). 문서를 갱신하지 않은 채 계획과 다르게 구현하지 않습니다.

### 브랜치·커밋

- `main`은 보호 브랜치라 직접 작업하지 않습니다. 평소 작업은 **`develop`에서 바로 커밋하고 푸시**합니다 (`main`에서 직접 작업하지 않습니다).
- 이슈 트래킹은 쓰지 않습니다(필요하면 GitHub Issues를 선택적으로 사용). 브랜치 네이밍·커밋 규약의 단일 출처는 [scm-collaboration](conventions/scm-collaboration.md)입니다.
- `develop`을 `main`으로 승격(릴리즈)할 시점이 되면 그때 병합 절차를 정해서 진행합니다 — 매 변경마다 PR을 강제하지 않습니다.

## 강제 규칙 (SKILL · HOOK)

핵심 규약 일부는 사람이 기억하지 않아도 **자동으로 강제**됩니다. 정의가 프로젝트 내부(`.claude/`)에 있습니다. 규약의 **올바른 값은 [`docs/conventions/`](conventions/)의 해당 문서가 단일 출처**이고, 아래 HOOK·SKILL은 그것을 강제·안내만 합니다.

### HOOK (자동 차단·검사)

검사 로직은 [`.claude/hooks/`](../.claude/hooks/)에 Python으로 **한 번만** 정의하고, Claude Code hooks와 git hooks 양쪽에서 호출합니다(단일 출처).

| 규칙 | 검사 스크립트 | Claude Code | git |
|------|--------------|-------------|-----|
| main 직접 커밋·푸시 금지 | `check_protected_branch.py` | PreToolUse(Bash) | pre-push |
| 커밋 메시지 Conventional Commits | `check_commit_message.py` | PreToolUse(Bash) | commit-msg |
| 브랜치 네이밍 `type/설명` (이슈번호 선택) | `check_branch_name.py` | PreToolUse(Bash) | — |
| 세션 시작 시 핵심 지침 리마인드 | `session_context.py` | SessionStart | — |

- **Claude Code hooks**: [`.claude/settings.json`](../.claude/settings.json) — 커밋되어 Claude Code로 작업할 때 공유됩니다. (개인 설정은 `.claude/settings.local.json`)
- **git hooks**: TODO — `.pre-commit-config.yaml` 또는 네이티브 git hooks 설치 방법을 정하고 1회 설치 절차를 문서화합니다.
- **CI**: TODO — GitHub Actions 테스트 워크플로를 추가하면 여기에 경로를 남깁니다. 로컬 훅만으로 검증됐다고 보지 마세요.

### SKILL (절차 안내)

- **`commit`** ([`.claude/skills/commit/`](../.claude/skills/commit/)): 브랜치 네이밍·Conventional Commits 규약에 맞춰 `develop`에 커밋/푸시하는 절차.
- **`dev-pr`** (글로벌): `main`으로 승격할 때(릴리즈 시점)만 선택적으로 사용하는 PR 생성 절차.
- **`doc-update`** (글로벌): 문서 변경 시 정합성·SOT·템플릿 검사.

## docs 문서 관리 원칙

`docs/`에 문서를 추가할 때는 다음 기준을 따릅니다.

- **모든 에이전트가 참고해야 할 영구적 내용**(구조·파이프라인·기능 등)이라면, 누구나 볼 수 있도록 README.md 등 `docs/` 최상위 공개 문서에 반영합니다.
- **추가될 예정이거나 진행 중인 개별 기능**의 계획·설명·구현 수준은 `docs/specs/` 아래 기능 단위 폴더로 관리합니다 (아래 '기능 스펙' 참고).
- 구현 수준은 `implementation.md`에 **현재 상태(계획 / 진행 중 / 완료 / 폐기 등)를 명시**하여, 읽는 사람이 지금 유효한 내용인지 판단할 수 있게 합니다.

## 기능 스펙 (docs/specs/)

새로운 기능이 **추가될 예정이거나 진행 중**이면, 그 기능의 계획·설명·구현 수준을 `docs/specs/` 아래 **기능 단위 폴더**로 관리합니다.

### 폴더 규칙

- 경로: `docs/specs/{NNN}-{feature-name}/`
- `{NNN}` = **5의 배수 prefix**(`000`, `005`, `010`, `015`, `020` …). 새 기능은 기존 폴더의 **최대 prefix + 5**로 매깁니다. (5 단위로 띄우는 이유: 나중에 순서 사이에 끼워 넣을 여지를 남기기 위함)
- `{feature-name}` = 기능을 알아볼 수 있는 짧은 이름 (kebab-case 권장)

### 필수 파일

각 기능 폴더에는 다음 3개 파일이 **반드시** 있어야 합니다.

| 파일 | 내용 |
|------|------|
| `plan.md` | 계획 — 배경/목적, 목표·비목표, 요구사항, 설계 개요, 의사결정(함께 논의·근거 필수), 영향 범위, 작업 단계 |
| `description.md` | 설명 — 기능이 무엇이고 어떻게 동작하는지, 주요 구성 요소와 위치 |
| `implementation.md` | 구현 수준 — **구현 규모 판단(한 번에/분할)**, (분할 시) 단위별 구현 계획, 현재 상태(계획/진행 중/완료/폐기), 구현·미구현 항목, 변경 이력 |

API·DB 설계가 있는 기능은 같은 폴더에 `api.md`·`database.md`를 선택적으로 추가할 수 있습니다.

### 템플릿 (필수 사용)

위 파일들은 **반드시 [`docs/template/specs/`](template/specs/)의 공통 템플릿을 복사해서 작성**하세요. 템플릿을 무시하고 임의 형식으로 쓰지 않습니다. 모든 기능 스펙이 같은 구조를 갖도록 유지해, 누구나 빠르게 찾아 읽을 수 있게 하기 위함입니다.

- `docs/template/specs/plan.md`
- `docs/template/specs/description.md`
- `docs/template/specs/implementation.md`
- `docs/template/specs/api.md` (선택)
- `docs/template/specs/database.md` (선택)

작성 예:

```text
docs/specs/
├── 000-{feature-name}/
│   ├── plan.md
│   ├── description.md
│   └── implementation.md
└── 005-{feature-name}/
    ├── plan.md
    ├── description.md
    └── implementation.md
```

> 구현이 진행되면 `implementation.md`의 상태와 변경 이력을 갱신하고, 기능이 안정화되어 영구 문서로 남길 내용은 README.md의 해당 섹션에도 반영하세요 (위 '문서 동기화' 참고).

## 도구별 설정

도구 전용 설정·스킬은 각 도구의 디렉토리에 둡니다. 공통 지침은 이 문서가 단일 출처입니다.

- Claude Code: `.claude/` (스킬·설정)
- Codex: `.codex/` (도구 설정) — TODO: 아직 생성되지 않음
