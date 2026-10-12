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

## 3. 전체 스택 실행 (TODO)

MySQL + Redis + Kafka + 전체 서비스를 한 번에 띄우는 `docker-compose.yml`은 아직 작성되지 않았습니다. 기능 구현과 함께 추가하고, 여기에 실행 명령을 채워 넣으세요.

```bash
# TODO: docker compose up -d
```

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
