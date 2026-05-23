# 관리자 API (ROLE_ADMIN)

모든 엔드포인트는 `ROLE_ADMIN` 권한이 필요합니다. 일반 사용자 접근 시 `403` 반환.

---
## POST /api/admin/artists

**용도**: 아티스트를 수동 등록합니다.

### 요청

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `mbid` | String | Y | MusicBrainz MBID (UUID) |
| `name` | String | Y | 아티스트명 |
| `sortName` | String | N | 정렬용 이름 |
| `debutDate` | String | N | 데뷔일 (`YYYY-MM-DD`) |

### 응답

`201 Created` (바디 없음)

---

## PUT /api/admin/artists/{id}

**용도**: 아티스트 정보를 수정합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 아티스트 ID |

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `name` | String | N | 아티스트명 |
| `sortName` | String | N | 정렬용 이름 |
| `debutDate` | String | N | 데뷔일 (`YYYY-MM-DD`) |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `ARTIST_NOT_FOUND` | 404 | 존재하지 않는 아티스트 |

---

## POST /api/admin/concerts

**용도**: 공연을 수동 등록합니다.

### 요청

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `kopisId` | String | Y | KOPIS 공연 ID |
| `title` | String | Y | 공연명 |
| `cast` | String | N | 출연진 |
| `startDate` | String | Y | 시작일 (`YYYY-MM-DD`) |
| `endDate` | String | Y | 종료일 (`YYYY-MM-DD`) |
| `venueName` | String | Y | 공연장명 |
| `venueAddress` | String | N | 공연장 주소 |
| `posterUrl` | String | N | 포스터 이미지 URL |
| `price` | String | N | 가격 정보 |
| `status` | String | Y | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` |
| `artistIds` | Long[] | N | 매핑할 아티스트 ID 목록 |

### 응답

`201 Created` (바디 없음)

---

## PUT /api/admin/concerts/{id}

**용도**: 공연 정보를 수정합니다. (상태 변경 불포함)

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `title` | String | N | 공연명 |
| `cast` | String | N | 출연진 |
| `startDate` | String | N | 시작일 (`YYYY-MM-DD`) |
| `endDate` | String | N | 종료일 (`YYYY-MM-DD`) |
| `venueName` | String | N | 공연장명 |
| `venueAddress` | String | N | 공연장 주소 |
| `posterUrl` | String | N | 포스터 이미지 URL |
| `price` | String | N | 가격 정보 |
| `bookingLinks` | Object[] | N | 예매처 링크 목록. `null`이면 기존 링크 유지, `[]`이면 전체 삭제 |
| `bookingLinks[].name` | String | Y | 예매처 이름 |
| `bookingLinks[].url` | String | Y | 예매처 URL |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |

### 비고

- 상태 변경은 `PUT /api/admin/concerts/{id}/state`로만 처리

---

## DELETE /api/admin/concerts/{id}

**용도**: 공연을 삭제합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |

### 비고

- 연관 데이터(예매처 링크·아티스트 매핑·유저 캘린더·셋리스트) 함께 삭제 (cascade)
- 복구 불가

---

## PUT /api/admin/concerts/{id}/state

**용도**: 공연 상태를 강제 변경합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `status` | String | Y | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` |
| `reason` | String | N | 변경 사유 |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |

### 비고

- 변경 이력 로그 기록

---

## GET /api/admin/inquiries

**용도**: 전체 문의 목록을 조회합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `type` | String | N | — | `ARTIST` \| `CONCERT` \| `SETLIST` |
| `status` | String | N | — | `PENDING` \| `IN_PROGRESS` \| `RESOLVED` \| `REJECTED` |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

`GET /api/inquiries/my` `content` 항목과 동일 구조에 아래 필드 추가:

| 필드 | 타입 | 설명 |
|------|------|------|
| `userId` | Long | 문의 등록 사용자 ID |
| `userNickname` | String | 사용자 닉네임 |

---

## GET /api/admin/inquiries/{id}

**용도**: 문의 상세를 조회합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 문의 ID |

### 응답

`GET /api/admin/inquiries` `content` 항목과 동일 구조에 아래 필드 추가:

| 필드 | 타입 | 설명 |
|------|------|------|
| `content` | String | 문의 내용 |
| `targetId` | Long | 문의 대상 ID |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INQUIRY_NOT_FOUND` | 404 | 존재하지 않는 문의 |

---

## PATCH /api/admin/inquiries/{id}/status

**용도**: 문의 처리 상태를 변경합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 문의 ID |

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `status` | String | Y | `IN_PROGRESS` \| `RESOLVED` \| `REJECTED` |
| `adminNote` | String | N | 처리 결과 또는 반려 사유 (`inquiry.admin_note`에 저장) |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INQUIRY_NOT_FOUND` | 404 | 존재하지 않는 문의 |
