# 인증 API

---

## GET /api/auth/login/{provider}

**용도**: OAuth 2.0 소셜 로그인 페이지로 리다이렉트합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `provider` | String | Y | `google` \| `kakao` |

### 응답

OAuth 제공자 로그인 페이지로 302 리다이렉트.

---

## GET /api/auth/callback/{provider}

**용도**: OAuth 콜백을 처리하고 토큰을 발급합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `provider` | String | Y | `google` \| `kakao` |

### 응답

Refresh Token을 HttpOnly Cookie에 설정한 뒤 아래 URL로 302 리다이렉트.

```
{redirectBaseUri}?isNewUser=true|false
```

정지 계정이면 토큰을 발급하지 않고 아래 URL로 리다이렉트한다.

```
{redirectBaseUri}?error=USER_SUSPENDED&suspendedUntil=2026-12-09T00:00:00+09:00&reason=SPAM
```

| 쿼리 파라미터 | 타입 | 설명 |
|--------------|------|------|
| `suspendedUntil` | String | 정지 종료 시각(ISO 8601, KST 오프셋). 영구 정지면 생략 |
| `reason` | String | 정지 사유 코드. 신고 사유(`ReportReason`)와 같은 값: `SPAM` \| `ABUSE` \| `SEXUAL` \| `ILLEGAL` \| `COPYRIGHT` \| `PRIVACY` \| `ETC`. 자유 서술 사유는 URL에 넣지 않고(GA4가 전체 URL을 수집) 정지 통지 메일에만 담는다 |

앱(JSON 응답)은 에러 응답에 같은 이름의 `suspendedUntil`·`reason` 필드를 담는다.

| 쿼리 파라미터 | 타입 | 설명 |
|--------------|------|------|
| `isNewUser` | Boolean | `true`: PENDING 역할 (신규 가입 또는 탈퇴 후 새 계정으로 재가입) / `false`: 기존 사용자 |

### 비고

- Refresh Token은 HttpOnly Cookie에 설정 (7일 만료)
- `isNewUser=true` → FE는 회원가입 완료 페이지로 이동, `POST /api/auth/register` 완료 후 Access Token 발급
- `isNewUser=false` → FE는 `POST /api/auth/refresh` 호출 후 홈으로 이동
- SUSPENDED 계정 로그인 시 OAuth2 인증 단계에서 차단, `USER_SUSPENDED` 에러로 실패 핸들러 호출(위 정지 리다이렉트 형식 참고)
- 정지 중 탈퇴한 회원이 보관 기간 내 로그인하면 가입 거부(재가입 제한 에러, 코드명은 BE 구현 시 확정)

---

## POST /api/auth/refresh

**용도**: Refresh Token으로 Access Token을 재발급합니다.

### 요청

Cookie에서 Refresh Token 자동 추출 (별도 바디 없음).

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `accessToken` | String | 새 Access Token |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `REFRESH_TOKEN_EXPIRED` | 401 | Refresh Token 만료 — 재로그인 필요 |
| `REFRESH_TOKEN_INVALID` | 401 | 유효하지 않은 Refresh Token |
| `USER_SUSPENDED` | 403 | 정지된 회원. 에러 응답에 `suspendedUntil`(영구 정지면 생략)·`reason` 포함 |

### 비고

- 호출마다 Refresh Token을 회전(새 값으로 교체)하고 `Set-Cookie`로 새 Refresh Token을 내려준다. 이전 Refresh Token은 즉시 사용 불가
- 세션(기기) 단위로 검증·교체하며 다른 기기 세션에는 영향이 없다 (`policy/auth-policy.md` "다중 기기 로그인 정책")
- 이미 회전된 Refresh Token이 다시 제출되면(재사용) 해당 세션을 폐기하고 `REFRESH_TOKEN_INVALID` — 해당 기기 재로그인 필요. 같은 Refresh Token으로 동시에 요청해도 재사용으로 판정되므로 클라이언트는 refresh 호출을 직렬화해야 한다
- Access Token(`typ=access`)을 Cookie에 넣어 호출하면 `REFRESH_TOKEN_INVALID`
- 정지된 회원이면 `REFRESH_TOKEN_INVALID` 대신 `USER_SUSPENDED`(정지 정보 포함)를 반환한다

