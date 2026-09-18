# 관리자 API (ROLE_ADMIN)

모든 엔드포인트는 `ROLE_ADMIN` 권한이 필요합니다. 일반 사용자 접근 시 `403` 반환.

---

## GET /api/admin/artists

**용도**: DB에 등록된 아티스트를 이름(별칭 포함)으로 검색합니다. EXCLUDED 공연에 아티스트를 수동 연결할 때 `artistId` 확인 용도입니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `name` | String | Y | — | 검색할 아티스트명 (대소문자 무시, 별칭 포함 부분 일치) |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 아티스트 ID |
| `name` | String | 아티스트명 |

### 비고

- `GET /api/admin/data/search/artists`(MusicBrainz 외부 검색)와 다름 — DB 등록 아티스트 전용

---

## GET /api/admin/artists/{id}

**용도**: 어드민 전용 아티스트 단건 조회. alias를 포함한 편집 폼 초기값 로드에 사용합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 아티스트 ID |

### 응답

`200 OK`

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 아티스트 ID |
| `name` | String | 아티스트명 |
| `aliases.ja` | String[] | 일본어 alias 목록 (없으면 `[]`) |
| `aliases.en` | String[] | 영어 alias 목록 (없으면 `[]`) |
| `aliases.ko` | String[] | 한국어 alias 목록 (없으면 `[]`) |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `ARTIST_NOT_FOUND` | 404 | 존재하지 않는 아티스트 |

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
| `aliases` | Object | N | locale별 alias 목록. 미전달 시 변경 없음 |
| `aliases.ja` | String[] | N | 일본어 alias 목록. 빈 배열 `[]` 전달 시 전체 삭제, 배열 전달 시 교체 |
| `aliases.en` | String[] | N | 영어 alias 목록. 빈 배열 `[]` 전달 시 전체 삭제, 배열 전달 시 교체 |
| `aliases.ko` | String[] | N | 한국어 alias 목록. 빈 배열 `[]` 전달 시 전체 삭제, 배열 전달 시 교체 |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `ARTIST_NOT_FOUND` | 404 | 존재하지 않는 아티스트 |

### 비고

- `aliases` 필드 자체를 미전달하면 alias 변경 없음
- `aliases.ja` 등 각 locale 필드는 항상 세 필드 모두 전송 권장
- 공백 문자열은 저장되지 않음 (서버에서 필터링)

---

## GET /api/admin/concerts/excluded

**용도**: EXCLUDED 상태의 공연 목록을 조회합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 공연 ID |
| `title` | String | 공연명 |
| `startDate` | String | 시작일 (`YYYY-MM-DD`) |
| `endDate` | String | 종료일 (`YYYY-MM-DD`) |
| `venueName` | String | 공연장명 |
| `posterUrl` | String? | 포스터 URL |
| `artists` | Object[] | 연결된 아티스트 목록 |
| `artists[].artistId` | Long | 아티스트 ID |
| `artists[].name` | String | 아티스트명 |

### 비고

- `rejectConcert()`로 EXCLUDED된 공연은 `concert_artist`가 삭제되어 `artists` 빈 배열로 반환

---

## GET /api/admin/concerts/pending

**용도**: 관리자 검토 대기 중인 PENDING 상태 공연 목록을 조회합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 공연 ID |
| `title` | String | 공연명 |
| `startDate` | String | 시작일 (`YYYY-MM-DD`) |
| `endDate` | String | 종료일 (`YYYY-MM-DD`) |
| `venueName` | String | 공연장명 |
| `posterUrl` | String? | 포스터 URL |
| `ticketOpenAt` | String? | 티켓 오픈 일시 (ISO 8601 datetime, 미입력 시 `null`) |
| `bookingLinks` | Object[] | 예매 링크 목록 (없으면 `[]`) |
| `bookingLinks[].name` | String | 예매처 이름 |
| `bookingLinks[].url` | String | 예매처 URL |
| `candidates` | Object[] | 후보 아티스트 목록 |
| `candidates[].artistId` | Long | 아티스트 ID |
| `candidates[].name` | String | 아티스트명 |
| `candidates[].matchedBy` | String | 매칭 방법 (`prfcast` / `prfnm` / `manual`) |

