# 마이페이지·문의 API

---

## GET /api/me/summary

**용도**: MY 첫 화면에 필요한 숫자(팔로잉 아티스트·예정 공연·다녀온 공연·처리 중 문의)와 다음 공연 1건을 한 번에 조회합니다. **(인증 필요)**

### 요청

파라미터 없음.

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `followingCount` | long | 팔로잉 아티스트 수. `GET /api/artists?following=true`의 `totalElements`와 같은 조건 |
| `upcomingCount` | long | 예정·진행 중 공연 수. `GET /api/me/concerts/upcoming`의 `totalElements`와 같은 조건 |
| `nextConcert` | Object? | 예정·진행 중 공연 중 `startDate` 오름차순 첫 항목. `GET /api/me/concerts/upcoming` `content` 항목과 동일 구조(식별자 필드 `concertId`, `posterUrl` 포함). 없으면 `null` |
| `historyCount` | long | 다녀온 공연 수. `GET /api/me/concerts/history`의 `totalElements`와 같은 조건 |
| `inProgressInquiryCount` | long | 처리 중 문의 수. `GET /api/me/inquiries?status=IN_PROGRESS`의 `totalElements`와 같은 조건 |

```json
{
  "followingCount": 12,
  "upcomingCount": 3,
  "nextConcert": {
    "concertId": 1,
    "type": "CONCERT",
    "artists": [{ "id": 2, "name": "Kenshi Yonezu", "koreanName": "요네즈 켄시" }],
    "title": "요네즈 켄시 KOREA LIVE 2026",
    "startDate": "2026-11-14",
    "endDate": "2026-11-15",
    "status": "UPCOMING",
    "posterUrl": "https://...",
    "venue": "KSPO DOME",
    "isInCalendar": true,
    "ticketOpenAt": null
  },
  "historyCount": 8,
  "inProgressInquiryCount": 1
}
```

### 비고

- 각 숫자는 대응하는 목록 API와 같은 조건(오늘 기준, PENDING·EXCLUDED 공연 제외)으로 센다. 목록 API를 `size=1`로 호출해 `totalElements`를 읽던 용도를 대체한다

---

## GET /api/me/concerts/upcoming

**용도**: 내 캘린더에 저장한 공연 중 아직 종료되지 않은 공연 목록을 조회합니다. **(인증 필요)**

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `10` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목은 `GET /api/calendar` 배열 항목과 동일 구조.

### 비고

- `endDate >= 오늘`인 항목만 반환 — UPCOMING·ONGOING 공연 모두 포함
- `type`은 항상 `"CONCERT"` (저장된 공연 기준, 티켓팅 항목 없음)
- `isInCalendar`는 항상 `true`
- 기본 정렬: `startDate` 오름차순
- PENDING·EXCLUDED 공연은 캘린더에 담겨 있어도 제외

---

## GET /api/me/concerts/history

**용도**: 내 캘린더에 저장한 공연 중 이미 종료된 공연 목록을 조회합니다. **(인증 필요)**

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `10` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 공연 ID |
| `artists` | Object[] | 출연 아티스트 목록 (연결된 아티스트가 없으면 빈 배열) |
| `artists[].id` | Long | 아티스트 ID |
| `artists[].name` | String | 아티스트명 |
| `artists[].koreanName` | String? | 한글 표기명 (`locale=ko` alias, 없으면 `null`) |
| `title` | String | 공연명 |
| `startDate` | String | 시작일 (`YYYY-MM-DD`) |
| `endDate` | String | 종료일 (`YYYY-MM-DD`) |
| `venue` | String | 공연장명 |
| `posterUrl` | String? | 공연 포스터 URL (없으면 `null`) |
| `status` | String | 공연 상태 |

### 비고

- `endDate < 오늘`인 항목만 반환 (BE 필터링)
- 기본 정렬: `startDate` 내림차순
- PENDING·EXCLUDED 공연은 캘린더에 담겨 있어도 제외


---

## POST /api/inquiries

**용도**: 데이터 문의를 등록합니다. **(인증 필요)**

### 요청

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `type` | String | Y | `ARTIST` \| `CONCERT` \| `SETLIST` \| `DATA_REQUEST` \| `FEEDBACK` |
| `targetId` | Long | 조건부 | 문의 대상 ID (아티스트 ID 또는 공연 ID). `ARTIST`·`CONCERT`·`SETLIST`는 필수, `DATA_REQUEST`·`FEEDBACK`은 생략(`null`) |
| `title` | String | Y | 문의 제목 (최대 255자) |
| `content` | String | Y | 문의 내용 |

