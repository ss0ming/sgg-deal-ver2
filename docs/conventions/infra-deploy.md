# [컨벤션] 인프라 & 배포 (CI/CD)

> **범위**: IaC·로컬 환경·운영 컴퓨팅·환경 분리·CI/CD·레지스트리·도메인/HTTPS
> **변경 시**: 이 문서를 단일 출처로 갱신합니다 ([AGENT_GUIDE 문서 동기화](../AGENT_GUIDE.md#문서-동기화-필수) 참고).

## 결정 사항

| 항목 | 결정 | 비고 |
|------|------|------|
| 로컬 개발 환경 | Docker Compose | 1차로 MySQL만 구성(`docker-compose.yml`). 컨테이너 1개 + 서비스별 스키마 분리(`ssgdeal_{service}`), Redis·Kafka는 추후 추가 |
| IaC 도구 | Terraform | `infra/modules/` + `infra/envs/{env}/` |
| 운영 컴퓨팅 런타임 | AWS ECS (Fargate) | 서버 패치·용량 관리 없이 운영. 실제 운영 배포 시점은 미정(로컬 Docker Compose로만 검증할 수도 있음) |
| 환경 분리 | `local` / `prod` + 필요 시 생성하는 `loadtest` | `dev`/`staging` 같은 상시 환경은 두지 않음 |
| 컨테이너 레지스트리 | AWS ECR | |
| 도메인 / HTTPS | Route 53 + ACM, ALB에서 TLS 종료 | `api.{domain}` → ALB → Gateway |
| 모니터링 | AWS CloudWatch | [logging-monitoring.md](./logging-monitoring.md) 참고 |
| 분산 트레이싱 | Zipkin | 서비스 간 호출 추적 |
| CI/CD | GitHub Actions | PR: 빌드·테스트·ktlint / main 머지: 변경된 서비스만 이미지 빌드→ECR 푸시→배포 |
| 목표 성능 지표 | 동시 접속 최대 3만 명(평균 5천) / TPS 200 | 이벤트성 특가 프로모션 트래픽 벤치마크 기준. `loadtest` 환경 검증 기준값 |

## 규칙 / 적용

### 1. IaC 도구 — Terraform

```text
infra/
├── modules/          # vpc, ecs, rds, elasticache, msk, alb ...
└── envs/
    ├── prod/
    └── loadtest/
```

- 콘솔에서 리소스를 직접 생성·수정하지 않는다. 예외적으로 수정했다면 즉시 Terraform에 반영한다.
- State: S3 backend + 잠금 사용. state 파일은 저장소에 커밋하지 않는다.
- 모든 리소스에 태그를 붙인다: `Project=ssg-deal`, `Env={env}`, `ManagedBy=terraform`.
- `terraform apply` 전 `terraform plan` 결과를 확인한다.
- 변수 중 시크릿은 `.tfvars`에 쓰지 않고 SSM에서 참조한다([auth-security.md](./auth-security.md)의 시크릿 저장소).
- Provider, Terraform 버전을 고정한다.

**이유**: 국내 채용 시장에서 가장 많이 쓰이는 IaC 도구다. 부하 테스트 환경을 필요할 때만 재현하려면 인프라가 코드로 정의되어 있어야 한다. 태그로 환경별 비용을 추적한다.

### 2. 환경 분리

| 환경 | 용도 | 실행 방식 |
|------|------|-----------|
| `local` | 개발 | docker compose (MySQL, Redis, Kafka) |
| `prod` | 실제 배포 | AWS, 상시 운영 |
| `loadtest` | 부하 테스트 | AWS, 테스트 시에만 생성 후 `terraform destroy` |

- Spring profile 이름은 환경 이름과 동일하게 맞춘다(`local`, `prod`, `loadtest`).
- 환경별 설정 차이는 `application-{profile}.yml`과 환경 변수로만 둔다. 코드에서 profile 분기 금지.
- `loadtest`는 `prod`와 같은 모듈·스펙으로 구성한다(결과를 prod 성능의 근거로 쓰기 위해, 위 표의 목표 성능 지표 참고).
- 부하 테스트 종료 후 결과(TPS, p95, 에러율)를 기록하고 `loadtest` 환경을 즉시 삭제한다.
- `dev`/`staging` 상시 환경은 두지 않는다.

**비용 주의**

- NAT Gateway는 시간당 과금이 크다. 꼭 필요한 경우가 아니면 퍼블릭 서브넷 + 보안 그룹으로 대체 검토.
- RDS, ElastiCache, MSK는 사용하지 않는 기간에 중지 또는 삭제한다.
- AWS Budgets 알림을 설정한다(월 한도: TODO).

**이유**: 개인 프로젝트에서 상시 환경을 여러 개 유지하면 비용이 감당되지 않는다. 부하 테스트는 prod와 동일 조건이어야 의미가 있으므로, IaC로 동일 스펙을 일시적으로 재현한다.

### 3. 도메인 / HTTPS

Route 53으로 도메인을 관리하고, ACM 인증서를 ALB에 연결해 TLS를 ALB에서 종료한다.

- 도메인 구성: `api.{domain}` → ALB → Gateway. 내부 서비스는 도메인을 부여하지 않는다.
- ALB 리스너: 443(ACM 인증서, Gateway 대상 그룹으로 전달), 80(443으로 리다이렉트).
- ALB → Gateway, Gateway → 내부 서비스 구간은 HTTP(VPC 내부).
- 내부 서비스 보안 그룹은 Gateway 보안 그룹에서 온 트래픽만 허용한다.
- Toss Payments 웹훅 URL: `https://api.{domain}/payments/webhook`.
- Slack 알림 등 외부 연동 URL 변경 시 이 문서에 함께 기록한다.

**이유**: ACM 인증서는 무료이고 자동 갱신된다. TLS를 ALB에서 종료하면 애플리케이션에서 인증서를 관리하지 않아도 된다. Toss Payments 웹훅은 공개 HTTPS 엔드포인트가 필요하다.

### 4. CI/CD

- GitHub Actions를 사용한다.
- PR: 빌드, 테스트, ktlint 검사.
- main 머지: 변경된 서비스만 이미지 빌드 → ECR 푸시 → 배포.
- 이미지 태그: git commit SHA. `latest` 태그로 배포하지 않는다.
- 배포 대상 런타임은 ECS(Fargate)다. 다만 실제로 AWS에 올릴지, 로컬 Docker Compose 검증까지만 할지는 아직 미정이므로 Terraform ECS 모듈은 실제 배포를 결정한 뒤 작성한다.

## 관련 문서

- [로깅 & 모니터링](./logging-monitoring.md)
- [SCM & 협업](./scm-collaboration.md) — 브랜치 전략
- [인증 & 보안](./auth-security.md) — 시크릿 관리(SSM)
