# [컨벤션] 데이터베이스 & 모델링

> **범위**: DBMS·DB 소유 전략·네이밍·PK·타임스탬프·Soft Delete·시간 기준·캐시
> **변경 시**: 이 문서를 단일 출처로 갱신합니다 ([AGENT_GUIDE 문서 동기화](../AGENT_GUIDE.md#문서-동기화-필수) 참고).

## 결정 사항

| 항목 | 결정 | 비고 |
|------|------|------|
| DBMS | MySQL | |
| 캐시 / 분산 락 | Redis | Lua Script 기반 분산 락([backend.md](./backend.md) 참고) |
| DB 소유 전략 | 서비스별 DB 분리 (Database per Service) | MSA 원칙상 서비스 간 직접 DB 접근 금지, API/이벤트로만 통신 |
| 마이그레이션 도구 | Flyway | SOT: [backend.md](./backend.md#1-db-마이그레이션-도구--flyway) |
| 테이블 / 컬럼 네이밍 | snake_case | |
| PK 전략 | `BIGINT AUTO_INCREMENT` (내부) + 외부 노출용 별도 식별자 컬럼 | 서비스 간 참조는 FK 제약 없는 `BIGINT` 컬럼 |
| 공통 타임스탬프(감사 컬럼) | created_at / updated_at | `BaseEntity` + JPA Auditing으로 채움 |
| Soft Delete 정책 | 테이블별 정책 (아래 표 참고) | `deleted_at` 컬럼 사용, `is_deleted` boolean 미사용 |
| 시간 저장 기준 | UTC, `DATETIME(6)` | 표시 시점에만 KST로 변환 |
| 시드 / 더미 데이터 | TODO | |

## 규칙 / 적용

### 1. PK 전략

- 내부 PK: `BIGINT AUTO_INCREMENT` (JPA `GenerationType.IDENTITY`). 엔티티 id는 `val id: Long? = null`.
- 외부에 노출하는 식별자(API 응답·URL)는 내부 PK를 그대로 쓰지 않고 별도 컬럼으로 둔다.
  - 주문: `order_no`(`VARCHAR(20)`, UNIQUE). 형식 `yyyyMMdd` + 랜덤 영숫자 8자리.
  - 결제: Toss Payments의 `orderId`로 `order_no`를 사용.
- 서비스 간 참조(다른 서비스의 엔티티 id)는 FK 제약 없이 `BIGINT` 컬럼으로만 둔다([DB 소유 전략](#결정-사항) 참고).
- 같은 서비스 내 연관관계도 FK 제약은 생성하지 않는다. 정합성은 애플리케이션에서 보장한다.

**이유**: 랜덤 UUID(v4)는 InnoDB 클러스터드 인덱스에서 페이지 분할이 잦아 삽입 성능이 떨어진다. 순차 PK를 외부에 노출하면 주문량 추정과 타 주문 조회 시도(IDOR)에 취약하다. `IDENTITY` 전략은 Hibernate 배치 insert를 비활성화하지만, 목표 TPS(200, [infra-deploy.md](./infra-deploy.md)의 목표 성능 지표)에서는 병목이 아니므로 단순성을 택한다. 대안으로 TSID(시간순 정렬 + 애플리케이션 생성)는 배치 insert가 가능하지만, 현재 규모에서 도입 이득이 작아 보류한다.

### 2. Soft Delete 정책

테이블 성격에 따라 다르게 적용한다.

| 대상 | 정책 |
|------|------|
| 주문, 결제 | 삭제하지 않는다. 상태(`CANCELED`, `REFUNDED`)로 관리 |
| 상품, 특가 딜 | Soft delete |
| 회원 | 탈퇴 시 soft delete + 개인정보 마스킹 |
| 장바구니, 토큰 등 임시 데이터 | Hard delete |

- Soft delete 컬럼: `deleted_at DATETIME(6) NULL`. `is_deleted` boolean은 사용하지 않는다.
- 조회 필터는 Hibernate `@SQLRestriction("deleted_at IS NULL")`로 적용한다.
- 회원 탈퇴 시 이메일·이름·전화번호를 마스킹한다. 이메일은 `deleted_{id}@deleted.local`로 치환해 unique 제약 충돌을 막는다.
- 탈퇴 회원 데이터는 법정 보관 기간 이후 파기한다(보관 기간: TODO).

**이유**: 주문·결제는 정산과 CS 대응을 위해 이력이 필요하고, 삭제 대신 상태 전이가 도메인에 맞다. 상품은 과거 주문이 참조하므로 hard delete하면 주문 내역 조회가 깨진다. `deleted_at`은 삭제 여부와 삭제 시각을 한 컬럼으로 표현한다. 이메일 unique 제약 + soft delete를 그대로 두면 탈퇴한 이메일로 재가입할 수 없어 마스킹이 필요하다.

### 3. 시간 저장 기준

DB와 애플리케이션 모두 **UTC**로 저장·처리하고, 표시 시점에만 KST로 변환한다.

- 컬럼 타입: `DATETIME(6)`. `TIMESTAMP` 사용 금지.
- JVM 타임존: UTC (`-Duser.timezone=UTC`, 컨테이너 `TZ=UTC`).
- JDBC URL: `connectionTimeZone=UTC&forceConnectionTimeZoneToSession=true`.
- Kotlin 타입: 시점은 `Instant`, 사용하지 않을 수 없는 경우만 `LocalDateTime`(UTC 기준).
- API 응답: 오프셋을 포함한 ISO-8601(예: `2026-10-16T09:00:00Z`) — [api-design.md](./api-design.md)의 날짜 포맷 규약과 함께 확인.
- 모든 테이블에 `created_at`, `updated_at`을 두고 `BaseEntity` + JPA Auditing으로 채운다.

**비즈니스 시간(KST) 처리**

- 프로모션 오픈/마감 시각은 운영자가 KST로 입력한다.
- KST → UTC 변환은 `application` 계층의 단일 지점(`PromotionTimePolicy`)에서만 수행한다([backend.md](./backend.md)의 레이어드 구조 참고).
- 테스트에서 시간은 `Clock`을 주입받아 고정한다. `Instant.now()` 직접 호출 금지.

**이유**: `TIMESTAMP`는 세션 타임존에 따라 저장·조회 값이 변하고 2038년 한계가 있다. 서버·DB·컨테이너 타임존이 섞이면 "금요일 18:00 오픈" 특가가 9시간 어긋나 열릴 수 있다. 변환 지점을 한 곳으로 모아야 버그 위치를 추적할 수 있다.

## 관련 문서

- [백엔드 스택](./backend.md) — Flyway, 레이어드 구조
- [API 설계 & 응답 형식](./api-design.md) — 날짜 포맷
- [인프라 & 배포](./infra-deploy.md) — 목표 TPS