---

## POST /api/auth/logout

**용도**: 로그아웃 처리 및 토큰을 무효화합니다. **(인증 필요)**

### 요청

별도 바디 없음.

### 응답

`200 OK` (바디 없음)

### 비고

- Cookie의 Refresh Token으로 현재 기기 세션을 식별해 해당 세션만 삭제 (다른 기기 세션 유지)
- Refresh Token Cookie가 없거나 유효하지 않아도 `200 OK` (세션은 TTL 만료로 정리)
- 현재 Access Token 블랙리스트 등록
- Refresh Token Cookie 삭제

---

## DELETE /api/auth/withdraw

**용도**: 회원 탈퇴 처리합니다. **(인증 필요)**

### 요청

별도 바디 없음.

### 응답

`200 OK` (바디 없음)

### 비고

- 개인정보(이메일·닉네임·출생연도·마케팅 동의) 즉시 파기, 연관 데이터(별점, 캘린더, 팔로우, 문의) 함께 삭제
- 게시글·댓글은 삭제하지 않고 작성자를 "탈퇴 회원"으로 표시
- Kakao 가입자는 연결 끊기(unlink) API 호출
- 시행일(2026-12-02) 이후 탈퇴부터 소셜 로그인 식별자 HMAC 해시를 보관 기간 동안 별도 보관(`auth-policy.md` "회원 탈퇴 및 재가입 제한")
- 해당 사용자의 모든 기기 세션(Refresh Token) 삭제

---

## GET /api/auth/me

**용도**: 로그인한 사용자의 프로필 정보를 조회합니다. **(인증 필요 — USER·ADMIN·PENDING)**

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 사용자 ID |
| `nickname` | String? | 닉네임 (PENDING 상태일 때 null 가능) |
| `birthYear` | Integer? | 출생연도 (PENDING 상태일 때 null) |
| `role` | String | `USER` \| `ADMIN` \| `PENDING` |
| `agreedMarketing` | Boolean | 마케팅 수신 동의 여부 (미동의 또는 PENDING 상태이면 `false`) |

```json
{
  "id": 1,
  "nickname": "라이브덕후",
  "birthYear": 1995,
  "role": "USER",
  "agreedMarketing": true
}
```

---

## PUT /api/auth/me

**용도**: 닉네임을 수정합니다. **(인증 필요)**

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `nickname` | String | N | 닉네임 (최대 20자) |

### 응답

`GET /api/auth/me` 응답과 동일 구조.

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `NICKNAME_TOO_LONG` | 400 | 닉네임 20자 초과 |

---

## PATCH /api/auth/me/marketing

**용도**: 마케팅 수신 동의 여부를 변경합니다. **(인증 필요)**

### 요청

**Content-Type**: `application/json`

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `agreedMarketing` | Boolean | Y | 마케팅 수신 동의 여부 |

### 응답

`204 No Content` (바디 없음)

---

## POST /api/auth/register

**용도**: 회원가입을 완료하고 USER 역할의 Access Token을 발급합니다. **(인증 필요 — PENDING)**

### 요청

**Content-Type**: `application/json`

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `nickname` | String | Y | 닉네임 (최대 20자) |
| `birthYear` | Integer | Y | 출생연도 |
| `agreedTerms` | Boolean | Y | 이용약관 동의 (반드시 `true`) |
| `agreedPrivacy` | Boolean | Y | 개인정보처리방침 동의 (반드시 `true`) |
| `agreedMarketing` | Boolean | N | 마케팅 정보 수신 동의 (선택) |

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `accessToken` | String | USER 역할 Access Token (30분 만료) |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `TERMS_NOT_AGREED` | 400 | 필수 약관 미동의 |
| `NICKNAME_REQUIRED` | 400 | 닉네임 미입력 |
| `NICKNAME_TOO_LONG` | 400 | 닉네임 20자 초과 |
| `BIRTH_YEAR_REQUIRED` | 400 | 출생연도 미입력 |
| `NICKNAME_DUPLICATE` | 409 | 닉네임 중복 |

---

## GET /api/auth/check-nickname

**용도**: 닉네임 중복 여부를 확인합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `nickname` | String | Y | 확인할 닉네임 |

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `available` | Boolean | `true`: 사용 가능 / `false`: 중복 |
