# 참고 — 이전 Java 프로젝트의 실제 DDL

> **주의**: 이 문서는 새 Kotlin 프로젝트의 설계가 아니라, 이전 Java 구현의 실제 스키마를 그대로 옮겨둔 **참고 자료**입니다.
> 현재 프로젝트의 확정 설계는 [database.md](database.md)이며, 아래 내용과 다른 부분(FK 제약, 감사 컬럼 구성, 테이블 분리 방식 등)은 의도적인 차이입니다 — [conventions/database.md](../../conventions/database.md)를 따릅니다.

## 1. `auth_service` — 인증

```sql
CREATE TABLE auth (
    id          BIGINT       NOT NULL AUTO_INCREMENT,
    user_id     BIGINT,
    username    VARCHAR(10)  NOT NULL,
    password    VARCHAR(255),
    role        VARCHAR(255) NOT NULL,   -- MASTER, COMPANY_MANAGER, CONSUMER
    created_at  DATETIME     NOT NULL,
    created_by  BIGINT,
    updated_at  DATETIME     NOT NULL,
    updated_by  BIGINT,
    deleted_at  DATETIME,
    deleted_by  BIGINT,
    is_deleted  BIT          DEFAULT 0,
    PRIMARY KEY (id)
);
```

`Passport`는 `@Entity`가 아닌 순수 DTO/VO이며 테이블 없음.

## 2. `user_service` — 유저

```sql
CREATE TABLE `user` (
    id          BIGINT       NOT NULL AUTO_INCREMENT,
    nickname    VARCHAR(255),
    slack_email VARCHAR(255) NOT NULL,
    created_at  DATETIME     NOT NULL,
    created_by  BIGINT,
    updated_at  DATETIME     NOT NULL,
    updated_by  BIGINT,
    deleted_at  DATETIME,
    deleted_by  BIGINT,
    is_deleted  BIT          DEFAULT 0,
    PRIMARY KEY (id)
);

CREATE TABLE destination (
    id          BIGINT       NOT NULL AUTO_INCREMENT,
    user_id     BIGINT,
    address     VARCHAR(255) NOT NULL,
    name        VARCHAR(255),
    created_at  DATETIME     NOT NULL,
    created_by  BIGINT,
    updated_at  DATETIME     NOT NULL,
    updated_by  BIGINT,
    deleted_at  DATETIME,
    deleted_by  BIGINT,
    is_deleted  BIT          DEFAULT 0,
    PRIMARY KEY (id)
);
```

## 3. `promotion_service` — 프로모션/상품

