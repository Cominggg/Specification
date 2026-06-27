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

| 쿼리 파라미터 | 타입 | 설명 |
|--------------|------|------|
| `isNewUser` | Boolean | `true`: PENDING 역할 (신규 가입 또는 탈퇴 후 재가입) / `false`: 기존 사용자 |

### 비고

- Refresh Token은 HttpOnly Cookie에 설정 (7일 만료)
- `isNewUser=true` → FE는 회원가입 완료 페이지로 이동, `POST /api/auth/register` 완료 후 Access Token 발급
- `isNewUser=false` → FE는 `POST /api/auth/refresh` 호출 후 홈으로 이동
- SUSPENDED 계정 로그인 시 OAuth2 인증 단계에서 차단, `USER_SUSPENDED` 에러로 실패 핸들러 호출

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

---

## POST /api/auth/logout

**용도**: 로그아웃 처리 및 토큰을 무효화합니다. **(인증 필요)**

### 요청

별도 바디 없음.

### 응답

`200 OK` (바디 없음)

### 비고

- Refresh Token Cookie 삭제
- 서버 측 토큰 블랙리스트 처리

---

## DELETE /api/auth/withdraw

**용도**: 회원 탈퇴 처리합니다. **(인증 필요)**

### 요청

별도 바디 없음.

### 응답

`200 OK` (바디 없음)

### 비고

- 연관 데이터(캘린더, 팔로우, 문의) 함께 삭제

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

```json
{
  "id": 1,
  "nickname": "라이브덕후",
  "birthYear": 1995,
  "role": "USER"
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
