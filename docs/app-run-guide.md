# 서비스 실행 가이드 (로컬)

SSG-DEAL은 Gradle 멀티모듈로 구성된 Kotlin/Spring Boot MSA입니다. 이 문서는 로컬에서 서비스를 **직접 띄워보기 위한 런북**입니다.

## 0. 사전 준비

- JDK 17 (Gradle toolchain이 `jvmToolchain(17)`로 고정합니다 — [backend.md](conventions/backend.md))
- 저장소 루트에 Gradle wrapper(`./gradlew`)가 포함되어 있어 별도 Gradle 설치가 필요 없습니다.

```bash
./gradlew build
```

## 1. 서비스 포트

| 서비스 | 포트 | 역할 |
|--------|------|------|
| `gateway_service` | 8080 | API Gateway — 외부 요청의 유일한 진입점 |
| `auth_service` | 8081 | 인증/인가 |
| `user_service` | 8082 | 회원 |
| `cart_service` | 8083 | 장바구니 |
| `order_service` | 8084 | 주문 |
| `payment_service` | 8085 | 결제 |
| `promotion_service` | 8086 | 프로모션/특가 |
| `notification_service` | 8087 | 알림 |

## 2. 서비스 하나만 실행

```bash
./gradlew :gateway_service:bootRun
```

다른 서비스도 같은 방식으로 `:{service}_service:bootRun`을 실행합니다.

> 현재 각 서비스는 독립적으로 기동만 되는 뼈대 상태입니다. MySQL·Redis·Kafka 연동, Gateway 라우팅 대상 서비스 실제 호출 등은 기능 구현에 따라 추가됩니다.

## 3. DB 띄우기 (Docker Compose)

MySQL 컨테이너 1개로 서비스별 스키마(`ssgdeal_auth`, `ssgdeal_user`, `ssgdeal_cart`, `ssgdeal_promotion`, `ssgdeal_order`, `ssgdeal_payment`, `ssgdeal_notification`)와 서비스 전용 유저가 한 번에 만들어집니다(초기화 스크립트: `docker/mysql/init/`, 테이블 DDL: [000-database-schema](specs/000-database-schema/database.md)).

```bash
cp .env.example .env   # 최초 1회
docker compose up -d --wait
```

확인:

```bash
docker exec ssgdeal-mysql mysql -uroot -proot -e "SHOW DATABASES LIKE 'ssgdeal_%';"
```

각 서비스는 `{service}_app` 전용 유저로 자기 스키마에만 접속합니다(예: `order_service` → `order_app`/`ssgdeal_order`). 접속 정보는 서비스별 `application.yml`에 설정합니다(아직 미설정 — Flyway 마이그레이션과 함께 추가 예정, [000-database-schema](specs/000-database-schema/implementation.md) 참고).

Redis·Kafka는 아직 compose에 없습니다. 필요해지는 시점(캐시/분산 락, 이벤트 발행 기능 구현)에 `docker-compose.yml`에 추가합니다.

## 4. API 문서 확인 (TODO)

springdoc-openapi(Swagger) 의존성은 아직 각 서비스 `build.gradle.kts`에 추가되지 않았습니다. 추가 후 아래 형식으로 확인할 수 있습니다.

```text
http://localhost:{포트}/swagger-ui/index.html
```

## 5. 자주 겪는 문제

| 증상 | 원인 · 해결 |
|------|------------|
| `compileKotlin`에서 `Internal compiler error` | 로컬 JDK가 Kotlin 컴파일러 미지원 버전(예: 매우 최신 JDK)인 경우 발생. `gradle.properties`의 `org.gradle.java.home`으로 지원되는 JDK(예: Temurin 21)를 Gradle 데몬에 지정하고, 기존 Kotlin daemon 프로세스를 종료 후 재시도 (`pkill -f KotlinCompileDaemon`) |
| 포트 충돌 | 위 [서비스 포트](#1-서비스-포트) 표를 참고해 다른 프로세스가 같은 포트를 쓰고 있지 않은지 확인 |

## 관련 문서

- [docs/conventions/backend.md](conventions/backend.md) — 빌드/언어 스택
- [docs/conventions/infra-deploy.md](conventions/infra-deploy.md) — 운영 배포
