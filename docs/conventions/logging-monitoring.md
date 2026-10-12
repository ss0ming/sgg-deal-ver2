# [컨벤션] 로깅 & 모니터링

> **범위**: 로그 형식·수집·분산 트레이싱·모니터링/알림·로그 레벨
> **변경 시**: 이 문서를 단일 출처로 갱신합니다 ([AGENT_GUIDE 문서 동기화](../AGENT_GUIDE.md#문서-동기화-필수) 참고).

## 결정 사항

| 항목 | 결정 | 비고 |
|------|------|------|
| 앱 로그 형식 | 구조화 JSON 로깅 | |
| 로그 수집 | AWS CloudWatch Logs | |
| 분산 트레이싱 | Zipkin | 서비스 간 호출(Gateway → 각 서비스, Kafka 이벤트 흐름) 추적 |
| 모니터링 / 알림 | CloudWatch + Slack 알림 | 장애·임계치 초과 시 Slack 채널로 알림 ([external-apis.md](./external-apis.md)) |
| 로그 레벨 운영 기준 | DEBUG(개발) / INFO(운영) | |
| 요청 식별자 | `X-Request-ID` 수용·생성, trace id는 Zipkin과 연동 | |

## 규칙 / 적용

- 앱 로그는 구조화 JSON 형식으로 출력하고 CloudWatch Logs로 수집한다.
- 서비스 간 호출(동기 API·Kafka 이벤트)은 Zipkin으로 trace를 남겨, 장애 발생 시 어느 서비스에서 지연/실패가 발생했는지 추적한다.
- 운영 중 임계치(에러율, 지연시간 등) 초과는 CloudWatch 알람으로 감지하고 Slack으로 알린다.
- 로그 레벨은 개발 환경 DEBUG, 운영 환경 INFO로 운영한다.
- 요청 body, Authorization header, access/refresh token, 비밀번호 등 민감정보는 로그에 남기지 않는다.

## 관련 문서

- [인프라 & 배포 (CI/CD)](./infra-deploy.md)
- [외부 API & 데이터 연동](./external-apis.md) — Slack 알림 연동
