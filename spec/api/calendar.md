# 캘린더 API

---

## GET /api/calendar

**용도**: 특정 월의 전체 공연 캘린더 목록을 조회합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `year` | int | Y | 연도 |
| `month` | int | Y | 월 (1–12) |

### 응답

배열 형태 (페이지네이션 없음):

| 필드 | 타입 | 설명 |
|------|------|------|
| `concertId` | Long | 공연 ID |
| `artistName` | String | 아티스트명 |
| `title` | String | 공연명 |
| `startDate` | String | 시작일 |
| `endDate` | String? | 종료일 |
| `status` | String | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` |
| `posterUrl` | String? | 포스터 이미지 URL |
| `venue` | String | 공연장명 |
| `isInCalendar` | boolean | 내 캘린더 추가 여부 (비인증 시 `false`) |

```json
[
  {
    "concertId": 1,
    "artistName": "YOASOBI",
    "title": "YOASOBI ARENA TOUR 2025",
    "startDate": "2025-08-15",
    "endDate": "2025-08-16",
    "status": "UPCOMING",
    "posterUrl": null,
    "venue": "KSPO DOME, 서울",
    "isInCalendar": false
  }
]
```

### 비고

- 멀티데이 공연: `startDate`~`endDate` 범위의 모든 날짜에 캘린더 도트 표시 (FE 처리)
- 인증 사용자: 실제 추가 여부 반환. 비인증 사용자: 항상 `false`

---

## GET /api/calendar/my

**용도**: 내 캘린더에 저장된 공연 목록을 조회합니다. **(인증 필요)**

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `10` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목은 `GET /api/calendar` 배열 항목과 동일 구조.

### 비고

- 마이페이지 "예정 공연" 탭(MY-03): 이 API 재사용. FE에서 `startDate >= 오늘`인 항목만 표시
- `isInCalendar`는 항상 `true` (저장된 공연만 반환)

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
