# 음악 발매 API

---

## GET /api/releases

**용도**: 음악 전체 목록을 조회합니다. (REL-03)

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `artistId` | Long | N | — | 특정 아티스트 필터 |
| `type` | String | N | — | `Album` \| `Single` \| `기타` |
| `following` | Boolean | N | — | `true`이면 팔로우 아티스트 릴리즈만 반환. 미인증이면 빈 페이지 반환 |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 릴리즈 ID |
| `coverUrl` | String? | 커버 이미지 URL |
| `artistName` | String | 아티스트명 |
| `title` | String | 타이틀 |
| `type` | String | `Album` \| `Single` 또는 기타 |
| `releaseDate` | String? | 발매일 |

```json
{
  "content": [
    {
      "id": 1,
      "coverUrl": null,
      "artistName": "Kenshi Yonezu",
      "title": "LOST CORNER",
      "type": "Album",
      "releaseDate": "2024-08-28"
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 12,
  "totalPages": 1
}
```

### 비고

- 정렬: `releaseDate` DESC NULLS LAST 고정 (발매일 없는 항목은 항상 마지막)
- `기타`: `Album`·`Single` 외 타입 (Live·Compilation·Remix·Soundtrack·Other 등) 전체
- `following=true`이면 `artistId` 필터 무시; `type` 필터는 동시 적용 가능
- 홈 "새 앨범·싱글" 섹션: `size=8`로 재사용

---

## GET /api/releases/search

**용도**: 릴리즈명·트랙명·아티스트명(alias 포함)으로 음악을 검색합니다. (REL-04)

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `q` | String | N | — | 검색어 (릴리즈명·트랙명·아티스트명·alias 부분 일치, 대소문자 무시). 생략 시 텍스트 필터 없이 다른 필터만 적용 |
| `type` | String | N | — | `Album` \| `Single`. 그 외 값 → 400. 생략 시 전체 타입 |
| `following` | Boolean | N | `false` | `true`이면 팔로우 아티스트 릴리즈만 검색. 미인증이면 빈 페이지 반환 |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |

**인증**: 불필요. 단, `following=true` 사용 시 `Authorization: Bearer {token}` 헤더 필요 (없으면 빈 페이지 반환).

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목은 `GET /api/releases`와 동일:

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 릴리즈 ID |
| `coverUrl` | String? | 커버 이미지 URL |
| `artistName` | String | 아티스트명 |
| `title` | String | 타이틀 |
| `type` | String | `Album` \| `Single` 또는 기타 |
| `releaseDate` | String? | 발매일 |

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INVALID_INPUT` | 400 | `type`에 `Album`·`Single` 외 값 전달 시 |

### 비고

- 검색 대상: 릴리즈명, 수록 트랙명, 아티스트명, 아티스트 alias (대소문자 무시 부분 일치)
- `q` 생략 시 `type`·`following` 필터만 적용 (텍스트 매칭 없음)
- 정렬: `releaseDate` DESC NULLS LAST 고정

---

## GET /api/releases/{id}

**용도**: 릴리즈 상세 정보를 트랙리스트·커버 포함해 조회합니다. (REL-02)

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 릴리즈 ID |

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 릴리즈 ID |
| `title` | String | 타이틀 |
| `type` | String | 릴리즈 타입 |
| `releaseDate` | String? | 발매일 |
| `coverUrl` | String? | 커버 이미지 URL |
| `label` | String? | 레이블 |
| `totalTracks` | int? | 전체 트랙 수 |
| `artistId` | Long | 아티스트 ID |
| `artistName` | String | 아티스트명 |
| `tracks` | Object[] | 수록곡 목록 |
| `tracks[].position` | int | 트랙 순서 |
| `tracks[].title` | String | 트랙 제목 |
| `tracks[].lengthMs` | int? | 재생 시간 (ms) |
| `tracks[].discNumber` | int? | 디스크 번호 (멀티 디스크 앨범) |
| `tracks[].explicit` | Boolean? | 명시적 콘텐츠 여부 |

```json
{
  "id": 1,
  "title": "LOST CORNER",
  "type": "Album",
  "releaseDate": "2024-08-28",
  "coverUrl": null,
  "label": "Sony Music",
  "totalTracks": 13,
  "artistId": 2,
  "artistName": "Kenshi Yonezu",
  "tracks": [
    { "position": 1, "title": "LOST CORNER", "lengthMs": 262000, "discNumber": 1, "explicit": false },
    { "position": 2, "title": "LADY", "lengthMs": 238000, "discNumber": 1, "explicit": false }
  ]
}
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `RELEASE_NOT_FOUND` | 404 | 존재하지 않는 릴리즈 |
