# 캘린더 API

---

## GET /api/calendar

**용도**: 특정 월의 공연 캘린더 및 티켓팅 일정을 통합 조회합니다. 공연 날짜(`startDate~endDate`) 기준 항목과 티켓 오픈 날짜(`ticketOpenAt`) 기준 항목을 모두 반환하며, `type` 필드로 구분됩니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `year` | int | Y | 연도 |
| `month` | int | Y | 월 (1–12) |

### 응답

배열 형태 (페이지네이션 없음). 날짜 기준 오름차순 정렬:

| 필드 | 타입 | 설명 |
|------|------|------|
| `concertId` | Long | 공연 ID |
| `type` | String | `CONCERT` (공연 날짜 기준) \| `TICKETING` (티켓 오픈 날짜 기준) |
| `artistName` | String? | 아티스트명 |
| `title` | String | 공연명 |
| `startDate` | String | 공연 시작일 (`YYYY-MM-DD`) |
| `endDate` | String? | 공연 종료일 (`YYYY-MM-DD`) |
| `status` | String | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` |
| `posterUrl` | String? | 포스터 이미지 URL |
| `venue` | String | 공연장명 |
| `isInCalendar` | boolean | 내 캘린더 추가 여부 (비인증 시 `false`) |
| `ticketOpenAt` | String? | 티켓 오픈 일시 (ISO 8601 datetime). `TICKETING` 타입에서는 항상 non-null |

```json
[
  {
    "concertId": 2,
    "type": "TICKETING",
    "artistName": "뉴진스",
    "title": "NewJeans 콘서트",
    "startDate": "2025-09-01",
    "endDate": "2025-09-01",
    "status": "UPCOMING",
    "posterUrl": null,
    "venue": "KSPO DOME, 서울",
    "isInCalendar": false,
    "ticketOpenAt": "2025-08-05T10:00:00"
  },
  {
    "concertId": 1,
    "type": "CONCERT",
    "artistName": "YOASOBI",
    "title": "YOASOBI ARENA TOUR 2025",
    "startDate": "2025-08-15",
    "endDate": "2025-08-16",
    "status": "UPCOMING",
    "posterUrl": null,
    "venue": "KSPO DOME, 서울",
    "isInCalendar": false,
    "ticketOpenAt": null
  }
]
```

### 비고

- **`CONCERT` 타입**: 해당 월에 `startDate~endDate` 범위가 걸치는 공연. 캘린더 기준 날짜는 `startDate`.
- **`TICKETING` 타입**: 해당 월에 `ticketOpenAt`이 속하는 공연. 캘린더 기준 날짜는 `ticketOpenAt`.
- 동일 공연이 같은 달에 두 타입으로 모두 등장할 수 있음 (`concertId`로 동일 공연 여부 확인)
- 전체 목록은 캘린더 기준 날짜 오름차순 정렬
- 인증 사용자: `isInCalendar` 실제 값 반환. 비인증: 항상 `false`

---

## POST /api/calendar/{concertId}

**용도**: 내 캘린더에 공연을 추가합니다. **(인증 필요)**

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `concertId` | Long | Y | 추가할 공연 ID |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |
| `ALREADY_IN_CALENDAR` | 409 | 이미 캘린더에 추가된 공연 |

---

## DELETE /api/calendar/{concertId}

**용도**: 내 캘린더에서 공연을 제거합니다. **(인증 필요)**

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `concertId` | Long | Y | 제거할 공연 ID |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `NOT_IN_CALENDAR` | 400 | 캘린더에 없는 공연 |