```json
{
  "type": "CONCERT",
  "targetId": 1,
  "title": "예매처 링크가 잘못되어 있습니다",
  "content": "인터파크 링크가 404 오류를 반환합니다."
}
```

### 응답

`201 Created` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INVALID_INPUT` | 400 | `type`·`title`·`content` 누락, `type` 허용 밖 값, `title` 255자 초과, `ARTIST`·`CONCERT`·`SETLIST`인데 `targetId` 누락 |
| `INQUIRY_ALREADY_PENDING` | 409 | 동일 `targetId`로 PENDING 상태 문의 존재 (`ARTIST`·`CONCERT`·`SETLIST`만 해당) |
| `TARGET_NOT_FOUND` | 404 | 존재하지 않는 대상 ID (`ARTIST`·`CONCERT`·`SETLIST`만 해당) |
| `USER_SUSPENDED` | 403 | 정지 중인 회원. 에러 응답에 `suspendedUntil`·`reason` 포함 ([auth-policy.md](../policy/auth-policy.md) "`USER_SUSPENDED` 응답 형식") |

### 비고

- 공연 정보 문의(INQ-01): `type=CONCERT`, `targetId=concertId`
- 아티스트 정보 문의(INQ-02): `type=ARTIST`, `targetId=artistId`
- 셋리스트 문의(INQ-03): `type=SETLIST`, `targetId=concertId`
- 데이터 요청(INQ-05): `type=DATA_REQUEST`, `targetId` 없음
- 피드백(INQ-06): `type=FEEDBACK`, `targetId` 없음
- `DATA_REQUEST`·`FEEDBACK`은 대상 존재 검증과 PENDING 중복 제한을 하지 않는다 (`targetId`를 보내면 검증 없이 그대로 저장)

---

## GET /api/me/inquiries

**용도**: 내가 등록한 문의 목록을 조회합니다. **(인증 필요)**

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `status` | String | N | — | `PENDING` \| `IN_PROGRESS` \| `RESOLVED` \| `REJECTED` |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `10` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 문의 ID |
| `type` | String | `ARTIST` \| `CONCERT` \| `SETLIST` \| `DATA_REQUEST` \| `FEEDBACK` |
| `title` | String | 문의 제목 |
| `status` | String | `PENDING` \| `IN_PROGRESS` \| `RESOLVED` \| `REJECTED` |
| `createdAt` | String | 등록일 (`YYYY-MM-DD`) |
| `resultMessage` | String? | 처리 결과 (`status=RESOLVED`일 때) |
| `rejectReason` | String? | 반려 사유 (`status=REJECTED`일 때) |

```json
{
  "content": [
    {
      "id": 1,
      "type": "CONCERT",
      "title": "예매처 링크 오류",
      "status": "RESOLVED",
      "createdAt": "2025-08-20",
      "resultMessage": "예매처 링크를 수정하였습니다.",
      "rejectReason": null
    }
  ],
  "page": 0,
  "size": 10,
  "totalElements": 5,
  "totalPages": 1
}
```

### 비고

- `resultMessage`와 `rejectReason`은 DB `inquiry.admin_note` 단일 컬럼을 status 기준으로 분리해 반환
- 해당 없는 필드는 `null`

---

## GET /api/me/inquiries/{id}

**용도**: 내 문의 상세를 조회합니다. **(인증 필요)**

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 문의 ID |

### 응답

`GET /api/me/inquiries` `content` 항목과 동일 구조에 아래 필드 추가:

| 필드 | 타입 | 설명 |
|------|------|------|
| `content` | String | 문의 내용 |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INQUIRY_NOT_FOUND` | 404 | 존재하지 않는 문의 또는 접근 권한 없음 |
---

## GET /api/me/inquiries/exists

**용도**: 특정 대상에 대해 현재 사용자의 PENDING 상태 문의가 존재하는지 조회합니다. **(인증 필요)**

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `type` | String | Y | `ARTIST` \| `CONCERT` \| `SETLIST` \| `DATA_REQUEST` \| `FEEDBACK` |
| `targetId` | Long | Y | 문의 대상 ID |

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `exists` | Boolean | PENDING 문의 존재 여부 |

```json
{ "exists": true }
```

### 비고

- `POST /api/inquiries` 호출 전 중복 등록 여부를 사전 확인하는 용도
- `exists=true`이면 해당 사용자·유형·대상에 이미 PENDING 문의가 존재함
- `DATA_REQUEST`·`FEEDBACK`은 항상 `exists=false` (중복 제한이 없어 사전 확인 불필요). 이 경우에도 `targetId`는 필수 파라미터

