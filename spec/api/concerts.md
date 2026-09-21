# 공연 API

---

## GET /api/concerts

**용도**: 내한 공연 목록을 검색어·필터·페이지네이션으로 조회합니다. 공연명·아티스트명 검색과 관심 아티스트 필터를 이 엔드포인트 하나로 통합 제공합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `q` | String | N | — | 검색어 (공연명·아티스트명·아티스트 alias 대소문자 무관 부분 일치). 생략·공백이면 텍스트 조건 없이 나머지 필터만 적용 |
| `status` | String | N | — | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` |
| `inCalendar` | Boolean | N | — | `true`이면 내 캘린더에 추가한 공연만 반환 (인증 필요, 미인증 시 빈 페이지) |
| `followedOnly` | Boolean | N | — | `true`이면 팔로우한 아티스트의 공연만 반환 (미인증·팔로잉 없으면 빈 페이지) |
| `ticketOpenPending` | Boolean | N | `false` | `true`이면 티켓 오픈 예정(`ticketOpenAt > 현재 시각`) 공연만 반환 |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |
| `sort` | String | N | `startDate,desc` | 정렬 기준 (`필드명,방향`). 허용 필드: `startDate`, `ticketOpenAt` |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 공연 ID |
| `posterUrl` | String? | 포스터 이미지 URL |
| `artists` | Object[] | 참여 아티스트 목록 (다중 아티스트 가능, 없으면 `[]`) |
| `artists[].artistId` | Long | 아티스트 ID |
| `artists[].name` | String | 아티스트명 |
| `artists[].koreanName` | String? | 한글 표기명 (없으면 `null`) |
| `title` | String | 공연명 |
| `startDate` | String | 시작일 (`YYYY-MM-DD`) |
| `endDate` | String? | 종료일 (`YYYY-MM-DD`) |
| `venue` | String | 공연장명 |
| `status` | String | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` |
| `isInCalendar` | Boolean | 내 캘린더 추가 여부 (비인증 시 `false`) |
| `ticketOpenAt` | String? | 티켓 오픈 일시 (ISO 8601 datetime, 관리자 입력값, 미입력 시 `null`) |
| `averageRating` | Double? | 평균 별점 (0.5~5.0, 소수 첫째 자리 반올림). 등록된 별점이 없으면 `null` |
| `ratingCount` | long | 등록된 별점 개수 (없으면 `0`) |

```json
{
  "content": [
    {
      "id": 1,
      "posterUrl": null,
      "artists": [
        { "artistId": 1, "name": "YOASOBI", "koreanName": null }
      ],
      "title": "YOASOBI ARENA TOUR 2025",
      "startDate": "2025-08-15",
      "endDate": "2025-08-16",
      "venue": "KSPO DOME, 서울",
      "status": "UPCOMING",
      "isInCalendar": false,
      "ticketOpenAt": "2025-07-01T10:00:00",
      "averageRating": 4.5,
      "ratingCount": 12
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 21,
  "totalPages": 2
}
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INVALID_INPUT` | 400 | `sort` 필드가 `startDate`·`ticketOpenAt` 외의 값인 경우 |

### 비고

- 기본 정렬: `startDate` 내림차순 (최신 공연 우선)
- `status` 미전달 시 전체 공연 반환. `EXCLUDED`·`PENDING` 상태는 `status` 값과 무관하게 항상 숨겨짐
- `q`·`status`·`inCalendar`·`followedOnly`·`ticketOpenPending`은 모두 AND로 조합됨
- 비인증 요청 허용 — `isInCalendar`는 비인증 시 항상 `false`, 인증 시 실제 값 반환
- 기존 `GET /api/concerts/search`(공연 검색), `GET /api/concerts/following`(관심 아티스트 공연)는 이 엔드포인트로 통합되어 제거됨 — 각각 `q`, `followedOnly=true` 파라미터로 대체

---

## GET /api/concerts/popular

**용도**: 조회수 기반 인기 공연 목록을 조회합니다.

### 요청

별도 파라미터 없음.

### 응답

배열 형태 (페이지네이션 없음). `GET /api/concerts` `content` 항목과 동일 구조.

### 비고

- BE 반환 건수: 상위 10건 (`view_count` 내림차순)
- 홈 캐러셀: 상위 5건 FE 슬라이싱
- 홈 "인기 공연" 섹션: desktop 3건 / mobile 6건 FE 슬라이싱

---

## GET /api/concerts/ticketing

**용도**: 티켓 오픈 예정 공연 목록을 조회합니다. (`ticket_open_at > 현재 시각` 기준, 최대 20건)

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `following` | Boolean | N | `false` | `true`이면 팔로우한 아티스트의 공연만 반환 (인증 필요) |

### 응답

배열 형태 (최대 20건). `GET /api/concerts` `content` 항목과 동일 구조.

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `UNAUTHORIZED` | 401 | `following=true`이고 비인증 요청인 경우 |

### 비고

- `ticket_open_at` 오름차순 정렬 (오픈 임박 순)
- `following=false` 또는 파라미터 미전달: 비인증 요청 허용
- `following=true`: 인증 필수. 팔로우 아티스트가 없으면 빈 배열 반환

---

## GET /api/concerts/stats

**용도**: 특정 월의 공연 건수를 조회합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `year` | int | Y | 연도 |
| `month` | int | Y | 월 (1–12) |

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `concertCount` | int | 해당 월 공연 건수 |

