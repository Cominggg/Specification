# 음악 발매 API

---

## GET /api/releases

**용도**: 음악 목록을 검색어·필터·페이지네이션으로 조회합니다. 음악 검색 기능을 이 엔드포인트 하나로 통합 제공합니다. (REL-03, REL-04)

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `q` | String | N | — | 검색어 (릴리즈명·트랙명·아티스트명·alias 대소문자 무관 부분 일치). 생략·공백이면 텍스트 조건 없이 나머지 필터만 적용 |
| `artistId` | Long | N | — | 특정 아티스트 필터 |
| `type` | String | N | — | `Album` \| `Single`. 그 외 값 → 400. 생략 시 전체 타입 |
| `following` | Boolean | N | — | `true`이면 팔로우 아티스트 릴리즈만 반환. 미인증이면 빈 페이지 반환 |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 릴리즈 ID |
| `coverUrl` | String? | 커버 이미지 URL |
| `artistName` | String | 아티스트명 |
| `title` | String | 타이틀 |
| `type` | String | `Album` \| `Single` 또는 기타 |
| `releaseDate` | String? | 발매일 |
| `spotifyId` | String? | Spotify 릴리즈 ID (`release_group.spotify_id`) |
| `averageRating` | Double? | 평균 별점 (0.5~5.0, 소수 첫째 자리 반올림). 등록된 별점이 없으면 `null` |
| `ratingCount` | long | 등록된 별점 개수 (없으면 `0`) |

```json
{
  "content": [
    {
      "id": 1,
      "coverUrl": null,
      "artistName": "Kenshi Yonezu",
      "title": "LOST CORNER",
      "type": "Album",
      "releaseDate": "2024-08-28",
      "spotifyId": "4CPMuGYB4TP0BdHiWlVMOA",
      "averageRating": 4.5,
      "ratingCount": 8
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 12,
  "totalPages": 1
}
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INVALID_INPUT` | 400 | `type`에 `Album`·`Single` 외 값 전달 시 |

### 비고

- 정렬: `releaseDate` DESC NULLS LAST 고정 (발매일 없는 항목은 항상 마지막)
- 검색 대상(`q`): 릴리즈명, 수록 트랙명, 아티스트명, 아티스트 alias (대소문자 무시 부분 일치)
- `q`·`artistId`·`type`·`following`은 모두 조합 가능. `following=true`이면 `artistId` 필터는 무시되고 `q`·`type`은 그대로 적용
- 홈 "새 앨범·싱글" 섹션: `size=8`로 재사용
- 기존 `GET /api/releases/search`(음악 검색)는 이 엔드포인트로 통합되어 제거됨 — `q` 파라미터로 대체
- `type`은 이제 `Album`·`Single`만 허용하며 그 외 값은 400을 반환함 (기존에는 `GET /api/releases`가 임의의 문자열을 허용했음)

---

## GET /api/releases/{id}

**용도**: 릴리즈 상세 정보를 트랙리스트·커버 포함해 조회합니다. (REL-02)

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 릴리즈 ID |

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 릴리즈 ID |
| `title` | String | 타이틀 |
| `type` | String | 릴리즈 타입 |
| `releaseDate` | String? | 발매일 |
| `coverUrl` | String? | 커버 이미지 URL |
| `label` | String? | 레이블 |
| `totalTracks` | int? | 전체 트랙 수 |
| `artistId` | Long | 아티스트 ID |
| `artistName` | String | 아티스트명 |
| `spotifyId` | String? | Spotify 릴리즈 ID (`release_group.spotify_id`) |
| `tracks` | Object[] | 수록곡 목록 |
| `tracks[].position` | int | 트랙 순서 |
| `tracks[].title` | String | 트랙 제목 |
| `tracks[].lengthMs` | int? | 재생 시간 (ms) |
| `tracks[].discNumber` | int? | 디스크 번호 (멀티 디스크 앨범) |
| `tracks[].explicit` | Boolean? | 명시적 콘텐츠 여부 |
| `tracks[].spotifyId` | String? | Spotify 트랙 ID (`track.spotify_id`) |
| `averageRating` | Double? | 평균 별점 (0.5~5.0, 소수 첫째 자리 반올림). 등록된 별점이 없으면 `null` |
| `ratingCount` | long | 등록된 별점 개수 (없으면 `0`) |

```json
{
  "id": 1,
  "title": "LOST CORNER",
  "type": "Album",
  "releaseDate": "2024-08-28",
  "coverUrl": null,
  "label": "Sony Music",
  "totalTracks": 13,
  "artistId": 2,
  "artistName": "Kenshi Yonezu",
  "spotifyId": "4CPMuGYB4TP0BdHiWlVMOA",
  "tracks": [
    { "position": 1, "title": "LOST CORNER", "lengthMs": 262000, "discNumber": 1, "explicit": false, "spotifyId": "1YHEuThB7dJjBBjDaJZJyO" },
    { "position": 2, "title": "LADY", "lengthMs": 238000, "discNumber": 1, "explicit": false, "spotifyId": null }
  ],
  "averageRating": 4.5,
  "ratingCount": 8
}
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `RELEASE_NOT_FOUND` | 404 | 존재하지 않는 릴리즈 |

---

## PUT /api/releases/{id}/rating

**용도**: 음악 발매(릴리즈)에 별점을 등록하거나 수정합니다. 등록·수정이 하나의 API로 통합되어 있어, 이미 등록된 별점이 있으면 갱신합니다.

### 요청

인증 필요.

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 릴리즈 ID |

**Body**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `score` | BigDecimal | Y | 0.5~5.0 범위의 0.5 단위 점수 |

```json
{ "score": 4.5 }
```

### 응답

`200 OK`, 본문 없음.

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INVALID_RATING_SCORE` | 400 | `score`가 0.5~5.0 범위를 벗어나거나 0.5 단위가 아닌 경우 |
| `RATING_TARGET_NOT_FOUND` | 404 | 존재하지 않는 릴리즈 |
| `UNAUTHORIZED` | 401 | 비인증 요청 |

### 비고

- 공연 별점([concerts.md](concerts.md) 참고)과 달리, 릴리즈 별점은 상태 제약 없이 항상 등록·수정 가능
- 사용자당 릴리즈당 별점 1개만 유지됨 (동일 사용자가 재요청하면 기존 값 갱신)

---

## GET /api/releases/{id}/rating/me

**용도**: 로그인한 사용자가 해당 릴리즈에 등록한 자신의 별점을 조회합니다.

### 요청

인증 필요.

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 릴리즈 ID |

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `score` | BigDecimal? | 등록한 별점. 등록한 적 없으면 `null` |

```json
{ "score": 4.5 }
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `UNAUTHORIZED` | 401 | 비인증 요청 |

---

## DELETE /api/releases/{id}/rating

**용도**: 등록한 별점을 취소합니다.

### 요청

인증 필요.

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 릴리즈 ID |

### 응답

`200 OK`, 본문 없음.

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `RATING_NOT_FOUND` | 404 | 등록된 별점이 없는 경우 |
| `UNAUTHORIZED` | 401 | 비인증 요청 |
