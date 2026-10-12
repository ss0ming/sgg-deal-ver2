---
name: commit
description: SSG-DEAL에서 브랜치를 만들거나 커밋할 때 따르는 절차. 보호 브랜치(main) 직접 작업 금지(평소엔 develop에 직접 커밋·푸시), 브랜치 네이밍 type/설명(이슈번호 선택), 커밋 메시지 Conventional Commits를 지켜 HOOK 차단에 걸리지 않게 한다. "커밋해", "브랜치 만들어", "commit" 등 요청 시 사용한다.
---

# commit — 브랜치·커밋 규약 절차

SSG-DEAL의 형상 관리 규약을 따라 브랜치를 만들고 커밋한다. 규약의 단일 출처는
[docs/conventions/scm-collaboration.md](../../../docs/conventions/scm-collaboration.md)이며,
아래 규칙은 HOOK(`.claude/hooks/`)으로 자동 검사되므로 위반하면 커밋·푸시가 차단된다.

## 1. 보호 브랜치 확인

현재 브랜치를 확인한다. `main`만 보호 브랜치라 **직접 커밋·푸시할 수 없다**. 평소 작업은 `develop`에서 바로 커밋·푸시한다.

```bash
git rev-parse --abbrev-ref HEAD
```

`main`에 있으면 `develop`으로 전환하거나(또는 필요 시 기능 브랜치를 새로 만든다, 아래 2).

## 2. 브랜치 네이밍 (기능 브랜치를 쓸 때만)

형식: `<type>/<설명>` (kebab-case). 이슈 번호는 선택사항이며 붙이고 싶으면 앞에 붙인다.

- `type`: `feat` `fix` `docs` `style` `refactor` `perf` `test` `build` `ci` `chore` `revert`
- `설명`: 소문자·하이픈 (kebab-case)

```bash
git switch -c feat/add-login
```

예) `feat/add-login`, `fix/pagination-bug`, `feat/123-add-login`(이슈 번호 붙이는 경우)

## 3. 커밋 메시지 (Conventional Commits)

형식: `<type>(scope)?: <설명>` — 첫 줄(헤더)이 검사 대상이다. 이슈 번호 없이 바로 설명을 쓴다.

```bash
git add <변경 파일>
git commit -m "feat: 00기능 만들기"
git push origin develop
```

예) `feat: 00기능 만들기`, `fix(api): 페이지네이션 오류 수정`, `docs: 컨벤션 문서 추가`

## 4. main 승격 (필요할 때만)

평소엔 `develop`에 직접 푸시하는 것으로 끝난다. `main`으로 릴리즈할 시점이 되면 그때 병합 방식을 정해 `develop → main`을 반영한다([scm-collaboration](../../../docs/conventions/scm-collaboration.md) 참고).

## 참고

- 규약 출처: [docs/conventions/scm-collaboration.md](../../../docs/conventions/scm-collaboration.md)
- 자동 강제(HOOK·SKILL) 개요: [docs/AGENT_GUIDE.md](../../../docs/AGENT_GUIDE.md#강제-규칙-skill--hook)
