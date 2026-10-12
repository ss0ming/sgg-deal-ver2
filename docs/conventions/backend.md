# [컨벤션] 백엔드 스택

> **범위**: 프레임워크·언어·의존성·ORM·마이그레이션·비동기/메시징·설정·코드 구성(디렉토리 구조)·예외 처리
> **변경 시**: 이 문서를 단일 출처로 갱신합니다 ([AGENT_GUIDE 문서 동기화](../AGENT_GUIDE.md#문서-동기화-필수) 참고).

## 결정 사항

| 항목 | 결정 | 비고 |
|------|------|------|
| 언어 | Kotlin | 기존 Java 17 구현을 Kotlin으로 전환 중 |
| 프레임워크 | Spring Boot 3.x | |
| 빌드 도구 | Gradle (Kotlin DSL), 멀티모듈 | 모듈 = `*_service` (서비스 단위) |
| MSA 구성요소 | Spring Cloud Gateway | Eureka(서비스 디스커버리)·Config Server 도입 여부·시점은 TODO |
| JDK | 17 (toolchain) | |
| ORM | Spring Data JPA | `spring.jpa.hibernate.ddl-auto: validate` 고정, 스키마 변경은 Flyway로만 |
| 동적 쿼리 | QueryDSL | Kotlin 환경 codegen 설정은 TODO |
| DB 마이그레이션 | Flyway | `src/main/resources/db/migration`, `V{번호}__{설명}.sql` |
| 비동기 메시징 | Kafka | 이벤트 계약은 `event-contract` 모듈로 분리 |
| 분산 락 | Redis + Lua Script | 재고 차감 등 동시성 제어 구간에 사용 |
| 캐시 | Redis | |
| 코드 구성 단위 | 서비스 = 도메인 단위, 서비스 내부 = 레이어드 (`presentation`/`application`/`domain`/`infrastructure`) | 의존 방향은 Konsist 아키텍처 테스트로 검증 |
| common 모듈 | 사용. 기술 공통 코드만 허용 (도메인 코드·서비스 간 DTO 공유 금지) | |
| 예외 처리 | `BusinessException(ErrorCode)` 기반, 커스텀 예외는 `RuntimeException` 상속 | 에러 코드는 서비스별 `enum` |
| 애플리케이션 코드 루트 | `{service}_service/src/main/kotlin/com/ssgdeal/{service}/` | |

## 규칙 / 적용

### 1. DB 마이그레이션 도구 — Flyway

- `spring.jpa.hibernate.ddl-auto`는 `validate`로 고정한다. `update`, `create`, `create-drop` 사용 금지.
- 마이그레이션 파일 위치: 각 서비스의 `src/main/resources/db/migration`.
- 파일명: `V{번호}__{설명}.sql` (예: `V3__add_deleted_at_to_product.sql`).
- 이미 적용된 마이그레이션 파일은 수정하지 않는다. 변경이 필요하면 새 버전을 추가한다.
- 각 서비스는 자기 스키마의 마이그레이션만 소유한다. 다른 서비스의 테이블을 생성·변경하지 않는다([database.md](./database.md)의 Database per Service 원칙).
- 엔티티를 변경하면 같은 커밋에 마이그레이션 파일을 포함한다.

**이유**: Liquibase 대비 SQL 파일을 그대로 관리할 수 있어 단순하고, Spring Boot 자동 설정을 바로 쓸 수 있다. `ddl-auto: update`는 컬럼 삭제·타입 변경을 반영하지 못하고 변경 이력이 남지 않는다. 서비스별 스키마 소유를 지키지 않으면 MSA인데도 DB 레벨에서 서비스가 결합된다.

### 2. 코드 구성 단위 — 레이어드 (서비스 내부)

서비스는 도메인 단위로 이미 분리되어 있으므로, 서비스 내부는 레이어드 구조를 사용한다.

```text
com.ssgdeal.{service}
├── presentation     # Controller, Request/Response DTO
├── application      # Service(유스케이스), 트랜잭션 경계
├── domain           # Entity, VO, Domain Service, Repository 인터페이스
└── infrastructure   # Repository 구현, 외부 API, Kafka, Redis
```

- 의존 방향: `presentation → application → domain ← infrastructure`.
- `domain`은 `presentation`, `application`, `infrastructure`를 import하지 않는다.
- `domain`에서 외부 연동(Redis, Kafka, Toss Payments)을 직접 호출하지 않는다. 인터페이스를 `domain`에 두고 구현은 `infrastructure`에 둔다.
- `@Transactional`은 `application` 계층에만 붙인다.
- Controller는 Entity를 반환하지 않는다. 항상 Response DTO로 변환한다.
- 의존 방향 규칙은 Konsist 아키텍처 테스트로 검증한다.

**이유**: 서비스 자체가 이미 도메인 단위라, 서비스 내부까지 피처 단위로 다시 쪼개면 구조가 과해진다. 도메인 로직이 인프라에 의존하지 않아야 Redis·Kafka 없이 단위 테스트할 수 있다. 규칙을 문서로만 두면 깨지므로 테스트로 강제한다.

### 3. common 모듈 — 기술 공통 코드만

`common` 모듈을 두되, 기술적 공통 코드만 허용한다. 서비스 간 이벤트 계약은 `event-contract` 모듈로 분리한다.

- **허용**: 공통 에러 응답 포맷·공통 예외 기반 클래스, Passport 파싱 및 컨텍스트 유틸([auth-security.md](./auth-security.md)), `BaseEntity`(createdAt, updatedAt), 공통 설정(Jackson, 타임존).
- **금지**: 특정 도메인의 Entity·VO·비즈니스 로직, 서비스 간 Request/Response DTO 공유, 특정 서비스에서만 쓰는 유틸.
- **`event-contract` 모듈**: Kafka 이벤트 클래스만 둔다(예: `OrderCreatedEvent`, `PaymentCompletedEvent`). 이벤트 필드는 추가만 허용하고, 삭제·타입 변경 시 새 이벤트 버전을 만든다.

**이유**: common이 비대해지면 한 곳만 수정해도 모든 서비스를 재배포해야 해서 MSA의 독립 배포 이점이 사라진다. 이벤트 스키마는 여러 서비스가 반드시 공유해야 하므로, 변경 영향 범위를 명확히 하려고 별도 모듈로 분리한다.

> `common`·`event-contract` 모듈은 현재 `settings.gradle.kts`에 아직 등록되지 않았다. 실제 모듈 추가는 별도 작업으로 진행한다.

### 4. 예외 처리

- 커스텀 예외는 반드시 `RuntimeException`을 상속한다.
- 비즈니스 예외는 `BusinessException(errorCode: ErrorCode)` 하나를 기반으로 한다.
- 에러 코드는 서비스별 `enum`으로 관리한다(예: `OrderErrorCode.OUT_OF_STOCK`).

**이유**: Spring `@Transactional`은 checked 계열 예외에서 기본적으로 롤백하지 않는다. Kotlin은 checked exception 구분이 없어 실수로 적절하지 않은 예외 타입을 상속하기 쉽고, 결제 실패 시 롤백 누락으로 이어질 수 있다. 단일 `BusinessException` 기반으로 통일하면 전역 예외 핸들러(`@RestControllerAdvice`)에서 일관되게 [API 에러 Envelope](./api-design.md)로 변환할 수 있다.

### 📁 디렉터리 구조 및 구성 원칙

- 서비스별 Gradle 모듈: `auth_service`, `cart_service`, `gateway_service`, `notification_service`, `order_service`, `payment_service`, `promotion_service`, `user_service`.
- 각 서비스의 소스 루트는 `{service}_service/src/main/kotlin/com/ssgdeal/{service}/`이며, 진입점은 `{Service}ServiceApplication.kt`.
- 설정은 `{service}_service/src/main/resources/application.yml`에 둔다.

> [!NOTE]
> 실제 전체 디렉토리 트리는 [README의 프로젝트 구조](../../README.md#프로젝트-구조)가 단일 출처(SOT)입니다. 본 문서는 구성 원칙과 책임을 정의합니다. (TODO: 루트 README.md 작성 필요)

## 관련 문서

- [데이터베이스 & 모델링](./database.md)
- [인증 & 보안](./auth-security.md) — Passport 유틸
- [API 설계 & 응답 형식](./api-design.md) — 에러 Envelope
- [외부 API & 데이터 연동](./external-apis.md) — Toss Payments
