# [컨벤션] 문서화 & 협업 도구

> **범위**: 문서·API 명세·아키텍처 다이어그램·커뮤니케이션.
> **변경 시**: 이 문서를 단일 출처로 갱신합니다 ([AGENT_GUIDE 문서 동기화](../AGENT_GUIDE.md#문서-동기화-필수) 참고).

## 결정 사항

| 항목 | 결정 | 비고 |
|------|------|------|
| 코드 저장소 내 문서 | `docs/`(AGENT_GUIDE·conventions·specs) | |
| API 명세 관리 | springdoc-openapi(Swagger) 자동 생성 | |
| 아키텍처 다이어그램 | TODO | 도구 미정 (현재 README용 이미지 1개만 존재) |
| 알림/커뮤니케이션 | Slack | 운영 알림 수신([logging-monitoring.md](./logging-monitoring.md)) · 1인 프로젝트라 팀 커뮤니케이션 용도는 아님 |

## 규칙 / 적용

- API 명세는 springdoc-openapi(Swagger)가 자동 출처다. 서비스별로 `/swagger-ui.html` 등 경로를 통해 확인한다(정확한 경로는 [app-run-guide.md](../app-run-guide.md) 참고).
- 코드 저장소 내 문서 체계는 `docs/`(AGENT_GUIDE·conventions·specs)를 사용한다.

## 관련 문서

- [AGENT_GUIDE](../AGENT_GUIDE.md)
- [API 설계 & 응답 형식](./api-design.md)
