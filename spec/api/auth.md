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

**용도**: OAuth 콜백을 처리하고 JWT를 발급합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `provider` | String | Y | `google` \| `kakao` |

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `accessToken` | String | Access Token (30분 만료) |

### 비고

- Refresh Token은 HttpOnly Cookie에 설정 (7일 만료)
- 로그인 성공 후 FE는 `redirect_uri` 파라미터 경로로 이동

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

**용도**: 로그인한 사용자의 프로필 정보를 조회합니다. **(인증 필요)**

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 사용자 ID |
| `nickname` | String | 닉네임 |
| `avatarUrl` | String? | 프로필 이미지 URL |
| `role` | String | `USER` \| `ADMIN` |

```json
{
  "id": 1,
  "nickname": "라이브덕후",
  "avatarUrl": null,
  "role": "USER"
}
```

---

## PUT /api/auth/me

**용도**: 닉네임·프로필 이미지를 수정합니다. **(인증 필요)**

### 요청

**Content-Type**: `multipart/form-data`

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `nickname` | String | N | 닉네임 (최대 20자) |
| `profileImage` | File | N | 이미지 파일 (jpg·png·webp, 최대 5MB) |

### 응답

`GET /api/auth/me` 응답과 동일 구조.

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INVALID_FILE_TYPE` | 400 | 허용되지 않는 파일 형식 |
| `FILE_TOO_LARGE` | 400 | 파일 크기 5MB 초과 |
| `NICKNAME_TOO_LONG` | 400 | 닉네임 20자 초과 |
