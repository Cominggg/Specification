# 공연 API

---

## GET /api/concerts

**용도**: 내한 공연 목록을 필터·페이지네이션으로 조회합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `status` | String | N | — | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` |
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
      "status": "UPCOMING"
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

---

## GET /api/concerts/popular

**용도**: 조회수 기반 인기 공연 목록을 조회합니다.

### 요청

별도 파라미터 없음.

### 응답

배열 형태 (페이지네이션 없음). `GET /api/concerts` `content` 항목과 동일 구조.

### 비고

- 고정 건수 반환 (size 파라미터 없음)
- 홈 캐러셀: 상위 5건 FE 슬라이싱
- 홈 "인기 공연" 섹션: desktop 3건 / mobile 6건 FE 슬라이싱

---

## GET /api/concerts/following

**용도**: 내가 팔로우한 아티스트의 예정 공연 목록을 조회합니다. **(인증 필요)**

### 요청

별도 파라미터 없음.

### 응답

배열 형태 (페이지네이션 없음). `GET /api/concerts` `content` 항목과 동일 구조.

### 비고

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
| `thumbnailUrl` | String? | 대표 이미지 URL (`concert.poster_url`) |
| `posterUrls` | String[] | 공연 정보 이미지 목록 |
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
  "thumbnailUrl": "https://...",
  "posterUrls": ["https://..."],
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
- `posterUrls`: DB `poster_url` 단일값을 1-element 배열로 래핑. `null`이면 `[]`

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
