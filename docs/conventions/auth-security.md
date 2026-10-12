# [컨벤션] 인증 & 보안

> **범위**: 로그인 방식·토큰 전략·서비스 간 인증 신뢰 전파·시크릿 관리·CORS
> **변경 시**: 이 문서를 단일 출처로 갱신합니다 ([AGENT_GUIDE 문서 동기화](../AGENT_GUIDE.md#문서-동기화-필수) 참고).

## 결정 사항

| 항목 | 결정 | 비고 |
|------|------|------|
| 로그인 방식 | 이메일/비밀번호 + JWT | 소셜 로그인은 범위 밖 |
| 토큰 전략 | Access + Refresh (JWT) | 발급·검증은 `auth_service` |
| Access token TTL | 30분 | 서명: HS256 |
| Refresh token TTL | 14일 (Rotation) | |
| Refresh token 저장 | Redis `refresh:{userId}` | 클라이언트 + Redis 양쪽 보관 |
| 서비스 간 인증 신뢰 전파 | Gateway Passport Token | `gateway_service`가 JWT를 검증한 뒤 `X-Passport` 헤더로 전달. 다운스트림 서비스는 JWT를 재검증하지 않음 |
| Passport 전달 방식 | `X-Passport` 헤더 (Base64URL JSON + HMAC-SHA256 서명) | 외부에서 온 동일 헤더는 Gateway가 무조건 제거 후 재생성 |
| 시크릿 관리 | local `.env` / prod AWS SSM Parameter Store / CI GitHub Actions Secrets + AWS OIDC | |
| CORS 정책 | 허용 도메인 화이트리스트 | |

## 규칙 / 적용

### 1. 로그인 방식

- 비밀번호 해시: BCrypt (strength 10).
- 로그인 실패 응답은 이메일 존재 여부를 구분하지 않는다("이메일 또는 비밀번호가 올바르지 않습니다").
- 로그인 실패 5회 시 해당 계정 로그인을 10분간 제한한다(Redis 카운터).
- 권한: `USER`, `ADMIN` 두 가지.

**이유**: 포트폴리오 핵심은 대규모 트래픽 처리이므로 인증은 표준 방식으로 단순하게 유지한다. 실패 응답을 구분하면 가입 이메일 목록을 수집당할 수 있다.

### 2. 토큰 TTL

| 토큰 | TTL | 저장 위치 |
|------|-----|-----------|
| Access Token | 30분 | 클라이언트 |
| Refresh Token | 14일 | 클라이언트 + Redis |

- Access Token 서명: HS256.
- Access Token claim: `sub`(userId), `role`, `exp`, `jti`.
- Refresh Token은 Redis에 `refresh:{userId}` 키로 저장, TTL 14일.
- 재발급 시 Refresh Token을 교체(rotation)한다. 이전 토큰으로 재발급 요청이 오면 해당 사용자의 Refresh Token을 삭제하고 재로그인을 요구한다.
- 로그아웃 시: Redis의 Refresh Token을 삭제하고, Access Token의 `jti`를 `blacklist:{jti}` 키로 남은 만료 시간만큼 등록한다.
- Gateway는 Access Token 검증 시 블랙리스트를 확인한다.

**이유**: 특가 오픈 시간대에 재로그인이 몰리지 않도록 Access Token을 너무 짧게 두지 않는다. Rotation으로 탈취된 Refresh Token의 재사용을 탐지한다. JWT는 서버에서 무효화할 수 없으므로 로그아웃은 블랙리스트로 처리한다.

### 3. Passport 전달 방식

JWT는 Gateway에서만 검증하고, 내부 서비스에는 `X-Passport` 헤더로 사용자 정보를 전달한다.

```text
Client ──(Authorization: Bearer JWT)──▶ Gateway
Gateway: JWT 검증 → 블랙리스트 확인 → Passport 생성
Gateway ──(X-Passport)──▶ 내부 서비스
```

**Passport 형식**

```json
{ "userId": 1, "role": "USER", "issuedAt": "2026-10-16T09:00:00Z" }
```

Base64URL 인코딩한 JSON + HMAC-SHA256 서명: `{payload}.{signature}`.

**규칙**

- **외부 요청에 포함된 `X-Passport` 헤더는 Gateway에서 무조건 제거한 뒤 새로 생성한다.**
- 내부 서비스는 JWT를 직접 파싱·검증하지 않는다.
- 내부 서비스는 Passport 서명을 검증한 뒤 사용한다(`common` 모듈의 `PassportResolver` — [backend.md](./backend.md)의 common 모듈 정책 참고).
- Controller에서는 `@CurrentUser passport: Passport`로 주입받는다.
- 내부 서비스는 외부에 직접 노출하지 않는다(Gateway를 통해서만 접근).
- Kafka 이벤트에는 Passport를 싣지 않고 `userId`만 포함한다.
- 서비스 간 동기 호출 시 받은 `X-Passport`를 그대로 전달한다.

**이유**: 서비스마다 JWT를 검증하면 키 관리와 블랙리스트 조회가 중복된다. 외부에서 온 헤더를 제거하지 않으면 헤더 위조만으로 다른 사용자로 행세할 수 있다. Gateway를 우회한 내부 호출까지 막기 위해 Passport에 서명을 붙인다.

### 4. 시크릿 저장소

| 환경 | 저장소 |
|------|--------|
| local | `.env` (gitignore) |
| prod | AWS SSM Parameter Store (`SecureString`) |
| CI | GitHub Actions Secrets + AWS OIDC |

**관리 대상**: `JWT_SECRET`, `PASSPORT_SECRET`, `TOSS_SECRET_KEY`, `TOSS_CLIENT_KEY`, `DB_PASSWORD`, `REDIS_PASSWORD`, `SLACK_WEBHOOK_URL`.

**규칙**

- `application.yml`에는 `${ENV_NAME}` 참조만 쓴다. 기본값에도 실제 시크릿을 넣지 않는다.
- `.env.example`에 키 이름만 적어 커밋한다.
- SSM 경로: `/ssg-deal/{env}/{KEY}` (예: `/ssg-deal/prod/TOSS_SECRET_KEY`).
- GitHub Actions에서 AWS 액세스 키를 저장하지 않고 OIDC로 Role을 assume한다.
- 로그에 토큰, 비밀번호, 카드·결제 정보를 출력하지 않는다.
- 커밋 전 gitleaks로 시크릿 유출을 검사한다.

**이유**: 포트폴리오 저장소는 public이므로 시크릿 유출 위험이 크다. 개인 프로젝트 비용 기준으로 Secrets Manager보다 SSM Parameter Store가 저렴하다. 장기 액세스 키는 유출 시 피해가 크므로 OIDC로 단기 자격 증명을 쓴다.

## 관련 문서

- [백엔드 스택](./backend.md) — Gateway 구성, common 모듈
- [인프라 & 배포](./infra-deploy.md) — AWS·CI/CD
- [외부 API & 데이터 연동](./external-apis.md) — Toss Payments
