# BE↔Data 파이프라인 연동 API (ROLE_ADMIN)

관리자가 Data 파이프라인과 직접 연동하는 엔드포인트입니다. 모든 엔드포인트는 `ROLE_ADMIN` 권한이 필요합니다.

BE는 `DataPipelineClient`(WebClient)를 통해 Data 파이프라인 내부 API를 호출하며, `X-Internal-Secret` 헤더로 인증합니다.

---

## GET /api/admin/data/search/artists

**용도**: Data 파이프라인을 통해 MusicBrainz에서 아티스트명으로 후보를 검색합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `name` | String | Y | 검색할 아티스트명 |

### 응답

`200 OK`

| 필드 | 타입 | 설명 |
|------|------|------|
| `mbid` | String | MusicBrainz MBID (UUID) |
| `name` | String | 아티스트명 |
| `country` | String? | 국가 코드 |
| `type` | String? | `Person` \| `Group` 등 |
| `url` | String? | MusicBrainz 아티스트 페이지 URL |

### 비고

- 최대 10건 반환
- Data 파이프라인이 MusicBrainz API를 동기적으로 호출하여 결과 반환

---

## GET /api/admin/data/search/concerts

**용도**: Data 파이프라인을 통해 KOPIS에서 공연명으로 후보를 검색합니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `title` | String | Y | 검색할 공연명 |

### 응답

`200 OK`

| 필드 | 타입 | 설명 |
|------|------|------|
| `kopisId` | String | KOPIS 공연 ID |
| `title` | String | 공연명 |
| `startDate` | String | 시작일 (`YYYY-MM-DD`) |
| `endDate` | String | 종료일 (`YYYY-MM-DD`) |
| `venue` | String? | 공연장명 |
| `url` | String? | KOPIS 공연 상세 URL |

### 비고

- 오늘~2년 후 범위, 최대 20건 반환
- Data 파이프라인이 KOPIS API를 동기적으로 호출하여 결과 반환

---

## POST /api/admin/data/collect/artists

**용도**: MBID로 Data 파이프라인에 아티스트 초기 수집을 트리거합니다. 아티스트 정보·릴리즈·이미지를 일괄 수집합니다.

### 요청

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `mbid` | String | Y | MusicBrainz MBID (UUID) |

### 응답

`200 OK` (바디 없음)

### 비고

- Data 파이프라인 측에서 비동기 처리. 응답은 트리거 성공 여부만 나타냄.

---

## POST /api/admin/data/collect/concerts

**용도**: KOPIS ID로 Data 파이프라인에 공연 단건 수집을 트리거합니다.

### 요청

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `kopisId` | String | Y | KOPIS 공연 ID |

### 응답

`200 OK` (바디 없음)

### 비고

- Data 파이프라인 측에서 비동기 처리. 수집 후 alias 매칭 실행, 내한 공연이거나 매칭이 있을 경우 DB에 PENDING 상태로 저장.

---

## POST /api/admin/data/collect/artists/{id}/releases

**용도**: Data 파이프라인에 특정 아티스트의 MusicBrainz 릴리즈 수집을 트리거합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 아티스트 ID (BE DB) |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `ARTIST_NOT_FOUND` | 404 | Data 파이프라인 측에서 해당 아티스트를 찾을 수 없음 |

---

## POST /api/admin/data/collect/concerts/{id}/setlist

**용도**: Data 파이프라인에 특정 공연의 셋리스트 수집을 트리거합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID (BE DB) |

### 응답

`200 OK` (바디 없음)

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `CONCERT_NOT_FOUND` | 404 | Data 파이프라인 측에서 해당 공연을 찾을 수 없음 |
