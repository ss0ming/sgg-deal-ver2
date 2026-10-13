# [계획] 초기 데이터베이스 스키마

| 항목 | 내용 |
|------|------|
| 기능명 | 초기 데이터베이스 스키마 |
| Spec 폴더 | `docs/specs/000-database-schema/` |
| 영역 | auth_service, user_service, cart_service, promotion_service, order_service, payment_service, notification_service (공통: database.md 규약) |
| 작성자 | 본인 |
| 작성일 | 2026-10-13 |
| 상태 | 계획 |

## 배경 / 목적

서비스 코드를 짜기 전에, 서비스별로 분리된 DB 스키마와 테이블을 먼저 설계해야 한다. MSA 원칙(Database per Service)에 따라 서비스마다 자기 스키마만 소유하고, 로컬에서도 그 분리를 그대로 재현할 수 있어야 한다.

## 목표 (Goals)

- 서비스 8개 중 DB가 필요한 7개(게이트웨이 제외)의 초기 테이블 DDL을 확정한다.
- 로컬에서 MySQL 컨테이너 하나로 7개 스키마를 한 번에 띄우는 방법을 마련한다([database.md](database.md) 참고).

## 비목표 (Non-Goals)

- 실제 Flyway 마이그레이션 파일 작성 및 각 서비스 `build.gradle.kts`에 JPA/MySQL 의존성 추가(이번 스펙은 설계만, 구현은 `implementation.md`의 남은 항목으로 분리).
- 상품 옵션·쿠폰 등 세부 비즈니스 규칙의 완성형 설계(최소 뼈대만 우선 확정).

## 요구사항

- [database.md](../../conventions/database.md) 규약(PK 전략, Soft Delete 정책, UTC 저장, FK 미사용)을 그대로 따른다.
- 서비스 간 참조는 FK 없이 `BIGINT` 컬럼만 사용한다.

## 설계 개요 / 접근 방식

서비스별로 스키마를 나누고(`ssgdeal_{service}`), 핵심 엔티티를 1차로 설계했다. 상세 DDL은 [database.md](database.md) 참고.

- **auth_service**: 로그인 자격 증명만 소유(`credentials`). 토큰은 Redis.
- **user_service**: 회원 프로필·배송지(`users`, `addresses`).
- **cart_service**: 장바구니(`carts`, `cart_items`) — 임시 데이터라 hard delete.
- **promotion_service**: 특가 딜·옵션(`deals`, `deal_options`) — 상품 마스터를 따로 두지 않고 딜 자체가 판매 단위.
- **order_service**: 주문·주문 상품(`orders`, `order_items`) — 딜 정보를 스냅샷으로 복제.
- **payment_service**: 결제(`payments`) — `order_no`로 주문과 연결.
- **notification_service**: 알림 로그(`notifications`) — append-only.

## 의사결정 (함께 논의 · 근거 필수)

| 결정할 항목 | 선택지 | 제안 / 근거 | 상태 |
|------|--------|------------|------|
| 인증(auth)과 회원(user) 분리 여부 | A) auth_service가 로그인 정보까지 다 가짐 B) auth_service(자격증명)·user_service(프로필) 분리 | B로 결정. **auth_service**는 인증(로그인/회원가입/토큰 발급)만 책임지고, **user_service**가 유저 프로필의 조회·수정 책임을 전담한다 | 합의됨 |
| auth↔user 간 가입 시 id 동기화 방식 | A) 동기 API 호출(Feign) B) 이벤트(Kafka) | A로 결정. 회원가입 시 auth_service가 credentials를 만든 뒤 **내부 API(Feign)로 user_service를 동기 호출**해 같은 id로 프로필을 생성한다. REST+OpenFeign을 서비스 간 동기 통신으로 이미 채택했고([api-design.md](../../conventions/api-design.md)), 가입은 두 레코드가 함께 성공해야 의미가 있어 이벤트의 최종 일관성보다 즉시 일관성이 더 맞는다 | 합의됨 |
| 주문 항목에 딜 정보를 스냅샷으로 복제할지, 매번 promotion_service를 호출할지 | A) 스냅샷 복제 B) 조회 시 Feign 호출 | A 제안. 과거 주문은 당시 가격·이름이 보존돼야 하고(딜이 삭제/변경돼도 주문 내역이 깨지지 않음), 조회할 때마다 다른 서비스를 부르지 않아도 됨 | 보류 — `order_service` 구현 시점에 결정 |
| 상품 마스터를 따로 둘지 | A) 딜(promotion) = 판매 단위 그 자체 B) 별도 product 마스터 + 딜이 참조 | A 제안. "이벤트성 특가 프로모션만 판매"하는 비즈니스 모델상 상시 상품 카탈로그가 없어 보여 단순화. 상시 판매로 확장되면 B로 분리 | 보류 — `promotion_service` 구현 시점에 결정 |
| ENUM 표현 방식 | A) MySQL `ENUM` 타입 B) `VARCHAR` + `CHECK` | B 제안. 값 추가/변경 시 `ALTER` 부담이 적고, JPA에서 문자열로 다루기 더 단순함 | 제안 — 확인 필요 |

## 영향 범위

- 각 `{service}_service/src/main/resources/db/migration/`에 들어갈 Flyway 마이그레이션 파일(이번 스펙 이후 구현 단계).
- 각 서비스 `build.gradle.kts`에 `mysql-connector-j`, `flyway-mysql` 의존성 추가 필요.
- `docker-compose.yml`(로컬 MySQL 7-스키마 구성).

## 작업 단계

- [x] 서비스별 테이블 DDL 1차 설계
- [x] 로컬 MySQL 컨테이너 구성 방법 결정(컨테이너 1개 + 스키마 분리)
- [ ] 위 "의사결정" 표 확인/확정
- [ ] 서비스별 Flyway 마이그레이션 파일 작성 (`V1__init_schema.sql`)
- [ ] 각 서비스 `build.gradle.kts`에 DB 의존성 추가

## 리스크 / 미해결 질문

- auth/user 분리 시 가입 트랜잭션을 어떻게 처리할지(둘 다 성공해야 하는 분산 트랜잭션 문제)는 이후 `005-auth-member` 같은 별도 스펙에서 다룬다.
- 쿠폰/적립금 등 프로모션 확장 기능은 이번 스펙 범위 밖이다.
