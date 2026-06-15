# 공연 API

---

## GET /api/concerts

**용도**: 내한 공연 목록을 필터·페이지네이션으로 조회합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `status` | String | N | — | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` |
| `inCalendar` | Boolean | N | — | `true`이면 내 캘린더에 추가한 공연만 반환 (인증 필요) |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |
| `sort` | String | N | `startDate,desc` | 정렬 기준 (`필드명,방향`) |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 공연 ID |
| `posterUrl` | String? | 포스터 이미지 URL |
| `artistName` | String? | 아티스트명 (`confidence=HIGH` 기준, 없으면 `null`) |
| `title` | String | 공연명 |
| `startDate` | String | 시작일 (`YYYY-MM-DD`) |
| `endDate` | String? | 종료일 (`YYYY-MM-DD`) |
| `venue` | String | 공연장명 |
| `status` | String | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` |
| `isInCalendar` | Boolean | 내 캘린더 추가 여부 (비인증 시 `false`) |

```json
{
  "content": [
    {
      "id": 1,
      "posterUrl": null,
      "artistName": "YOASOBI",
      "title": "YOASOBI ARENA TOUR 2025",
      "startDate": "2025-08-15",
      "endDate": "2025-08-16",
      "venue": "KSPO DOME, 서울",
      "status": "UPCOMING",
      "isInCalendar": false
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 21,
  "totalPages": 2
}
```

### 비고

- 기본 정렬: `startDate` 내림차순 (최신 공연 우선)
- `status` 미전달 시 전체 공연 반환
- 비인증 요청 허용 — `isInCalendar`는 비인증 시 항상 `false`, 인증 시 실제 값 반환
- `inCalendar=true`와 `status` 동시 전달 시 `inCalendar`가 우선 적용되어 `status`는 무시됨

---

## GET /api/concerts/search

**용도**: 공연명·아티스트명·아티스트 alias 키워드로 공연을 검색합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `q` | String | Y | — | 검색어 (빈 문자열·공백 불가) |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |
| `sort` | String | N | `startDate,desc` | 정렬 기준 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목은 `GET /api/concerts`와 동일 구조.

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `VALIDATION_ERROR` | 400 | `q`가 전달되지 않았거나 빈 문자열·공백인 경우 |

### 비고

- 공연명(`title`)·아티스트명(`artist.name`)·아티스트 alias(`artist_alias.name`) 대소문자 무관 LIKE 검색
- `isInCalendar`는 인증 시 실제 값 반환, 비인증 시 `false`

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

## GET /api/concerts/following

**용도**: 내가 팔로우한 아티스트의 공연 목록을 status 조건으로 조회합니다. **(인증 필요)**

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `status` | String | N | — | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` |

### 응답

배열 형태 (페이지네이션 없음). `GET /api/concerts` `content` 항목과 동일 구조.

### 비고

- `status` 미전달 시 전체 공연 반환
- 인증 필수이므로 `isInCalendar`는 항상 실제 값 반환
- 홈 "관심 아티스트 공연" 탭 및 공연 목록 "관심 아티스트만" 필터에서 재사용

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
| `artistName` | String? | 아티스트명 (`confidence=HIGH` 기준, 없으면 `null`) |
| `artistId` | Long? | 아티스트 ID (없으면 `null`) |
| `title` | String | 공연명 |
| `startDate` | String | 시작일 (`YYYY-MM-DD`) |
| `endDate` | String? | 종료일 (`YYYY-MM-DD`) |
| `venue` | String | 공연장명 |
| `status` | String | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` |
| `price` | String? | 가격 정보 |
| `isInCalendar` | Boolean | 내 캘린더 추가 여부 |
| `ticketLinks` | Object[] | 예매처 목록 |
| `ticketLinks[].id` | Long | 예매처 ID (`concert_booking_link.id`) |
| `ticketLinks[].label` | String | 예매처 표시명 (`concert_booking_link.name`) |
| `ticketLinks[].url` | String | 예매처 URL |

```json
{
  "id": 1,
  "posterUrl": "https://...",
  "imageUrls": ["https://..."],
  "artistName": "YOASOBI",
  "artistId": 1,
  "title": "YOASOBI ARENA TOUR 2025",
  "startDate": "2025-08-15",
  "endDate": "2025-08-16",
  "venue": "KSPO DOME, 서울",
  "status": "UPCOMING",
  "price": "전석 165,000원",
  "isInCalendar": false,
  "ticketLinks": [
    { "id": 1, "label": "인터파크", "url": "https://..." }
  ]
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

```json
{
  "tracks": [
    { "order": 1, "title": "Pale Blue" },
    { "order": 2, "title": "KICK BACK" }
  ]
}
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |
