# 팀 컨벤션 (conventions)

SSG-DEAL 프로젝트에서 합의한 **기술 스택·규약·프로세스 결정**을 영역별로 나눠 관리합니다.
한 문서는 한 영역(책임)만 담고, 그 영역의 **결정 사항 표가 단일 출처(SOT)**입니다. README·다른 문서는 여기를 링크로 참조합니다.

새 문서를 추가하거나 갱신할 때는 [`docs/template/conventions.md`](../template/conventions.md) 형식을 따르고, [AGENT_GUIDE 문서 동기화](../AGENT_GUIDE.md#문서-동기화-필수) 규칙을 지킵니다.

## 영역 트리

아래에서 필요한 영역을 타고 들어가세요.

```text
docs/conventions/
├── scm-collaboration.md     # 형상 관리 & 협업 — GitHub·멀티모듈 모노레포·브랜치·커밋·PR·이슈
├── backend.md               # 백엔드 스택 — Kotlin·Spring Boot·Gradle 멀티모듈·JPA·QueryDSL
├── database.md              # DB & 모델링 — MySQL·Redis·서비스별 DB·네이밍
├── api-design.md            # API 설계 & 응답 형식 — envelope·/api/v1·페이지네이션
├── auth-security.md         # 인증 & 보안 — JWT·Gateway Passport Token·시크릿 관리
├── infra-deploy.md          # 인프라 & 배포 — Docker·AWS ECS/ECR·Naver Cloud·GitHub Actions
├── logging-monitoring.md    # 로깅 & 모니터링 — 구조화 로깅·Zipkin·CloudWatch·Slack 알림
├── code-quality.md          # 코드 품질 & 컨벤션 — ktlint/detekt·테스트 정책
├── docs-tools.md            # 문서화 & 협업 도구 — Swagger/OpenAPI·Notion·Slack
├── external-apis.md         # 외부 API & 데이터 연동 — Slack Webhook·Toss Payments
└── release.md                # 배포 릴리즈 — 버전 관리·배포 전략(서버, 스토어 없음)
```

## 자동 강제되는 규약

일부 규약(보호 브랜치, 커밋 메시지, 브랜치 네이밍)은 **HOOK과 SKILL로 자동 강제**됩니다. 무엇이 어떻게 강제되는지는 [AGENT_GUIDE 강제 규칙](../AGENT_GUIDE.md#강제-규칙-skill--hook)을 참고하세요. 각 규약의 올바른 값은 위 영역 문서가 단일 출처입니다.
