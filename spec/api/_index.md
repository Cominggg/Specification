# API 명세

Base URL: `/api`

인증 필요 엔드포인트는 `Authorization: Bearer {accessToken}` 헤더를 요구합니다.
ADMIN 전용 엔드포인트는 일반 사용자 접근 시 `403` 반환합니다.

인증 열 표기: `불필요`(비로그인 허용) · `필요`(USER·ADMIN) · `ADMIN` · `PENDING 전용`.
가입 미완료(PENDING) 토큰은 `GET /api/auth/me`, `POST /api/auth/logout`·`/api/auth/app/logout`, `DELETE /api/auth/withdraw`, `POST /api/auth/register`만 호출할 수 있고, 그 외 `필요` 엔드포인트는 `403 FORBIDDEN`.

---

## 공통 규칙

### 날짜 및 필드명

| 항목 | 규칙 |
|------|------|
| 날짜 포맷 | `YYYY-MM-DD` (ISO 8601). FE 표시 변환: `YYYY.MM.DD` |
| 필드명 | DB snake_case → 응답 camelCase 변환 |
| `isFollowing` | 인증 요청 시 `user_follow_artist` 기준. 비인증 시 `false` |
| `isInCalendar` | 인증 요청 시 `user_concert_calendar` 기준. 비인증 시 `false` |
| `imageUrl` | `artist.image_url` 값. 이미지가 수집되지 않은 아티스트는 `null` |

주요 DB 컬럼 → 응답 필드 매핑:

| DB 컬럼 | 응답 필드 |
|---------|---------|
| `venue_name` | `venue` |
| `start_date` | `startDate` |
| `end_date` | `endDate` |
| `poster_url` | `posterUrl` |
| `first_release_date` | `releaseDate` |
| `cover_url` | `coverUrl` |

---

### 페이지네이션 응답

페이지네이션이 적용된 모든 엔드포인트는 아래 형태로 응답합니다.