```sql
CREATE TABLE company (
    id          BIGINT       NOT NULL AUTO_INCREMENT,
    manager_id  BIGINT       NOT NULL,
    name        VARCHAR(100) NOT NULL,
    logo_url    VARCHAR(2048) NOT NULL,
    promotion_id BIGINT,
    created_at  DATETIME     NOT NULL,
    created_by  BIGINT,
    updated_at  DATETIME     NOT NULL,
    updated_by  BIGINT,
    deleted_at  DATETIME,
    deleted_by  BIGINT,
    is_deleted  BIT          DEFAULT 0,
    PRIMARY KEY (id),
    CONSTRAINT fk_company_promotion FOREIGN KEY (promotion_id) REFERENCES promotion (id)
);

CREATE TABLE promotion (
    id                   BIGINT        NOT NULL AUTO_INCREMENT,
    title                VARCHAR(255)  NOT NULL,
    preview_url          VARCHAR(2048) NOT NULL,
    content_image_url    VARCHAR(2048) NOT NULL,
    content              TEXT          NOT NULL,
    status               VARCHAR(255)  NOT NULL,  -- PENDING, IN_PROGRESS, FINISHED
    start_promotion_date DATE          NOT NULL,
    end_promotion_date   DATE          NOT NULL,
    created_at           DATETIME      NOT NULL,
    created_by           BIGINT,
    updated_at           DATETIME      NOT NULL,
    updated_by           BIGINT,
    deleted_at           DATETIME,
    deleted_by           BIGINT,
    is_deleted           BIT           DEFAULT 0,
    PRIMARY KEY (id)
);

CREATE TABLE product (
    id                BIGINT        NOT NULL AUTO_INCREMENT,
    company_id        BIGINT        NOT NULL,
    name              VARCHAR(100)  NOT NULL,
    original_price    BIGINT        NOT NULL,
    promotion_price   BIGINT        NOT NULL,
    preview_url       VARCHAR(2048) NOT NULL,
    content_image_url VARCHAR(2048) NOT NULL,
    content           VARCHAR(255)  NOT NULL,
    version           BIGINT        NOT NULL,
    created_at        DATETIME      NOT NULL,
    created_by        BIGINT,
    updated_at        DATETIME      NOT NULL,
    updated_by        BIGINT,
    deleted_at        DATETIME,
    deleted_by        BIGINT,
    is_deleted        BIT           DEFAULT 0,
    PRIMARY KEY (id),
    CONSTRAINT fk_product_company FOREIGN KEY (company_id) REFERENCES company (id)
);

CREATE TABLE product_option (
    id            BIGINT       NOT NULL AUTO_INCREMENT,
    product_id    BIGINT,
    option_name   VARCHAR(100) NOT NULL,
    extra_price   BIGINT       NOT NULL,
    product_stock BIGINT       NOT NULL,
    created_at    DATETIME     NOT NULL,
    created_by    BIGINT,
    updated_at    DATETIME     NOT NULL,
    updated_by    BIGINT,
    deleted_at    DATETIME,
    deleted_by    BIGINT,
    is_deleted    BIT          DEFAULT 0,
    PRIMARY KEY (id),
    CONSTRAINT fk_product_option_product FOREIGN KEY (product_id) REFERENCES product (id)
);

CREATE TABLE product_ranking (
    id          BIGINT       NOT NULL AUTO_INCREMENT,
    name        VARCHAR(100) NOT NULL,  -- ProductRankingTitle
    created_at  DATETIME     NOT NULL,
    created_by  BIGINT,
    updated_at  DATETIME     NOT NULL,
    updated_by  BIGINT,
    deleted_at  DATETIME,
    deleted_by  BIGINT,
    is_deleted  BIT          DEFAULT 0,
    PRIMARY KEY (id)
);

CREATE TABLE product_ranking_info (
    id                  BIGINT   NOT NULL AUTO_INCREMENT,
    product_id          BIGINT   NOT NULL,
    product_ranking_id  BIGINT   NOT NULL,
    score               BIGINT   NOT NULL,
    created_at          DATETIME NOT NULL,
    created_by          BIGINT,
    updated_at          DATETIME NOT NULL,
    updated_by          BIGINT,
    deleted_at          DATETIME,
    deleted_by          BIGINT,
    is_deleted          BIT      DEFAULT 0,
    PRIMARY KEY (id),
    CONSTRAINT fk_pri_product FOREIGN KEY (product_id) REFERENCES product (id),
    CONSTRAINT fk_pri_product_ranking FOREIGN KEY (product_ranking_id) REFERENCES product_ranking (id)
);
```

## 4. `order_service` — 주문

