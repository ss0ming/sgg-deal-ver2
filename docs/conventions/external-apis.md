# [컨벤션] 외부 API & 데이터 연동

> **범위**: 결제·알림 등 외부 서비스 연동, API 키 관리
> **변경 시**: 이 문서를 단일 출처로 갱신합니다 ([AGENT_GUIDE 문서 동기화](../AGENT_GUIDE.md#문서-동기화-필수) 참고).

## 결정 사항

| 항목 | 결정 | 비고 |
|------|------|------|
| 결제 | Toss Payments | `payment_service`에서 연동 |
| 운영 알림 | Slack (Incoming Webhook) | 장애·배포 알림 ([logging-monitoring.md](./logging-monitoring.md)) |
| API 키 / 시크릿 관리 | local `.env` / prod AWS SSM Parameter Store | SOT: [auth-security.md](./auth-security.md#4-시크릿-저장소) |

## 규칙 / 적용

- 결제 연동은 `payment_service` 내부의 연동 모듈 뒤에 둬서, 결제사(Toss Payments)를 교체하더라도 인터페이스가 바뀌지 않도록 한다(모듈 경로는 TODO).
- 운영 알림(배포 결과, 장애 감지)은 Slack Webhook으로 전송한다.
- 외부 API 키(`TOSS_SECRET_KEY`, `TOSS_CLIENT_KEY`, `SLACK_WEBHOOK_URL` 등)는 코드/깃에 두지 않고, [auth-security.md](./auth-security.md#4-시크릿-저장소)의 시크릿 저장소를 따른다.

## 관련 문서

- [인증 & 보안](./auth-security.md)
- [로깅 & 모니터링](./logging-monitoring.md)