```json
{
  "content": [...],
  "page": 0,
  "size": 20,
  "totalElements": 100,
  "totalPages": 5
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `content` | `T[]` | 데이터 목록 |
| `page` | `int` | 현재 페이지 (0-based) |
| `size` | `int` | 페이지 크기 |
| `totalElements` | `long` | 전체 데이터 수 |
| `totalPages` | `int` | 전체 페이지 수 |

---

### 에러 응답

```json
{
  "code": "CONCERT_NOT_FOUND",
  "message": "존재하지 않는 공연입니다."
}
```

`USER_SUSPENDED`(403)만 예외로 `suspendedUntil`(정지 종료 시각, 오프셋 없는 KST `yyyy-MM-ddTHH:mm:ss`, 영구 정지면 `null`)·`reason`(정지 사유 코드) 필드를 더한다([auth-policy.md](../policy/auth-policy.md) "`USER_SUSPENDED` 응답 형식"). 다른 에러는 위 형식을 유지한다.

매핑되지 않은 경로는 에러 바디 없이 `404`만 반환한다(로그인 상태 기준. 비로그인이면 인증 단계에서 `401 UNAUTHORIZED`가 먼저 반환됨).

쓰기 API(팔로우·별점·캘린더·게시글·댓글·신고·문의 등)는 탈퇴(INACTIVE)했거나 없는 회원의 토큰이면 `USER_NOT_FOUND`(404)를 반환한다. 엔드포인트별 에러 표에는 반복 기재하지 않는다.

요청 본문을 읽을 수 없으면(JSON 문법 오류, 필드 타입 불일치, enum 필드의 허용 밖 값 등) `INVALID_INPUT`(400)을 반환한다. 이때 message는 기본 문구이며 `필드: 사유`는 담기지 않는다.

공통 HTTP 상태 코드:

| 상태 코드 | 대표 code | 발생 조건 |
|----------|----------|---------|
| `400` | `INVALID_INPUT` 외 도메인 코드 | 요청 파라미터·바디 유효성 오류. `@Valid` 바디 오류·필수 파라미터 누락은 message에 `필드: 사유`가 담김 |
| `401` | `UNAUTHORIZED` | 인증 필요 경로에 토큰 없음·만료·위조. 비로그인 허용 경로에서는 잘못된 토큰을 무시하고 비로그인으로 처리 |
| `403` | `FORBIDDEN` | 권한 없음 (ADMIN 전용, PENDING 사용자의 일반 API 접근 등). 정지 회원은 `USER_SUSPENDED` |
| `404` | 도메인 `*_NOT_FOUND` | 리소스 없음 |
| `409` | 도메인 코드 | 중복 요청 |
| `429` | `RATE_LIMIT_EXCEEDED` | IP 단위 요청 한도 초과 (전 엔드포인트) |
| `500` | `INTERNAL_ERROR` | 서버 내부 오류 |

---

## 도메인별 문서

| 도메인 | 파일 |
|--------|------|
| 인증 | [auth.md](auth.md) |
| 아티스트 | [artists.md](artists.md) |
| 공연 | [concerts.md](concerts.md) |
| 캘린더 | [calendar.md](calendar.md) |
| 음악 발매 | [releases.md](releases.md) |
| 마이페이지·문의 | [my.md](my.md) |
| 게시판 | [posts.md](posts.md) |
| 관리자 | [admin.md](admin.md) |
| Data 파이프라인 연동 | [pipeline.md](pipeline.md) |

---

## 전체 엔드포인트 목록

### 인증
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/auth/login/{provider}` | 불필요 |
| GET | `/api/auth/callback/{provider}` | 불필요 |
| POST | `/api/auth/refresh` | 불필요 |
| POST | `/api/auth/logout` | 필요 |
| DELETE | `/api/auth/withdraw` | 필요 |
| GET | `/api/auth/me` | 필요 |
| PUT | `/api/auth/me` | 필요 |
| POST | `/api/auth/login/nonce` | 불필요 |
| POST | `/api/auth/login/{provider}/token` | 불필요 |
| POST | `/api/auth/app/refresh` | 불필요 |
| POST | `/api/auth/app/logout` | 필요 |
| POST | `/api/auth/register` | PENDING 전용 |
| PATCH | `/api/auth/me/marketing` | 필요 |
| GET | `/api/auth/check-nickname` | 불필요 |

### 아티스트
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/artists` | 불필요 |
| GET | `/api/artists/{id}` | 불필요 |
| GET | `/api/artists/{id}/concerts` | 불필요 |
| GET | `/api/artists/{id}/releases` | 불필요 |
| POST | `/api/artists/{id}/follow` | 필요 |
| DELETE | `/api/artists/{id}/follow` | 필요 |
| GET | `/api/artists/following` | 필요 |

### 공연
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/concerts` | 불필요 |
| GET | `/api/concerts/popular` | 불필요 |
| GET | `/api/concerts/ticketing` | 불필요 |
| GET | `/api/concerts/stats` | 불필요 |
| GET | `/api/concerts/{id}` | 불필요 |
| GET | `/api/concerts/{id}/setlist` | 불필요 |
| PUT | `/api/concerts/{id}/rating` | 필요 |
| GET | `/api/concerts/{id}/rating/me` | 필요 |
| DELETE | `/api/concerts/{id}/rating` | 필요 |

### 캘린더
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/calendar` | 불필요 |
| POST | `/api/calendar/{concertId}` | 필요 |
| DELETE | `/api/calendar/{concertId}` | 필요 |

### 음악 발매
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/releases` | 불필요 |
| GET | `/api/releases/{id}` | 불필요 |
| PUT | `/api/releases/{id}/rating` | 필요 |
| GET | `/api/releases/{id}/rating/me` | 필요 |
| DELETE | `/api/releases/{id}/rating` | 필요 |