```sql
CREATE TABLE total_order (
    id                 BIGINT       NOT NULL AUTO_INCREMENT,
    total_order_number VARCHAR(10)  NOT NULL UNIQUE,
    price              BIGINT       NOT NULL,   -- TotalPrice.value (column name: price)
    status             VARCHAR(255) NOT NULL,   -- PENDING, PAID, FAILED, CANCELED, EXPIRED
    created_at         DATETIME     NOT NULL,
    created_by         BIGINT,
    updated_at         DATETIME     NOT NULL,
    updated_by         BIGINT,
    deleted_at         DATETIME,
    deleted_by         BIGINT,
    is_deleted         BIT          DEFAULT 0,
    PRIMARY KEY (id)
);

CREATE TABLE `orders` (
    id            BIGINT       NOT NULL AUTO_INCREMENT,
    total_order_id BIGINT,
    price         BIGINT,
    status        VARCHAR(255),                -- OrderStatus
    company_id    BIGINT,
    company_name  VARCHAR(255),
    created_at    DATETIME     NOT NULL,
    created_by    BIGINT,
    updated_at    DATETIME     NOT NULL,
    updated_by    BIGINT,
    deleted_at    DATETIME,
    deleted_by    BIGINT,
    is_deleted    BIT          DEFAULT 0,
    PRIMARY KEY (id),
    CONSTRAINT fk_orders_total_order FOREIGN KEY (total_order_id) REFERENCES total_order (id)
);

CREATE TABLE order_product (
    id               BIGINT       NOT NULL AUTO_INCREMENT,
    orders_id        BIGINT,
    product_id       BIGINT,
    product_name     VARCHAR(255) NOT NULL,
    quantity         BIGINT       NOT NULL,
    preview_url      VARCHAR(255),
    original_price   BIGINT       NOT NULL,
    promotion_price  BIGINT       NOT NULL,
    option_id        BIGINT       NOT NULL,
    option_name      VARCHAR(255) NOT NULL,
    extra_price      BIGINT       NOT NULL,
    total_price      BIGINT       NOT NULL,
    created_at       DATETIME     NOT NULL,
    created_by       BIGINT,
    updated_at       DATETIME     NOT NULL,
    updated_by       BIGINT,
    deleted_at       DATETIME,
    deleted_by       BIGINT,
    is_deleted       BIT          DEFAULT 0,
    PRIMARY KEY (id),
    CONSTRAINT fk_order_product_orders FOREIGN KEY (orders_id) REFERENCES `orders` (id)
);

CREATE TABLE orderer (
    id              BIGINT       NOT NULL AUTO_INCREMENT,
    total_order_id  BIGINT,
    user_id         BIGINT,
    nickname        VARCHAR(255),
    slack_email     VARCHAR(255),
    destination     VARCHAR(255),
    delivery_request VARCHAR(20),
    created_at      DATETIME     NOT NULL,
    created_by      BIGINT,
    updated_at      DATETIME     NOT NULL,
    updated_by      BIGINT,
    deleted_at      DATETIME,
    deleted_by      BIGINT,
    is_deleted      BIT          DEFAULT 0,
    PRIMARY KEY (id),
    CONSTRAINT fk_orderer_total_order FOREIGN KEY (total_order_id) REFERENCES total_order (id)
);

CREATE TABLE total_order_payment (
    id                 BIGINT       NOT NULL AUTO_INCREMENT,
    total_order_id     BIGINT,
    payment_id         BIGINT,
    payment_type       VARCHAR(255),  -- TOSS
    payment_method     VARCHAR(255),  -- CARD
    payment_amount     BIGINT,
    payment_key        VARCHAR(255),
    payment_date       DATETIME,
    payment_status     VARCHAR(255),  -- PENDING, CANCELED, PARTIAL_CANCELED, COMPLETED
    payment_fail_reason VARCHAR(255),
    created_at         DATETIME     NOT NULL,
    created_by         BIGINT,
    updated_at         DATETIME     NOT NULL,
    updated_by         BIGINT,
    deleted_at         DATETIME,
    deleted_by         BIGINT,
    is_deleted         BIT          DEFAULT 0,
    PRIMARY KEY (id),
    CONSTRAINT fk_top_total_order FOREIGN KEY (total_order_id) REFERENCES total_order (id)
);
```

## 5. `payment_service` — 결제

```sql
CREATE TABLE payment (
    id                  BIGINT       NOT NULL AUTO_INCREMENT,
    total_order_id      BIGINT       NOT NULL,
    user_id             BIGINT       NOT NULL,
    payment_key         VARCHAR(255) NOT NULL,
    payment_amount      BIGINT       NOT NULL,
    payment_method      VARCHAR(255) NOT NULL,  -- CARD
    payment_type        VARCHAR(255) NOT NULL,  -- TOSS, NAVER_PAY, KAKAO_PAY, CREDIT_CARD, ACCOUNT_TRANSFER, VIRTUAL_ACCOUNT
    payment_status      VARCHAR(255) NOT NULL,  -- PENDING, CANCELED, PARTIAL_CANCELED, COMPLETED, FAILED
    payment_fail_reason VARCHAR(255),
    balance_amount      BIGINT       NOT NULL,
    created_at          DATETIME     NOT NULL,
    created_by          BIGINT,
    updated_at          DATETIME     NOT NULL,
    updated_by          BIGINT,
    deleted_at          DATETIME,
    deleted_by          BIGINT,
    is_deleted          BIT          DEFAULT 0,
    PRIMARY KEY (id)
);
```

