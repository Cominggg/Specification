# 아티스트 API

---

## GET /api/artists

**용도**: 아티스트 목록을 조회합니다. 이름 검색을 지원합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `name` | String | N | — | 이름 검색어 (부분 일치) |
| `page` | int | N | `0` | 페이지 번호 (0-based) |
| `size` | int | N | `25` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 아티스트 ID |
| `name` | String | 아티스트명 |
| `imageUrl` | String? | 프로필 이미지 URL (항상 `null`) |
| `hasUpcomingConcert` | Boolean | 예정 내한 공연 여부 (`artist.is_coming`) |
| `isFollowing` | Boolean | 팔로우 여부 (비인증 시 `false`) |

```json
{
  "content": [
    {
      "id": 1,
      "name": "YOASOBI",
      "imageUrl": null,
      "hasUpcomingConcert": true,
      "isFollowing": false
    }
  ],
  "page": 0,
  "size": 25,
  "totalElements": 27,
  "totalPages": 2
}
```

### 비고

- 검색어 변경 시 FE에서 `page=0`으로 초기화

---

## GET /api/artists/{id}

**용도**: 아티스트 상세 정보를 조회합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 아티스트 ID |

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 아티스트 ID |
| `name` | String | 아티스트명 |
| `imageUrl` | String? | 프로필 이미지 URL (항상 `null`) |
| `hasUpcomingConcert` | Boolean | 예정 내한 공연 여부 |
| `isFollowing` | Boolean | 팔로우 여부 (비인증 시 `false`) |
| `followersCount` | int | 팔로워 수 (`user_follow_artist` COUNT) |
| `debutDate` | String? | 데뷔일 (`YYYY-MM-DD`) |
| `links` | Object[] | 외부 링크 목록 |
| `links[].id` | String | 링크 타입 식별자 (`artist_url.type`) |
| `links[].label` | String | 표시 이름 (예: `"Spotify"`) |
| `links[].url` | String | URL |

```json
{
  "id": 1,
  "name": "YOASOBI",
  "imageUrl": null,
  "hasUpcomingConcert": true,
  "isFollowing": false,
  "followersCount": 24800,
  "debutDate": "2019-09-12",
  "links": [
    { "id": "spotify", "label": "Spotify", "url": "https://..." },
    { "id": "youtube", "label": "YouTube", "url": "https://..." }
  ]
}
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `ARTIST_NOT_FOUND` | 404 | 존재하지 않는 아티스트 |

### 비고

- `links[].label`은 `artist_url.type`을 BE에서 사람이 읽기 좋은 이름으로 변환해 반환

---

## GET /api/artists/{id}/concerts

**용도**: 아티스트의 내한 공연 목록을 조회합니다. (2020년 이후)

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 아티스트 ID |

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `tab` | String | N | `all` | `all` \| `upcoming` \| `past` |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `10` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 공연 ID |
| `title` | String | 공연명 |
| `startDate` | String | 시작일 |
| `endDate` | String? | 종료일 |
| `venue` | String | 공연장명 |
| `status` | String | `공연예정` \| `공연중` \| `공연완료` \| `공연취소` |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `ARTIST_NOT_FOUND` | 404 | 존재하지 않는 아티스트 |

### 비고

- `upcoming`: status가 `UPCOMING` 또는 `ONGOING`인 항목
- `past`: status가 `ENDED` 또는 `CANCELLED`인 항목

---

## GET /api/artists/{id}/releases

**용도**: 아티스트의 디스코그래피(앨범·싱글·EP)를 수록곡 포함해 조회합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 아티스트 ID |

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `type` | String | N | — | `ALBUM` \| `SINGLE` \| `EP` (복수 허용) |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `10` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 릴리즈 ID |
| `title` | String | 앨범·싱글·EP 타이틀 |
| `type` | String | `ALBUM` \| `SINGLE` \| `EP` |
| `releaseDate` | String? | 발매일 |
| `coverUrl` | String? | 커버 이미지 URL |
| `tracks` | Object[] | 수록곡 목록 |
| `tracks[].position` | int | 트랙 순서 |
| `tracks[].title` | String | 트랙 제목 |
| `tracks[].lengthMs` | int? | 재생 시간 (ms) |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `ARTIST_NOT_FOUND` | 404 | 존재하지 않는 아티스트 |

---

## POST /api/artists/{id}/follow

**용도**: 아티스트를 팔로우합니다. **(인증 필요)**

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 아티스트 ID |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `ARTIST_NOT_FOUND` | 404 | 존재하지 않는 아티스트 |
| `ALREADY_FOLLOWING` | 409 | 이미 팔로우한 아티스트 |

---

## DELETE /api/artists/{id}/follow

**용도**: 아티스트 팔로우를 취소합니다. **(인증 필요)**

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 아티스트 ID |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `ARTIST_NOT_FOUND` | 404 | 존재하지 않는 아티스트 |
| `NOT_FOLLOWING` | 400 | 팔로우하지 않은 아티스트 |

---

## GET /api/artists/following

**용도**: 내가 팔로우한 아티스트 목록을 조회합니다. **(인증 필요)**

### 요청

별도 파라미터 없음.

### 응답

배열 형태 (페이지네이션 없음):

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 아티스트 ID |
| `name` | String | 아티스트명 |
| `imageUrl` | String? | 프로필 이미지 URL (항상 `null`) |
| `hasUpcomingConcert` | Boolean | 예정 내한 공연 여부 |

### 비고

- 마이페이지 "관심 아티스트" 탭(MY-02)에서 재사용
