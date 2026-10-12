<!--
이 파일은 기능 스펙용 API 명세 템플릿입니다. (선택 — API 설계가 필요한 기능에서만 추가)
새 기능을 시작할 때 docs/specs/{NNN}-{feature-name}/api.md 로 복사해 작성하세요.
공통 규약(응답 Envelope·에러 코드·페이지네이션 등)의 단일 출처는 docs/conventions/api-design.md이며,
이 문서는 그 규약을 따르는 "이 기능의 엔드포인트 목록과 상세"만 적습니다.
{ } 안의 placeholder는 실제 값으로 바꾸고, 작성 후 이 안내 주석 블록은 지웁니다.
-->

# API 명세서 — {기능명}

## 공통 규약

> 전역 규약은 [api-design.md](../../conventions/api-design.md)를 따릅니다. 아래는 이 문서에서만 참고할 값이 있을 때만 적습니다.

- **Base URL** : `/api/v1/{resource}`
- **담당 서비스** : `{service}_service`
- **인증** : 보호 엔드포인트는 `Authorization: Bearer <accessToken>` 헤더 필요

---

## 엔드포인트 목록

| 기능 | Method | Path | 설명 | 인증 |
|------|--------|------|------|------|
| {기능 ID} | {GET/POST/...} | `/api/v1/{resource}` | {설명} | Y/N |

---

## 상세 명세

### `{METHOD} {PATH}` — {엔드포인트 이름}

- **인증** : {필요/불필요}
- **설명** : {무엇을 하는 API인지}
- **요청**
  ```json
  { }
  ```
- **응답**
  ```json
  { "code": "SUCCESS", "status": 200, "message": "OK", "data": { } }
  ```
- **에러 케이스**: {어떤 상황에 어떤 에러 코드}
