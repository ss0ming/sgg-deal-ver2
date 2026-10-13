# DB 설계 — 초기 스키마 (서비스별)

> DBMS: MySQL 8.0 · 컨테이너 1개 + 서비스별 스키마 분리 (`ssgdeal_{service}`)
> 공통 네이밍·PK·타임스탬프·Soft Delete 규약은 [database.md](../../conventions/database.md)를 따릅니다.
> **FK 제약은 어디에도 두지 않습니다** — 서비스 간은 물론 같은 서비스 내 연관관계도 `BIGINT` 컬럼 + 인덱스로만 표현하고, 정합성은 애플리케이션에서 보장합니다([database.md](../../conventions/database.md#1-pk-전략) 참고).

## 스키마 ↔ 서비스 매핑

| 스키마 | 서비스 | Soft Delete 정책 |
|--------|--------|-------------------|
| `ssgdeal_auth` | auth_service | 회원 정책(soft delete + 마스킹) |
| `ssgdeal_user` | user_service | 회원 정책(soft delete + 마스킹) |
| `ssgdeal_cart` | cart_service | 임시 데이터(hard delete) |
| `ssgdeal_promotion` | promotion_service | 상품/특가(soft delete) |
| `ssgdeal_order` | order_service | 주문(삭제 없음, 상태로 관리) |
| `ssgdeal_payment` | payment_service | 결제(삭제 없음, 상태로 관리) |
| `ssgdeal_notification` | notification_service | 로그성 데이터(삭제 없음, append-only) |
| _(없음)_ | gateway_service | DB 없음 — 라우팅만 수행 |

---

## 1. auth_service — `ssgdeal_auth`

### 1-1. credentials — 로그인 자격 증명

- 이메일/비밀번호와 권한(role)만 가진다. 토큰(Access/Refresh)은 DB에 저장하지 않는다 — Refresh Token은 Redis `refresh:{userId}`에 저장([auth-security.md](../../conventions/auth-security.md)).
- 탈퇴 시 soft delete + 이메일 마스킹(`deleted_{id}@deleted.local`)으로 unique 제약 충돌 없이 재가입을 허용한다.

```sql
CREATE TABLE credentials (
    id            BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
    email         VARCHAR(100) NOT NULL,
    password_hash VARCHAR(100) NOT NULL,
    role          VARCHAR(20)  NOT NULL DEFAULT 'USER'
                      CHECK (role IN ('USER', 'ADMIN')),
    created_at    DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at    DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
    deleted_at    DATETIME(6),

    CONSTRAINT uq_credentials_email UNIQUE (email)
);
```

> `id`는 가입 처리 시 user_service의 `users.id`와 동일한 값으로 맞춘다(동기 호출 또는 `UserRegistered` 이벤트로 전파). 서비스 간 FK는 두지 않으므로 애플리케이션에서 이 불변식을 지킨다.

---

## 2. user_service — `ssgdeal_user`

### 2-1. users — 회원 프로필

```sql
CREATE TABLE users (
    id          BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
    nickname    VARCHAR(30)  NOT NULL,
    phone       VARCHAR(20),
    birth_date  DATE,
    created_at  DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at  DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
    deleted_at  DATETIME(6)
);
```

### 2-2. addresses — 배송지

```sql
CREATE TABLE addresses (
    id             BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id        BIGINT       NOT NULL,
    recipient_name VARCHAR(30)  NOT NULL,
    phone          VARCHAR(20)  NOT NULL,
    zip_code       VARCHAR(10)  NOT NULL,
    address1       VARCHAR(200) NOT NULL,
    address2       VARCHAR(200),
    is_default     BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at     DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at     DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
    deleted_at     DATETIME(6)
);

CREATE INDEX idx_addresses_user_id ON addresses (user_id);
```

---

## 3. cart_service — `ssgdeal_cart` (hard delete — `deleted_at` 없음)

```sql
CREATE TABLE carts (
    id         BIGINT      NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT      NOT NULL,
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    CONSTRAINT uq_carts_user_id UNIQUE (user_id)
);

CREATE TABLE cart_items (
    id         BIGINT      NOT NULL AUTO_INCREMENT PRIMARY KEY,
    cart_id    BIGINT      NOT NULL,
    deal_id    BIGINT      NOT NULL,
    option_id  BIGINT,
    quantity   INT         NOT NULL DEFAULT 1,
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    CONSTRAINT uq_cart_items_cart_deal_option UNIQUE (cart_id, deal_id, option_id)
);

CREATE INDEX idx_cart_items_cart_id ON cart_items (cart_id);
```

---

## 4. promotion_service — `ssgdeal_promotion` (soft delete)

```sql
CREATE TABLE deals (
    id             BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
    title          VARCHAR(200) NOT NULL,
    description    TEXT,
    original_price BIGINT       NOT NULL,
    deal_price     BIGINT       NOT NULL,
    stock_quantity INT          NOT NULL,
    status         VARCHAR(20)  NOT NULL DEFAULT 'SCHEDULED'
                       CHECK (status IN ('SCHEDULED', 'OPEN', 'CLOSED', 'SOLD_OUT')),
    open_at        DATETIME(6)  NOT NULL,
    close_at       DATETIME(6)  NOT NULL,
    created_at     DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at     DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
    deleted_at     DATETIME(6)
);

CREATE INDEX idx_deals_status_open_at ON deals (status, open_at);

CREATE TABLE deal_options (
    id               BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
    deal_id          BIGINT       NOT NULL,
    option_name      VARCHAR(100) NOT NULL,
    additional_price BIGINT       NOT NULL DEFAULT 0,
    stock_quantity   INT          NOT NULL,
    created_at       DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at       DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
    deleted_at       DATETIME(6)
);

CREATE INDEX idx_deal_options_deal_id ON deal_options (deal_id);
```

> `open_at`/`close_at`은 UTC로 저장한다. 운영자는 KST로 입력하고, 변환은 `application` 계층의 `PromotionTimePolicy` 단일 지점에서만 수행한다([database.md](../../conventions/database.md#3-시간-저장-기준)).

---

## 5. order_service — `ssgdeal_order` (삭제 없음 — 상태로 관리)

```sql
CREATE TABLE orders (
    id           BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
    order_no     VARCHAR(20)  NOT NULL,
    user_id      BIGINT       NOT NULL,
    status       VARCHAR(20)  NOT NULL DEFAULT 'CREATED'
                     CHECK (status IN ('CREATED', 'PAID', 'CANCELED', 'REFUNDED', 'COMPLETED')),
    total_amount BIGINT       NOT NULL,
    created_at   DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at   DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    CONSTRAINT uq_orders_order_no UNIQUE (order_no)
);

CREATE INDEX idx_orders_user_id ON orders (user_id);

CREATE TABLE order_items (
    id         BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
    order_id   BIGINT       NOT NULL,
    deal_id    BIGINT       NOT NULL,
    option_id  BIGINT,
    item_name  VARCHAR(200) NOT NULL,
    unit_price BIGINT       NOT NULL,
    quantity   INT          NOT NULL,
    created_at DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6)
);

CREATE INDEX idx_order_items_order_id ON order_items (order_id);
```

> `item_name`·`unit_price`는 주문 시점의 `promotion_service` 딜 정보를 **스냅샷**으로 복제해둔 값이다. 이후 딜이 수정/삭제돼도 과거 주문 내역이 깨지지 않도록 하기 위함이며, 매 조회마다 다른 서비스를 호출하지 않아도 되는 이점도 있다. `order_no` 포맷은 `yyyyMMdd` + 랜덤 영숫자 8자리([database.md](../../conventions/database.md#1-pk-전략)).

---

## 6. payment_service — `ssgdeal_payment` (삭제 없음 — 상태로 관리)

```sql
CREATE TABLE payments (
    id               BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
    order_no         VARCHAR(20)  NOT NULL,
    toss_payment_key VARCHAR(200),
    amount           BIGINT       NOT NULL,
    status           VARCHAR(20)  NOT NULL DEFAULT 'READY'
                         CHECK (status IN ('READY', 'PAID', 'CANCELED', 'FAILED', 'REFUNDED')),
    approved_at      DATETIME(6),
    created_at       DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at       DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),

    CONSTRAINT uq_payments_order_no UNIQUE (order_no)
);
```

> `order_no`로 order_service와 연결한다(FK 없음). Toss Payments 쪽 `orderId`에도 이 값을 그대로 사용한다([auth-security.md](../../conventions/auth-security.md) 관련 언급).

---

## 7. notification_service — `ssgdeal_notification` (append-only — `deleted_at` 없음)

```sql
CREATE TABLE notifications (
    id         BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT,
    type       VARCHAR(30)  NOT NULL,
    channel    VARCHAR(20)  NOT NULL DEFAULT 'SLACK'
                   CHECK (channel IN ('SLACK', 'EMAIL', 'PUSH')),
    message    TEXT         NOT NULL,
    status     VARCHAR(20)  NOT NULL DEFAULT 'PENDING'
                   CHECK (status IN ('PENDING', 'SENT', 'FAILED')),
    sent_at    DATETIME(6),
    created_at DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6)
);

CREATE INDEX idx_notifications_user_id ON notifications (user_id);
```