---

## GET /api/admin/concerts/{id}

**용도**: 관리자 전용 공연 단건 조회. EXCLUDED·PENDING을 포함한 모든 status의 공연을 조회합니다. 공연 수정 폼 데이터 로드에 사용합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |

### 응답

`200 OK`

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 공연 ID |
| `title` | String | 공연명 |
| `cast` | String? | 출연진 |
| `startDate` | String | 시작일 (`YYYY-MM-DD`) |
| `endDate` | String | 종료일 (`YYYY-MM-DD`) |
| `venueName` | String | 공연장명 |
| `posterUrl` | String? | 포스터 URL |
| `price` | String? | 가격 정보 |
| `status` | String | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` \| `EXCLUDED` \| `PENDING` |
| `ticketOpenAt` | String? | 티켓 오픈 일시 (ISO 8601 datetime, 미입력 시 `null`) |
| `bookingLinks` | Object[] | 예매처 링크 목록 |
| `bookingLinks[].name` | String | 예매처 이름 |
| `bookingLinks[].url` | String | 예매처 URL |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |

### 비고

- 공개 API `GET /api/concerts/{id}`와 달리 EXCLUDED·PENDING 상태도 정상 반환

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
| `posterUrl` | String | N | 포스터 이미지 URL |
| `price` | String | N | 가격 정보 |
| `ticketOpenAt` | String? | N | 티켓 오픈 일시 (ISO 8601 datetime, 예: `2026-06-20T10:00:00`). `null` 또는 필드 생략 시 기존 값이 `null`로 초기화됨 (다른 필드와 달리 null-reset 지원) |
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
| `status` | String | Y | `UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED` \| `EXCLUDED` |
| `reason` | String | N | 변경 사유 |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |

### 비고

- `EXCLUDED` → `UPCOMING` / `ONGOING`으로 복원 시 연결된 아티스트의 `is_coming` 자동 갱신

---

## PUT /api/admin/concerts/{id}/approve

**용도**: PENDING 상태의 공연을 승인합니다. `concert_artist_candidate`를 `concert_artist`로 이동하고, 날짜 기반으로 status를 자동 계산합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |

**Request Body** (`application/json`, 선택 — 바디 전체 생략 가능)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `ticketOpenAt` | String? | N | 티켓 오픈 일시 (ISO 8601 datetime, 예: `2026-07-01T10:00:00`). `null` 또는 생략 시 미설정 |
| `bookingLinks` | Object[] | N | 예매 링크 목록. `null` 또는 생략 시 링크 미생성, `[]` 이면 no-op |
| `bookingLinks[].name` | String | Y | 예매처 이름 |
| `bookingLinks[].url` | String | Y | 예매처 URL |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |
| `CONCERT_NOT_PENDING` | 400 | PENDING 상태가 아닌 공연 |

### 비고

- 승인 시 날짜 기반 status 자동 계산: `today < startDate` → `UPCOMING`, `today ≤ endDate` → `ONGOING`, 그 외 → `ENDED`
- 관련 아티스트의 `is_coming` 자동 동기화

---

## PUT /api/admin/concerts/{id}/reject

**용도**: PENDING 상태의 공연을 거절합니다. 공연 status를 `EXCLUDED`로 변경하고 후보 아티스트(`concert_artist_candidate`) 및 확정 매핑(`concert_artist`) 레코드를 삭제합니다.

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
| `CONCERT_NOT_PENDING` | 400 | PENDING 상태가 아닌 공연 |

---

## POST /api/admin/concerts/{id}/candidates

**용도**: PENDING 공연의 후보 아티스트(`concert_artist_candidate`)를 추가합니다. PENDING 공연 검토 중 파이프라인 매칭 결과를 보완하는 용도입니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `artistId` | Long | Y | 추가할 아티스트 ID |

### 응답

`201 Created` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |
| `CONCERT_NOT_PENDING` | 400 | PENDING 상태가 아닌 공연 |
| `ARTIST_NOT_FOUND` | 404 | 존재하지 않는 아티스트 |
| `CONCERT_ARTIST_ALREADY_EXISTS` | 409 | 이미 후보로 등록된 아티스트 |

### 비고

- PENDING 전용. 확정 공연(UPCOMING·ONGOING·ENDED·EXCLUDED)에는 `POST /api/admin/concerts/{id}/artists` 사용

---

## DELETE /api/admin/concerts/{id}/candidates/{artistId}

**용도**: PENDING 공연의 후보 아티스트(`concert_artist_candidate`)를 제거합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |
| `artistId` | Long | Y | 제거할 아티스트 ID |

### 응답

`204 No Content`

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |
| `CONCERT_NOT_PENDING` | 400 | PENDING 상태가 아닌 공연 |
| `ARTIST_NOT_FOUND` | 404 | 존재하지 않는 아티스트 |
| `CONCERT_ARTIST_NOT_FOUND` | 404 | 해당 공연의 후보로 등록되지 않은 아티스트 |

### 비고

- PENDING 전용. 확정 공연에는 `DELETE /api/admin/concerts/{id}/artists/{artistId}` 사용

---

## POST /api/admin/concerts/{id}/artists

**용도**: 확정 공연(UPCOMING·ONGOING·ENDED·EXCLUDED)에 아티스트를 직접 매핑합니다. `concert_artist`에 바로 저장합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `artistId` | Long | Y | 지정할 아티스트 ID |

### 응답

`201 Created` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |
| `CONCERT_IS_PENDING` | 400 | PENDING 상태의 공연 — `/candidates` 엔드포인트를 사용해야 함 |
| `ARTIST_NOT_FOUND` | 404 | 존재하지 않는 아티스트 |
| `CONCERT_ARTIST_ALREADY_EXISTS` | 409 | 이미 매핑된 아티스트 |

### 비고

- PENDING 공연에는 `POST /api/admin/concerts/{id}/candidates` 사용

---

## DELETE /api/admin/concerts/{id}/artists/{artistId}

**용도**: 확정 공연(UPCOMING·ONGOING·ENDED·EXCLUDED)에서 아티스트 매핑을 제거합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID |
| `artistId` | Long | Y | 제거할 아티스트 ID |

### 응답

`204 No Content`

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | 존재하지 않는 공연 |
| `CONCERT_IS_PENDING` | 400 | PENDING 상태의 공연 — `/candidates` 엔드포인트를 사용해야 함 |
| `ARTIST_NOT_FOUND` | 404 | 존재하지 않는 아티스트 |
| `CONCERT_ARTIST_NOT_FOUND` | 404 | 해당 공연에 매핑되지 않은 아티스트 |

### 비고

- 공연이 UPCOMING / ONGOING 상태일 경우 해당 아티스트의 다른 활성 공연 존재 여부로 `is_coming` 자동 재계산
- PENDING 공연에는 `DELETE /api/admin/concerts/{id}/candidates/{artistId}` 사용

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

---

## GET /api/admin/data/search/artists

**용도**: Data 파이프라인에서 아티스트를 이름으로 검색합니다. MBID 기반 수집 트리거 전 대상을 확인하는 용도입니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `name` | String | Y | 검색할 아티스트명 |

### 응답

`200 OK` — 배열

| 필드 | 타입 | 설명 |
|------|------|------|
| `mbid` | String | MusicBrainz ID |
| `name` | String | 아티스트명 |
| `country` | String? | 국가 코드 |
| `type` | String? | 아티스트 유형 (`Person` / `Group` 등) |
| `url` | String? | MusicBrainz 아티스트 페이지 URL |

---

## GET /api/admin/data/search/concerts

**용도**: Data 파이프라인에서 공연을 제목으로 검색합니다. KOPIS ID 기반 수집 트리거 전 대상을 확인하는 용도입니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `title` | String | Y | 검색할 공연명 |

### 응답

`200 OK` — 배열

| 필드 | 타입 | 설명 |
|------|------|------|
| `kopisId` | String | KOPIS 공연 ID |
| `title` | String | 공연명 |
| `startDate` | String | 시작일 (`YYYY-MM-DD`) |
| `endDate` | String | 종료일 (`YYYY-MM-DD`) |
| `venue` | String? | 공연장명 |
| `url` | String? | KOPIS 공연 상세 URL |

---

## POST /api/admin/data/collect/artists

**용도**: MBID를 지정해 Data 파이프라인의 아티스트 수집을 트리거합니다.

### 요청

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `mbid` | String | Y | MusicBrainz ID |

### 응답

`200 OK` (바디 없음)

---

## POST /api/admin/data/collect/concerts

**용도**: KOPIS ID를 지정해 Data 파이프라인의 공연 수집을 트리거합니다.

### 요청

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `kopisId` | String | Y | KOPIS 공연 ID |

### 응답

`200 OK` (바디 없음)

---

## POST /api/admin/data/collect/artists/{id}/releases

**용도**: 특정 아티스트의 릴리즈 수집을 Data 파이프라인에 트리거합니다.

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
| `ARTIST_NOT_FOUND` | 404 | Data 파이프라인 측에서 아티스트를 찾지 못한 경우 |

---

## POST /api/admin/data/collect/concerts/{id}/setlist

**용도**: 특정 공연의 셋리스트 수집을 Data 파이프라인에 트리거합니다.

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
| `CONCERT_NOT_FOUND` | 404 | Data 파이프라인 측에서 공연을 찾지 못한 경우 |

---

## POST /api/admin/policies

**용도**: 이용약관·개인정보처리방침의 새 버전을 등록합니다. 등록 트랜잭션 커밋 후 정책 변경 이메일 고지 배치가 자동으로 트리거됩니다.

### 요청

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `type` | String | Y | `TERMS` \| `PRIVACY` |
| `version` | String | Y | 정책 버전 (최대 50자) |
| `effectiveDate` | String | Y | 시행일 (`YYYY-MM-DD`) |
| `changeSummary` | String | Y | 변경 사항·사유 요약 (이메일 본문에 노출) |
| `detailUrl` | String | Y | 약관 원문 페이지 URL (최대 500자) |

### 응답

`201 Created`

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 정책 ID |
| `type` | String | `TERMS` \| `PRIVACY` |
| `version` | String | 정책 버전 |
| `effectiveDate` | String | 시행일 (`YYYY-MM-DD`) |
| `changeSummary` | String | 변경 사항·사유 요약 |
| `detailUrl` | String | 약관 원문 페이지 URL |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `POLICY_VERSION_DUPLICATE` | 409 | 동일 `type`·`version`이 이미 등록된 경우 |

### 비고

- 약관 원문 전체 텍스트는 저장하지 않는다. `detailUrl`로 FE가 SSOT로 관리하는 약관 페이지에 연결한다.
- 등록 완료(커밋) 후 알림 대상 생성 및 메일 발송 배치(`policyNotificationJob`)가 자동 실행된다. 발송 실패 대상은 1시간 간격으로 최대 3회까지 재시도한다.
- `POST /api/auth/register` 회원가입 시점에 시행 중인 최신 TERMS·PRIVACY 정책에 대한 동의 이력이 함께 기록된다. **시행 중인 정책이 하나도 등록되어 있지 않으면 회원가입이 `POLICY_NOT_FOUND`(404)로 실패**하므로, 배포 후 신규 가입을 받기 전 초기 정책 버전을 반드시 먼저 등록해야 한다.
- 기존 회원에 대한 정책 변경 고지는 [terms-of-service.md](../legal/terms-of-service.md) 제3조 3항의 묵시적 동의(고지 후 계속 이용 시 동의 간주) 원칙을 따른다. 로그인 시점 재동의 강제(하드 게이트)는 별도 기능으로 구현하지 않는다 — 민감정보 수집 목적 추가·제3자 제공 신설처럼 능동적 동의가 법적으로 필요한 개정이 실제로 생기기 전까지는 과설계로 판단.
