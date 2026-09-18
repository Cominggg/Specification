# API 명세

Base URL: `/api`

인증 필요 엔드포인트는 `Authorization: Bearer {accessToken}` 헤더를 요구합니다.
ADMIN 전용 엔드포인트는 일반 사용자 접근 시 `403` 반환합니다.

---

## 공통 규칙

### 날짜 및 필드명

| 항목 | 규칙 |
|------|------|
| 날짜 포맷 | `YYYY-MM-DD` (ISO 8601). FE 표시 변환: `YYYY.MM.DD` |
| 필드명 | DB snake_case → 응답 camelCase 변환 |
| `isFollowing` | 인증 요청 시 `user_follow_artist` 기준. 비인증 시 `false` |
| `isInCalendar` | 인증 요청 시 `user_concert_calendar` 기준. 비인증 시 `false` |
| `imageUrl` | `artist` 테이블에 이미지 컬럼 없음 — 항상 `null` 반환 |

주요 DB 컬럼 → 응답 필드 매핑:

| DB 컬럼 | 응답 필드 |
|---------|---------|
| `venue_name` | `venue` |
| `start_date` | `startDate` |
| `end_date` | `endDate` |
| `poster_url` | `posterUrl` |
| `first_release_date` | `releaseDate` |
| `cover_url` | `coverUrl` |
| `profile_image_url` | `avatarUrl` |

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

공통 HTTP 상태 코드:

| 상태 코드 | 발생 조건 |
|----------|---------|
| `400` | 요청 파라미터·바디 유효성 오류 |
| `401` | 인증 토큰 없음 또는 만료 |
| `403` | 권한 없음 |
| `404` | 리소스 없음 |
| `409` | 중복 요청 |

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
| GET | `/api/concerts/search` | 불필요 |
| GET | `/api/concerts/popular` | 불필요 |
| GET | `/api/concerts/following` | 필요 |
| GET | `/api/concerts/stats` | 불필요 |
| GET | `/api/concerts/{id}` | 불필요 |
| GET | `/api/concerts/{id}/setlist` | 불필요 |

### 캘린더
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/calendar` | 불필요 |
| GET | `/api/calendar/my` | 필요 |
| POST | `/api/calendar/{concertId}` | 필요 |
| DELETE | `/api/calendar/{concertId}` | 필요 |

### 음악 발매
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/releases` | 불필요 |
| GET | `/api/releases/search` | 불필요 |
| GET | `/api/releases/{id}` | 불필요 |

### 마이페이지·문의
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/my/history` | 필요 |
| POST | `/api/inquiries` | 필요 |
| GET | `/api/inquiries/my` | 필요 |
| GET | `/api/inquiries/my/{id}` | 필요 |

### 게시판
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/mentions/search` | 불필요 |
| POST | `/api/posts` | 필요 |
| GET | `/api/posts/{id}` | 불필요 |
| GET | `/api/posts` | 불필요 |
| GET | `/api/posts/popular` | 불필요 |
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

### 관리자 (ROLE_ADMIN)
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| PUT | `/api/admin/artists/{id}` | ADMIN |
| GET | `/api/admin/concerts/pending` | ADMIN |
| PUT | `/api/admin/concerts/{id}` | ADMIN |
| PUT | `/api/admin/concerts/{id}/state` | ADMIN |
| PUT | `/api/admin/concerts/{id}/approve` | ADMIN |
| PUT | `/api/admin/concerts/{id}/reject` | ADMIN |
| POST | `/api/admin/concerts/{id}/artists` | ADMIN |
| GET | `/api/admin/inquiries` | ADMIN |
| GET | `/api/admin/inquiries/{id}` | ADMIN |
| PATCH | `/api/admin/inquiries/{id}/status` | ADMIN |
| POST | `/api/admin/policies` | ADMIN |

### Data 파이프라인 연동 (ROLE_ADMIN)
| 메서드 | 엔드포인트 | 인증 |
|--------|------------|------|
| GET | `/api/admin/data/search/artists` | ADMIN |
| GET | `/api/admin/data/search/concerts` | ADMIN |
| POST | `/api/admin/data/collect/artists` | ADMIN |
| POST | `/api/admin/data/collect/concerts` | ADMIN |
| POST | `/api/admin/data/collect/artists/{id}/releases` | ADMIN |
| POST | `/api/admin/data/collect/concerts/{id}/setlist` | ADMIN |
