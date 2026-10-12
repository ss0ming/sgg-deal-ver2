<!--
이 파일은 기능 스펙용 DB 설계 템플릿입니다. (선택 — 테이블 추가/변경이 있는 기능에서만 추가)
새 기능을 시작할 때 docs/specs/{NNN}-{feature-name}/database.md 로 복사해 작성하세요.
공통 규약(네이밍·PK 전략·공통 컬럼 등)의 단일 출처는 docs/conventions/database.md이며,
이 문서는 그 규약을 따르는 "이 기능의 테이블 DDL"만 적습니다. 실제 스키마의 단일 출처는
JPA 엔티티 + 마이그레이션 스크립트이며, 이 문서는 설계 의도를 설명하는 보조 문서입니다.
{ } 안의 placeholder는 실제 값으로 바꾸고, 작성 후 이 안내 주석 블록은 지웁니다.
-->

# DB 설계 — {기능명}

> 담당 서비스: `{service}_service` · DBMS: MySQL
> 공통 네이밍·PK·타임스탬프 규약은 [database.md](../../conventions/database.md)를 따릅니다.

## {N}. {table_name} — {한 줄 설명}

- {테이블 책임 1}
- {테이블 책임 2}

```sql
CREATE TABLE {table_name} (
    id          BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
    {column}    {TYPE}       NOT NULL,
    created_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at  DATETIME,

    CONSTRAINT uq_{table_name}_{column} UNIQUE ({column})
);

CREATE INDEX idx_{table_name}_{column} ON {table_name} ({column});
```