```json
{ "concertCount": 12 }
```

### 비고

- 홈 통계 배너 미사용으로 엔드포인트 유지만
- `month`가 1–12 범위를 벗어나면 `400 Bad Request` 반환

---

## GET /api/concerts/{id}

**용도**: 공연 상세 정보를 조회합니다. 호출 시 조회수가 1 증가합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 공연 ID |
| `posterUrl` | String? | 포스터 이미지 URL (`concert.poster_url`) |
| `imageUrls` | String[] | 스틸컷 이미지 목록 (`concert_image` 테이블) |
| `artists` | Object[] | 참여 아티스트 목록 (다중 아티스트 가능, 없으면 `[]`) |
| `artists[].artistId` | Long | 아티스트 ID |
| `artists[].name` | String | 아티스트명 |
| `artists[].koreanName` | String? | 한글 표기명 (없으면 `null`) |
| `title` | String | 공연명 |
| `startDate` | String | 시작일 (`YYYY-MM-DD`) |
| `endDate` | String? | 종료일 (`YYYY-MM-DD`) |
| `venue` | String | 공연장명 |
| `status` | String | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` |
| `price` | String? | 가격 정보 |
| `isInCalendar` | Boolean | 내 캘린더 추가 여부 |
| `ticketOpenAt` | String? | 티켓 오픈 일시 (ISO 8601 datetime, 미입력 시 `null`) |
| `ticketLinks` | Object[] | 예매처 목록 |
| `ticketLinks[].id` | Long | 예매처 ID (`concert_booking_link.id`) |
| `ticketLinks[].label` | String | 예매처 표시명 (`concert_booking_link.name`) |
| `ticketLinks[].url` | String | 예매처 URL |
| `averageRating` | Double? | 평균 별점 (0.5~5.0, 소수 첫째 자리 반올림). 등록된 별점이 없으면 `null` |
| `ratingCount` | long | 등록된 별점 개수 (없으면 `0`) |

```json
{
  "id": 1,
  "posterUrl": "https://...",
  "imageUrls": ["https://..."],
  "artists": [
    { "artistId": 1, "name": "YOASOBI", "koreanName": null }
  ],
  "title": "YOASOBI ARENA TOUR 2025",
  "startDate": "2025-08-15",
  "endDate": "2025-08-16",
  "venue": "KSPO DOME, 서울",
  "status": "UPCOMING",
  "price": "전석 165,000원",
  "isInCalendar": false,
  "ticketOpenAt": "2025-07-01T10:00:00",
  "ticketLinks": [
    { "id": 1, "label": "인터파크", "url": "https://..." }
  ],
  "averageRating": 4.5,
  "ratingCount": 12
}
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |

### 비고

- 비인증 요청에서도 호출 가능하나 `isInCalendar`는 항상 `false`
- `imageUrls`: `concert_image` 테이블에서 `position` 오름차순으로 반환. 없으면 `[]`

---

## GET /api/concerts/{id}/setlist

**용도**: 공연의 셋리스트를 조회합니다. (공연 완료 후 제공)

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `tracks` | Object[] | 트랙 목록 (없으면 `[]`) |
| `tracks[].order` | int | 순서 (`setlist_track.position`) |
| `tracks[].title` | String | 곡명 (`setlist_track.song_name`) |
| `sourceUrl` | String? | setlist.fm 원본 URL (`setlist.attribution_url`). 셋리스트 데이터 미존재 시 `null` |

```json
{
  "tracks": [
    { "order": 1, "title": "Pale Blue" },
    { "order": 2, "title": "KICK BACK" }
  ],
  "sourceUrl": "https://www.setlist.fm/setlist/yoasobi/2025/kspo-dome-seoul-south-korea-1a2b3c4d.html"
}
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |

### 비고

- `sourceUrl`이 `null`인 경우(셋리스트 데이터 없음) FE는 `https://www.setlist.fm/`으로 폴백

---

## PUT /api/concerts/{id}/rating

**용도**: 공연에 별점을 등록하거나 수정합니다. 등록·수정이 하나의 API로 통합되어 있어, 이미 등록된 별점이 있으면 갱신합니다.

### 요청

인증 필요.

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |

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
| `CONCERT_NOT_ENDED` | 400 | 공연 상태가 `ENDED`(공연완료)가 아닌 경우 |
| `RATING_TARGET_NOT_FOUND` | 404 | 존재하지 않는 공연 |
| `UNAUTHORIZED` | 401 | 비인증 요청 |

### 비고

- 공연은 상태가 `ENDED`일 때만 별점을 등록·수정할 수 있음 (음악 발매 별점과 다른 정책 — [releases.md](releases.md) 참고)
- 사용자당 공연당 별점 1개만 유지됨 (동일 사용자가 재요청하면 기존 값 갱신)

---

## GET /api/concerts/{id}/rating/me

**용도**: 로그인한 사용자가 해당 공연에 등록한 자신의 별점을 조회합니다.

### 요청

인증 필요.

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |

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

## DELETE /api/concerts/{id}/rating

**용도**: 등록한 별점을 취소합니다.

### 요청

인증 필요.

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |

### 응답

`200 OK`, 본문 없음.

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `RATING_NOT_FOUND` | 404 | 등록된 별점이 없는 경우 |
| `UNAUTHORIZED` | 401 | 비인증 요청 |
