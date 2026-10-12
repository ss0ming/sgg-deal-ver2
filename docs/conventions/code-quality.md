# [컨벤션] 코드 품질 & 컨벤션

> **범위**: 린트·포맷·커밋 전 자동 검사·테스트 정책.
> **변경 시**: 이 문서를 단일 출처로 갱신합니다 ([AGENT_GUIDE 문서 동기화](../AGENT_GUIDE.md#문서-동기화-필수) 참고).

## 결정 사항

| 항목 | 결정 | 비고 |
|------|------|------|
| Kotlin 포맷터/린터 | ktlint | Gradle 플러그인으로 적용 (적용 시점 TODO) |
| 정적 분석 | detekt | 도입 여부·규칙셋 TODO |
| 빌드 도구 | Gradle (Kotlin DSL) | |
| 커밋 전 자동 검사 | `.claude/hooks/`(브랜치명·커밋 메시지) + TODO(린트 훅) | 린트를 pre-commit에 포함할지 미정 |
| PR 자동 검사(CI) | TODO | GitHub Actions에서 `./gradlew build`·테스트 실행 여부/범위 미정 |
| 테스트 정책 | **작성·유지 필수** | 기능 변경에는 회귀 테스트를 함께 넣는다 |
| 테스트 도구 | JUnit5 + MockK | Kotlin 환경 표준 조합 |

## 규칙 / 적용

- `[강제]` **브랜치 네이밍 · 커밋 메시지** — `.claude/hooks/check_branch_name.py`, `check_commit_message.py`로 Claude Code PreToolUse 시점에 자동 검사된다. 로직 단일 출처는 `.claude/hooks/_rules.py`. 상세: [scm-collaboration.md](./scm-collaboration.md).
- 포맷/린트는 ktlint, 정적 분석은 detekt를 사용한다(Gradle 모듈 적용 범위·CI 연동 시점은 TODO).
- 테스트는 **작성·유지 대상**이다. 기능을 바꾸면 그 동작을 고정하는 테스트를 함께 넣는다. 특히 **버그를 고칠 때는 그 버그를 재현하는 테스트**를 먼저 추가한다.
- 이미 실패 중인 테스트를 발견하면 "원래 깨져 있었다"로 넘기지 않는다 — 실제 결함을 가리고 있을 수 있다.

## 관련 문서

- [형상 관리 & 협업](./scm-collaboration.md)
- [AGENT_GUIDE](../AGENT_GUIDE.md)
