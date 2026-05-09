# 마이페이지·문의 API

---

## GET /api/my/history

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
| `artistName` | String | 아티스트명 |
| `title` | String | 공연명 |
| `startDate` | String | 시작일 |
| `endDate` | String? | 종료일 |
| `venue` | String | 공연장명 |
| `status` | String | 공연 상태 |

### 비고

- `startDate < 오늘`인 항목만 반환 (BE 필터링)

---

## POST /api/inquiries

**용도**: 데이터 문의를 등록합니다. **(인증 필요)**

### 요청

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `type` | String | Y | `ARTIST` \| `CONCERT` \| `SETLIST` |
| `targetId` | Long | Y | 문의 대상 ID (아티스트 ID 또는 공연 ID) |
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
| `INQUIRY_ALREADY_PENDING` | 409 | 동일 `targetId`로 PENDING 상태 문의 존재 |
| `TARGET_NOT_FOUND` | 404 | 존재하지 않는 대상 ID |

### 비고

- 공연 정보 문의(INQ-01): `type=CONCERT`, `targetId=concertId`
- 아티스트 정보 문의(INQ-02): `type=ARTIST`, `targetId=artistId`
- 셋리스트 문의(INQ-03): `type=SETLIST`, `targetId=concertId`

---

## GET /api/inquiries/my

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
| `type` | String | `ARTIST` \| `CONCERT` \| `SETLIST` |
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

## GET /api/inquiries/my/{id}

**용도**: 내 문의 상세를 조회합니다. **(인증 필요)**

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 문의 ID |

### 응답

`GET /api/inquiries/my` `content` 항목과 동일 구조에 아래 필드 추가:

| 필드 | 타입 | 설명 |
|------|------|------|
| `content` | String | 문의 내용 |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INQUIRY_NOT_FOUND` | 404 | 존재하지 않는 문의 또는 접근 권한 없음 |