### 마이페이지·문의
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/me/summary` | 필요 |
| GET | `/api/me/concerts/upcoming` | 필요 |
| GET | `/api/me/concerts/history` | 필요 |
| POST | `/api/inquiries` | 필요 |
| GET | `/api/me/inquiries` | 필요 |
| GET | `/api/me/inquiries/{id}` | 필요 |
| GET | `/api/me/inquiries/exists` | 필요 |

### 게시판
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/mentions/search` | 필요 |
| POST | `/api/posts` | 필요 |
| GET | `/api/posts/{id}` | 불필요 |
| GET | `/api/posts` | 불필요 |
| GET | `/api/posts/popular` | 불필요 |
| GET | `/api/posts/popular-board` | 불필요 |
| GET | `/api/posts/trending-tags` | 불필요 |
| PATCH | `/api/posts/{id}` | 필요 |
| DELETE | `/api/posts/{id}` | 필요 |
| POST | `/api/posts/{id}/recommend` | 필요 |
| DELETE | `/api/posts/{id}/recommend` | 필요 |
| GET | `/api/entities/{type}/{id}/posts` | 불필요 |
| GET | `/api/search` | 불필요 |
| GET | `/api/posts/{id}/comments` | 불필요 |
| POST | `/api/posts/{id}/comments` | 필요 |
| DELETE | `/api/comments/{commentId}` | 필요 |
| POST | `/api/comments/{commentId}/like` | 필요 |
| DELETE | `/api/comments/{commentId}/like` | 필요 |
| POST | `/api/reports` | 필요 |
| GET | `/api/notices` | 불필요 |
| GET | `/api/notices/{id}` | 불필요 |

### 관리자 (ROLE_ADMIN)
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/admin/artists` | ADMIN |
| GET | `/api/admin/artists/{id}` | ADMIN |
| PUT | `/api/admin/artists/{id}` | ADMIN |
| GET | `/api/admin/concerts/excluded` | ADMIN |
| GET | `/api/admin/concerts/pending` | ADMIN |
| POST | `/api/admin/concerts` | ADMIN |
| GET | `/api/admin/concerts/{id}` | ADMIN |
| PUT | `/api/admin/concerts/{id}` | ADMIN |
| PUT | `/api/admin/concerts/{id}/state` | ADMIN |
| PUT | `/api/admin/concerts/{id}/approve` | ADMIN |
| PUT | `/api/admin/concerts/{id}/reject` | ADMIN |
| POST | `/api/admin/concerts/{id}/candidates` | ADMIN |
| DELETE | `/api/admin/concerts/{id}/candidates/{artistId}` | ADMIN |
| POST | `/api/admin/concerts/{id}/artists` | ADMIN |
| DELETE | `/api/admin/concerts/{id}/artists/{artistId}` | ADMIN |

| GET | `/api/admin/inquiries` | ADMIN |
| GET | `/api/admin/inquiries/{id}` | ADMIN |
| PATCH | `/api/admin/inquiries/{id}/status` | ADMIN |
| POST | `/api/admin/policies` | ADMIN |
| GET | `/api/admin/notices` | ADMIN |
| GET | `/api/admin/notices/{id}` | ADMIN |
| POST | `/api/admin/notices` | ADMIN |
| PATCH | `/api/admin/notices/{id}` | ADMIN |
| DELETE | `/api/admin/notices/{id}` | ADMIN |
| GET | `/api/admin/reports` | ADMIN |
| GET | `/api/admin/reports/{id}` | ADMIN |
| PATCH | `/api/admin/reports/{id}/status` | ADMIN |
| GET | `/api/admin/users` | ADMIN |
| POST | `/api/admin/users/{id}/suspension` | ADMIN |
| DELETE | `/api/admin/users/{id}/suspension` | ADMIN |
| DELETE | `/api/admin/users/{id}` | ADMIN |

### Data 파이프라인 연동 (ROLE_ADMIN)
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/admin/data/search/artists` | ADMIN |
| GET | `/api/admin/data/search/concerts` | ADMIN |
| POST | `/api/admin/data/collect/artists` | ADMIN |
| POST | `/api/admin/data/collect/concerts` | ADMIN |
| POST | `/api/admin/data/collect/artists/{id}/releases` | ADMIN |
| POST | `/api/admin/data/collect/concerts/{id}/setlist` | ADMIN |
