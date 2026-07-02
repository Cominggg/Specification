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

**용도**: MBID로 Data 파이프라인에 아티스트 수집을 요청하고 결과를 반환합니다.

### 요청

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `mbid` | String | Y | MusicBrainz MBID (UUID) |

### 응답

`200 OK`

| 필드 | 타입 | 설명 |
|------|------|------|
| `success` | Boolean | 수집 성공 여부 |
| `artistId` | Long? | BE DB 아티스트 ID |
| `mbid` | String? | MusicBrainz MBID |
| `name` | String? | 아티스트명 |
| `imageUrl` | String? | Spotify 이미지 URL (수집 실패 시 null) |
| `aliases` | Array? | alias 목록 |
| `aliases[].name` | String | alias명 |
| `aliases[].locale` | String | 언어 코드 (`ko` \| `en` \| `ja`) |
| `reason` | String? | 실패 사유 (`success: false`일 때) |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `PIPELINE_NOT_FOUND` | 404 | MusicBrainz에 해당 MBID 없음 |
| `PIPELINE_CONFLICT` | 409 | 동일 리소스 수집이 이미 처리 중 |

### 비고

- Data 파이프라인이 동기적으로 처리하여 수집 결과를 즉시 반환
- 릴리즈(앨범) 수집은 포함되지 않음 — `POST /api/admin/data/collect/artists/{id}/releases` 별도 호출 필요

---

## POST /api/admin/data/collect/concerts

**용도**: KOPIS ID로 Data 파이프라인에 공연 수집을 요청하고 결과를 반환합니다.

### 요청

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `kopisId` | String | Y | KOPIS 공연 ID |

### 응답

`200 OK`

| 필드 | 타입 | 설명 |
|------|------|------|
| `success` | Boolean | 수집 성공 여부 |
| `concertId` | Long? | BE DB 공연 ID |
| `title` | String? | 공연명 |
| `matchedArtists` | Array? | 매칭된 아티스트 목록 (빈 배열이어도 success: true 가능) |
| `matchedArtists[].artistId` | Long | BE DB 아티스트 ID |
| `matchedArtists[].name` | String | 아티스트명 |
| `reason` | String? | 스킵 사유 (`success: false`일 때, 예: `not_touring`, `no_alias_match`) |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `PIPELINE_NOT_FOUND` | 404 | KOPIS에 해당 ID 없음 |
| `PIPELINE_CONFLICT` | 409 | 동일 리소스 수집이 이미 처리 중 |

### 비고

- Data 파이프라인이 동기적으로 처리하여 수집 결과를 즉시 반환
- 내한 공연이 아니거나 alias 매칭 실패 시 `success: false`, DB 저장 없음
- 수집 성공 시 공연은 `PENDING` 상태로 저장됨

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

### 비고

- Data 파이프라인 측에서 비동기 처리. 응답은 트리거 성공 여부만 나타냄.

---

## POST /api/admin/data/collect/concerts/{id}/setlist

**용도**: Data 파이프라인에 특정 공연의 셋리스트 수집을 요청하고 결과를 반환합니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 공연 ID (BE DB) |

### 응답

`200 OK`

| 필드 | 타입 | 설명 |
|------|------|------|
| `success` | Boolean | 수집 성공 여부 |
| `concertId` | Long? | BE DB 공연 ID |
| `setlistFmId` | String? | setlist.fm 셋리스트 ID |
| `attributionUrl` | String? | setlist.fm attribution URL |
| `tracks` | Array? | 트랙 목록 |
| `tracks[].position` | Integer | 트랙 순서 |
| `tracks[].songName` | String | 곡명 |
| `tracks[].info` | String? | 부가 정보 (예: 앙코르 등) |
| `reason` | String? | 스킵 사유 (`success: false`일 때, 예: `no_setlist_found`) |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `PIPELINE_NOT_FOUND` | 404 | 해당 공연 또는 셋리스트를 찾을 수 없음 |
| `PIPELINE_CONFLICT` | 409 | 동일 리소스 수집이 이미 처리 중 |

### 비고

- Data 파이프라인이 동기적으로 처리하여 수집 결과를 즉시 반환
- setlist.fm에 데이터 미존재 시 `success: false`, `reason: no_setlist_found`