## 6. `notification_service` — 알림

```sql
CREATE TABLE notification_template (
    id          BIGINT       NOT NULL AUTO_INCREMENT,
    content     TEXT         NOT NULL,
    type        VARCHAR(255) NOT NULL,  -- FIND_ID, FIND_PASSWORD, ORDER_COMPLETED
    created_at  DATETIME     NOT NULL,
    created_by  BIGINT,
    updated_at  DATETIME     NOT NULL,
    updated_by  BIGINT,
    deleted_at  DATETIME,
    deleted_by  BIGINT,
    is_deleted  BIT          DEFAULT 0,
    PRIMARY KEY (id)
);

CREATE TABLE notification (
    id                     BIGINT       NOT NULL AUTO_INCREMENT,
    notification_template_id BIGINT     NOT NULL,
    sender_slack_email     VARCHAR(255) NOT NULL,
    receiver_slack_email   VARCHAR(255) NOT NULL,
    content                TEXT         NOT NULL,
    template_type          VARCHAR(255) NOT NULL,  -- FIND_ID, FIND_PASSWORD, ORDER_COMPLETED
    status                 VARCHAR(255) NOT NULL,  -- PENDING, SUCCESS, FAILED
    send_at                DATETIME,
    created_at             DATETIME     NOT NULL,
    created_by             BIGINT,
    updated_at             DATETIME     NOT NULL,
    updated_by             BIGINT,
    deleted_at             DATETIME,
    deleted_by             BIGINT,
    is_deleted             BIT          DEFAULT 0,
    PRIMARY KEY (id),
    CONSTRAINT fk_notification_template FOREIGN KEY (notification_template_id) REFERENCES notification_template (id)
);

CREATE TABLE notification_record (
    id                   BIGINT       NOT NULL AUTO_INCREMENT,
    notification_id      BIGINT       NOT NULL,
    receiver_slack_email VARCHAR(255) NOT NULL,
    send_at              DATETIME     NOT NULL,
    template_type        VARCHAR(255) NOT NULL,
    status               VARCHAR(255) NOT NULL,
    created_at           DATETIME     NOT NULL,
    created_by           BIGINT,
    updated_at           DATETIME     NOT NULL,
    updated_by           BIGINT,
    deleted_at           DATETIME,
    deleted_by           BIGINT,
    is_deleted           BIT          DEFAULT 0,
    PRIMARY KEY (id),
    CONSTRAINT fk_record_notification FOREIGN KEY (notification_id) REFERENCES notification (id)
);
```

## 현재 설계와의 주요 차이

| 항목 | 이전 Java 프로젝트 | 현재 Kotlin 프로젝트 ([database.md](database.md)) |
|------|--------------------|------------------------------------------------|
| FK 제약 | 사용 (`REFERENCES`) | 미사용 — 서비스 간은 물론 서비스 내부도 `BIGINT` + 인덱스만 |
| 감사 컬럼 | `created_by`/`updated_by`/`deleted_by`/`is_deleted`/`deleted_at` | `created_at`/`updated_at`/`deleted_at`만 (soft delete는 테이블 성격별 정책) |
| 주문 테이블 구조 | `total_order` + `orders`(업체별 분할) + `order_product` + `orderer` + `total_order_payment` | 단일 `orders` + `order_items`(스냅샷) |
| 상품/프로모션 구조 | `company` + `promotion` + `product` + `product_option` + `product_ranking(_info)` | `deals` + `deal_options` (판매 단위 = 딜) |
| 알림 구조 | `notification_template` + `notification` + `notification_record` | `notifications` 단일 테이블 |
| 인증 테이블명 | `auth` (단일 테이블, `user_id` 컬럼으로 user 참조) | `credentials` (user_service와 `id` 동기화) |
