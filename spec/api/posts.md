# 게시판 API

> ⚠️ **설계 확정 단계 — 아직 구현 전입니다.** (이슈 #116, 브랜치 `feat/#116-post-board`) 배포 완료 후 이 안내와 `[예정]` 표기는 제거됩니다.

---

## GET /api/mentions/search `[예정]`

**용도**: 게시글 작성 시 본문 내 공연·아티스트·음악(발매) 인라인 멘션(`/공연 요네즈` 등) 자동완성을 위한 엔티티 검색입니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `type` | String | Y | — | `CONCERT` \| `ARTIST` \| `RELEASE` |
| `q` | String | Y | — | 검색어. 공백·누락 시 400 |
| `limit` | int | N | `10` | 최대 `20` |

### 응답

페이지네이션 없이 배열을 그대로 반환합니다.

| 필드 | 타입 | 설명 |
|------|------|------|
| `type` | String | `CONCERT` \| `ARTIST` \| `RELEASE` |
| `id` | Long | 엔티티 ID |
| `title` | String | 표시명 (공연명 / 아티스트명 / 발매명) |
| `subtitle` | String? | 부가 정보 (아래 비고 참고) |
| `thumbnailUrl` | String? | 썸네일 이미지 URL |

```json
[
  { "type": "CONCERT", "id": 1, "title": "요네즈 켄시 KOREA LIVE 2026", "subtitle": "2026-11-14 · KSPO DOME", "thumbnailUrl": "https://..." },
  { "type": "ARTIST", "id": 2, "title": "요네즈 켄시", "subtitle": null, "thumbnailUrl": "https://..." },
  { "type": "RELEASE", "id": 3, "title": "LOST CORNER", "subtitle": "요네즈 켄시", "thumbnailUrl": "https://..." }
]
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INVALID_INPUT` | 400 | `q` 공백·누락 |

### 비고

- `RELEASE`는 `release_group`(앨범·싱글·EP 단위) 기준입니다. 트랙 단위 멘션은 스코프 아웃되어 지원하지 않습니다.
- `CONCERT`: `subtitle` = `"{startDate} · {venue}"`, `thumbnailUrl` = `posterUrl`
- `ARTIST`: `subtitle` = `null`, `thumbnailUrl` = `artist.imageUrl`
- `RELEASE`: `subtitle` = 아티스트명(`artist_id` join), `thumbnailUrl` = `coverUrl`
- 매치 0건이면 200과 함께 빈 배열을 반환합니다.

---

## POST /api/posts `[예정]`

**용도**: 게시글을 작성합니다. 인증이 필요합니다.

### 요청

**Request Body**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `category` | String | Y | `REVIEW`(후기) \| `INFO`(정보·제보) \| `FREE`(자유) |
| `title` | String | Y | 제목 |
| `content` | Object | Y | Tiptap 에디터 JSON (본문) |
| `entityTags` | Object[] | N | 본문에서 멘션한 엔티티 목록. `category`가 `REVIEW`\|`INFO`면 1개 이상 필수 |
| `entityTags[].entityType` | String | Y | `CONCERT` \| `ARTIST` \| `RELEASE` |
| `entityTags[].entityId` | Long | Y | 엔티티 ID |

```json
{
  "category": "REVIEW",
  "title": "요네즈 켄시 내한 첫 공연, 앙코르만 3번",
  "content": { "type": "doc", "content": [] },
  "entityTags": [ { "entityType": "CONCERT", "entityId": 1 } ]
}
```

### 응답

`201 Created`

```json
{ "id": 10 }
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INVALID_INPUT` | 400 | `category`가 `REVIEW`\|`INFO`인데 `entityTags`가 비어 있음 |

### 비고

- BE가 `content`에서 텍스트 노드만 추출해 검색 전용 `content_text` 컬럼을 서버 측에서 생성합니다. FE가 별도로 평문을 보낼 필요는 없습니다.

---

## GET /api/posts/{id} `[예정]`

**용도**: 게시글 상세를 조회합니다. 조회될 때마다 `viewCount`가 1 증가합니다(중복 조회 방지 없음).

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | Long | Y | 게시글 ID |

인증 불필요. 단, `isRecommended`는 인증 요청일 때만 계산됩니다.

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 게시글 ID |
| `authorNickname` | String? | 작성자 닉네임 (탈퇴 회원이면 `null`) |
| `category` | String | `REVIEW` \| `INFO` \| `FREE` |
| `title` | String | 제목 |
| `content` | Object | Tiptap 에디터 JSON |
| `entityTags` | Object[] | 멘션 엔티티 목록 (아래 `EntityTag` 참고) |
| `recommendCount` | Long | 추천 수 |
| `viewCount` | Long | 조회 수 |
| `isRecommended` | Boolean? | 현재 사용자의 추천 여부. 비인증 시 `null` |
| `createdAt` | String | 작성일시 (ISO 8601) |
| `updatedAt` | String | 수정일시 (ISO 8601) |

`EntityTag` 객체:

| 필드 | 타입 | 설명 |
|------|------|------|
| `entityType` | String | `CONCERT` \| `ARTIST` \| `RELEASE` |
| `entityId` | Long | 엔티티 ID |
| `title` | String | 엔티티 표시명 |
| `subtitle` | String? | `GET /api/mentions/search` 응답과 동일 규칙 |
| `thumbnailUrl` | String? | 썸네일 URL |

```json
{
  "id": 10,
  "authorNickname": "moonlit_haze",
  "category": "REVIEW",
  "title": "요네즈 켄시 내한 첫 공연, 앙코르만 3번",
  "content": { "type": "doc", "content": [] },
  "entityTags": [
    { "entityType": "CONCERT", "entityId": 1, "title": "요네즈 켄시 KOREA LIVE 2026", "subtitle": "2026-11-14 · KSPO DOME", "thumbnailUrl": "https://..." }
  ],
  "recommendCount": 128,
  "viewCount": 3204,
  "isRecommended": false,
  "createdAt": "2026-09-08T10:00:00",
  "updatedAt": "2026-09-08T10:00:00"
}
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `POST_NOT_FOUND` | 404 | 존재하지 않는 게시글 |

---

## GET /api/posts `[예정]`

**용도**: 게시글 목록을 카테고리별로 조회합니다. 정렬은 최신순 고정입니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `category` | String | N | — | `REVIEW` \| `INFO` \| `FREE`. 생략 시 전체 |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목(`PostSummary`):

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 게시글 ID |
| `authorNickname` | String? | 작성자 닉네임 |
| `category` | String | `REVIEW` \| `INFO` \| `FREE` |
| `title` | String | 제목 |
| `entityTags` | Object[] | 멘션 엔티티 목록 (`EntityTag`, 위 상세 API 참고) |
| `recommendCount` | Long | 추천 수 |
| `createdAt` | String | 작성일시 |

`viewCount`는 목록 응답에 포함되지 않습니다(상세 전용).

```json
{
  "content": [
    {
      "id": 10,
      "authorNickname": "moonlit_haze",
      "category": "REVIEW",
      "title": "요네즈 켄시 내한 첫 공연, 앙코르만 3번",
      "entityTags": [ { "entityType": "CONCERT", "entityId": 1, "title": "요네즈 켄시 KOREA LIVE 2026", "subtitle": null, "thumbnailUrl": null } ],
      "recommendCount": 128,
      "createdAt": "2026-09-08T10:00:00"
    }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1
}
```

---

## PATCH /api/posts/{id} `[예정]`

**용도**: 게시글을 수정합니다. 작성자 본인만 가능합니다.

### 요청

**Path Parameters**: `id` (Long, Y)

**Request Body**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `title` | String | N | 생략·`null`이면 기존 값 유지 |
| `content` | Object | N | 생략·`null`이면 기존 값 유지 |
| `entityTags` | Object[] | Y | 항상 전체 교체(기존 태그 전체 삭제 후 재삽입). `category`가 `REVIEW`\|`INFO`면 1개 이상 필수 |

### 응답

`200 OK`, body 없음

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `POST_NOT_FOUND` | 404 | 존재하지 않는 게시글 |
| `FORBIDDEN` | 403 | 작성자 본인이 아님 |
| `INVALID_INPUT` | 400 | `category`가 `REVIEW`\|`INFO`인데 `entityTags`가 비어 있음 |

### 비고

- `entityTags`는 부분 수정을 지원하지 않습니다. 수정 시 항상 전체 목록을 보내야 합니다.

---

## DELETE /api/posts/{id} `[예정]`

**용도**: 게시글을 삭제합니다. 작성자 본인만 가능합니다.

### 요청

**Path Parameters**: `id` (Long, Y)

### 응답

`200 OK`, body 없음

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `POST_NOT_FOUND` | 404 | 존재하지 않는 게시글 |
| `FORBIDDEN` | 403 | 작성자 본인이 아님 |

### 비고

- 연관된 `entityTags`·추천 레코드도 함께 삭제됩니다.

---

## POST /api/posts/{id}/recommend `[예정]`

**용도**: 게시글을 추천합니다. 인증이 필요합니다.

### 요청

**Path Parameters**: `id` (Long, Y)

### 응답

`200 OK`, body 없음

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `POST_NOT_FOUND` | 404 | 존재하지 않는 게시글 |
| `ALREADY_RECOMMENDED` | 409 | 이미 추천한 게시글 |

### 비고

- 사용자당 게시글당 1회로 제한됩니다(`UNIQUE(user_id, post_id)`).
- 엔티티(공연·아티스트) 팔로우/좋아요와는 별개의 카운터입니다.

---

## DELETE /api/posts/{id}/recommend `[예정]`

**용도**: 게시글 추천을 취소합니다. 인증이 필요합니다.

### 요청

**Path Parameters**: `id` (Long, Y)

### 응답

`200 OK`, body 없음

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `POST_NOT_FOUND` | 404 | 존재하지 않는 게시글 |
| `NOT_RECOMMENDED` | 400 | 추천하지 않은 게시글 |

---

## GET /api/entities/{type}/{id}/posts `[예정]`

**용도**: 특정 엔티티(공연·아티스트·발매)를 멘션한 게시글 목록(백링크)을 조회합니다. 공연·아티스트·발매 상세 페이지의 "관련 게시글" 탭에서 사용됩니다.

### 요청

**Path Parameters**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `type` | String | Y | `CONCERT` \| `ARTIST` \| `RELEASE` |
| `id` | Long | Y | 엔티티 ID |

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `sort` | String | N | `latest` | `recommend` \| `latest` |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content`는 `GET /api/posts`와 동일한 `PostSummary`.

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INVALID_INPUT` | 400 | `sort`가 `recommend`\|`latest` 외의 값 |
| `CONCERT_NOT_FOUND` \| `ARTIST_NOT_FOUND` \| `RELEASE_NOT_FOUND` | 404 | `type`에 해당하는 엔티티가 존재하지 않음 |

---

## GET /api/search `[예정]`

**용도**: 게시글 제목·본문·멘션된 엔티티명을 통합 검색합니다. 정렬은 최신순 고정입니다.

### 요청

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `q` | String | Y | — | 검색어. 공백·누락 시 400 |
| `page` | int | N | `0` | 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content`는 `GET /api/posts`와 동일한 `PostSummary`.

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INVALID_INPUT` | 400 | `q` 공백·누락 |

### 비고

- 제목·`content_text` LIKE 매치 또는 `entityTags`를 통한 엔티티명(공연명·아티스트명·발매명) 매치를 단일 DB 쿼리로 처리합니다. 텍스트 매치와 멘션 매치를 애플리케이션 레벨에서 별도로 병합하지 않습니다.
