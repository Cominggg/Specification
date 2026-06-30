# 아티스트 API

---

## GET /api/artists

**용도**: 아티스트 목록을 조회합니다. 이름 검색, isComing 필터, 팔로잉 필터를 지원합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `name` | String | N | — | 이름 검색어 (부분 일치, 아티스트명 및 alias 포함) |
| `isComing` | Boolean | N | — | `true`/`false`이면 예정 내한 공연 여부(`is_coming`)로 필터. 미전달 시 전체 조회 |
| `following` | Boolean | N | — | `true`이면 팔로잉 아티스트만 조회. 미인증 또는 팔로잉 없으면 빈 페이지 반환 |
| `page` | int | N | `0` | 페이지 번호 (0-based) |
| `size` | int | N | `25` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 아티스트 ID |
| `name` | String | 아티스트명 |
| `imageUrl` | String? | 프로필 이미지 URL |
| `hasUpcomingConcert` | Boolean | 예정 내한 공연 여부 (`artist.is_coming`) |
| `isFollowing` | Boolean | 팔로우 여부 (비인증 시 `false`) |
| `spotifyUrl` | String? | Spotify 아티스트 URL (없으면 `null`) |

```json
{
  "content": [
    {
      "id": 1,
      "name": "YOASOBI",
      "imageUrl": null,
      "hasUpcomingConcert": true,
      "isFollowing": false,
      "spotifyUrl": "https://open.spotify.com/artist/..."
    }
  ],
  "page": 0,
  "size": 25,
  "totalElements": 27,
  "totalPages": 2
}
```

### 비고

- `name`, `isComing`, `following` 세 파라미터는 모두 독립적으로 조합 가능
- `following=true` 시 인증 없거나 팔로잉 아티스트가 없으면 빈 페이지 반환
- 검색어 변경 시 FE에서 `page=0`으로 초기화
- `name` 검색은 `artist.name` 및 `artist_alias.name` 모두 포함 (대소문자 무시)

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
| `imageUrl` | String? | 프로필 이미지 URL |
| `hasUpcomingConcert` | Boolean | 예정 내한 공연 여부 |
| `isFollowing` | Boolean | 팔로우 여부 (비인증 시 `false`) |
| `followersCount` | int | 팔로워 수 (`user_follow_artist` COUNT) |
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
| `status` | String | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `ARTIST_NOT_FOUND` | 404 | 존재하지 않는 아티스트 |

### 비고

- `upcoming`: status가 `UPCOMING` 또는 `ONGOING`인 항목
- `past`: status가 `ENDED` 또는 `CANCELLED`인 항목

---

## GET /api/artists/{id}/releases

**용도**: 아티스트의 디스코그래피(앨범·싱글)를 수록곡 포함해 조회합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 아티스트 ID |

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `type` | String | N | — | `Album` \| `Single` (복수 허용) |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `10` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 릴리즈 ID |
| `title` | String | 앨범·싱글 타이틀 |
| `type` | String | `Album` \| `Single` |
| `releaseDate` | String? | 발매일 |
| `coverUrl` | String? | 커버 이미지 URL |
| `spotifyId` | String? | Spotify 릴리즈 ID (`release_group.spotify_id`) |
| `tracks` | Object[] | 수록곡 목록 |
| `tracks[].position` | int | 트랙 순서 |
| `tracks[].title` | String | 트랙 제목 |
| `tracks[].lengthMs` | int? | 재생 시간 (ms) |
| `tracks[].discNumber` | int? | 디스크 번호 (멀티 디스크 앨범) |
| `tracks[].explicit` | Boolean? | 명시적 콘텐츠 여부 |
| `tracks[].spotifyId` | String? | Spotify 트랙 ID (`track.spotify_id`) |

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
| `imageUrl` | String? | 프로필 이미지 URL |
| `hasUpcomingConcert` | Boolean | 예정 내한 공연 여부 |
| `isFollowing` | Boolean | 팔로우 여부 (항상 `true`) |

### 비고

- 마이페이지 "관심 아티스트" 탭(MY-02)에서 재사용
- `isFollowing`은 이 엔드포인트가 팔로우한 아티스트만 반환하므로 항상 `true`
