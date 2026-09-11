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
| `isAuthor` | Boolean | 현재 사용자가 작성자 본인인지 여부. 비인증 시 `false` |
| `category` | String | `REVIEW` \| `INFO` \| `FREE` |
| `title` | String | 제목 |
| `content` | Object | Tiptap 에디터 JSON |
| `entityTags` | Object[] | 멘션 엔티티 목록 (아래 `EntityTag` 참고) |
| `recommendCount` | Long | 추천 수 |
| `viewCount` | Long | 조회 수 |
| `commentCount` | Long | 댓글 수 (답글 포함 전체 개수). 댓글 API(하단 참고)와 별도 카운터로 관리 |
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
  "isAuthor": false,
  "category": "REVIEW",
  "title": "요네즈 켄시 내한 첫 공연, 앙코르만 3번",
  "content": { "type": "doc", "content": [] },
  "entityTags": [
    { "entityType": "CONCERT", "entityId": 1, "title": "요네즈 켄시 KOREA LIVE 2026", "subtitle": "2026-11-14 · KSPO DOME", "thumbnailUrl": "https://..." }
  ],
  "recommendCount": 128,
  "viewCount": 3204,
  "commentCount": 4,
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
| `viewCount` | Long | 조회 수 |
| `createdAt` | String | 작성일시 |

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
      "viewCount": 3204,
      "createdAt": "2026-09-08T10:00:00"
    }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1
}
```

`PostSummary`는 이 목록 API·백링크 API(`GET /api/entities/{type}/{id}/posts`)·통합 검색 API(`GET /api/search`)에서 공통으로 사용되며 `viewCount`를 동일하게 포함합니다.

---

## PATCH /api/posts/{id} `[예정]`

**용도**: 게시글을 수정합니다. 작성자 본인만 가능합니다.

### 요청

**Path Parameters**: `id` (Long, Y)

**Request Body**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `category` | String | N | `REVIEW` \| `INFO` \| `FREE`. 생략·`null`이면 기존 값 유지 |
| `title` | String | N | 생략·`null`이면 기존 값 유지 |
| `content` | Object | N | 생략·`null`이면 기존 값 유지 |
| `entityTags` | Object[] | Y | 항상 전체 교체(기존 태그 전체 삭제 후 재삽입). 수정 후 최종 `category`(요청값 또는 기존값)가 `REVIEW`\|`INFO`면 1개 이상 필수 |

### 응답

`200 OK`, body 없음

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `POST_NOT_FOUND` | 404 | 존재하지 않는 게시글 |
| `FORBIDDEN` | 403 | 작성자 본인이 아님 |
| `INVALID_INPUT` | 400 | 수정 후 최종 `category`가 `REVIEW`\|`INFO`인데 `entityTags`가 비어 있음 |

### 비고

- `entityTags`는 부분 수정을 지원하지 않습니다. 수정 시 항상 전체 목록을 보내야 합니다.
- `category`를 변경해도 `entityTags` 검증은 우회되지 않습니다 — 검증은 항상 변경 후 최종 `category` 기준으로 수행됩니다.

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

`200 OK`

```json
{ "recommendCount": 129 }
```

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

`200 OK`

```json
{ "recommendCount": 128 }
```

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

---

## GET /api/posts/{id}/comments `[예정]`

**용도**: 게시글의 댓글·답글 목록을 조회합니다. 답글은 1단계로 제한되어(대댓글에는 답글 불가) 최상위 댓글 안에 중첩된 형태로 내려옵니다.

### 요청

**Path Parameters**: `id` (Long, Y) — 게시글 ID

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `page` | int | N | `0` | 최상위 댓글 기준 페이지 번호 |
| `size` | int | N | `20` | 페이지 크기 |

인증 불필요. 단, `isLiked`·`isAuthor`는 인증 요청일 때만 계산됩니다.

### 응답

[페이지네이션 응답](_index.md#페이지네이션-응답) 형태. `content` 항목(`Comment`, 최상위 댓글만 페이지네이션 대상 — `replies`는 매 항목에 전체 포함):

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | Long | 댓글 ID |
| `authorNickname` | String? | 작성자 닉네임 (탈퇴 회원이면 `null`) |
| `isAuthor` | Boolean | 현재 사용자가 작성자 본인인지 여부. 비인증 시 `false` |
| `content` | String | 댓글 내용 (plain text, 게시글 본문과 달리 리치 텍스트 아님) |
| `likeCount` | Long | 좋아요 수 |
| `isLiked` | Boolean? | 현재 사용자의 좋아요 여부. 비인증 시 `null` |
| `createdAt` | String | 작성일시 (ISO 8601) |
| `replies` | `Comment[]` | 이 댓글에 달린 답글 목록(작성일시 오름차순). 최상위 댓글에만 존재하며 배열은 페이지네이션되지 않음. 답글 항목의 `replies`는 항상 빈 배열 |

```json
{
  "content": [
    {
      "id": 101,
      "authorNickname": "민트초코최고",
      "isAuthor": false,
      "content": "오프닝으로 KICK BACK 나왔을 때 심장 떨어지는 줄... 후기 잘 봤습니다 ㅠㅠ",
      "likeCount": 8,
      "isLiked": false,
      "createdAt": "2026-09-08T13:20:00",
      "replies": [
        {
          "id": 108,
          "authorNickname": "요네즈러버",
          "isAuthor": true,
          "content": "저도 그 순간 진짜 소름이었어요! 다음에도 같이 가요 ㅎㅎ",
          "likeCount": 2,
          "isLiked": false,
          "createdAt": "2026-09-08T14:05:00",
          "replies": []
        }
      ]
    }
  ],
  "page": 0, "size": 20, "totalElements": 4, "totalPages": 1
}
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `POST_NOT_FOUND` | 404 | 존재하지 않는 게시글 |

### 비고

- `totalElements`·`totalPages`는 최상위 댓글 기준입니다. 답글까지 포함한 전체 개수는 `GET /api/posts/{id}` 응답의 `commentCount`를 사용합니다.

---

## POST /api/posts/{id}/comments `[예정]`

**용도**: 댓글 또는 답글을 작성합니다. 인증이 필요합니다.

### 요청

**Path Parameters**: `id` (Long, Y) — 게시글 ID

**Request Body**

| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `content` | String | Y | 댓글 내용. 공백만으로는 불가 |
| `parentCommentId` | Long | N | 답글로 작성할 때만 상위 댓글 ID 지정. 생략 시 최상위 댓글로 작성 |

```json
{ "content": "저도 그 순간 진짜 소름이었어요!", "parentCommentId": 101 }
```

### 응답

`201 Created`

```json
{ "id": 108 }
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INVALID_INPUT` | 400 | `content` 공백·누락 |
| `POST_NOT_FOUND` | 404 | 존재하지 않는 게시글 |
| `COMMENT_NOT_FOUND` | 404 | `parentCommentId`에 해당하는 댓글이 존재하지 않음 |
| `INVALID_REPLY_DEPTH` | 400 | `parentCommentId`가 이미 답글(최상위 댓글이 아님)인 경우 — 2단계 이상 중첩 금지 |

---

## DELETE /api/comments/{commentId} `[예정]`

**용도**: 댓글 또는 답글을 삭제합니다. 작성자 본인만 가능합니다.

### 요청

**Path Parameters**: `commentId` (Long, Y)

### 응답

`200 OK`, body 없음

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `COMMENT_NOT_FOUND` | 404 | 존재하지 않는 댓글 |
| `FORBIDDEN` | 403 | 작성자 본인이 아님 |

### 비고

- **BE 설계 시 확정 필요**: 답글이 달린 최상위 댓글을 삭제할 때 (a) 답글까지 함께 삭제(cascade)할지, (b) 게시글 삭제(`DELETE /api/posts/{id}`)처럼 하드 삭제하되 답글은 유지할지, (c) 내용만 "삭제된 댓글입니다"로 치환하는 소프트 삭제로 처리할지 협의가 필요합니다. FE는 세 방식 모두 대응 가능하도록 구현하되, 확정된 동작에 맞춰 `content`/`authorNickname` null 처리 여부를 조정합니다.

---

## POST /api/comments/{commentId}/like `[예정]`

**용도**: 댓글에 좋아요를 남깁니다. 인증이 필요합니다.

### 요청

**Path Parameters**: `commentId` (Long, Y)

### 응답

`200 OK`

```json
{ "likeCount": 9 }
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `COMMENT_NOT_FOUND` | 404 | 존재하지 않는 댓글 |
| `ALREADY_LIKED` | 409 | 이미 좋아요한 댓글 |

### 비고

- 사용자당 댓글당 1회로 제한됩니다(`UNIQUE(user_id, comment_id)`). 게시글 추천(`POST /api/posts/{id}/recommend`)과 동일한 패턴입니다.

---

## DELETE /api/comments/{commentId}/like `[예정]`

**용도**: 댓글 좋아요를 취소합니다. 인증이 필요합니다.

### 요청

**Path Parameters**: `commentId` (Long, Y)

### 응답

`200 OK`

```json
{ "likeCount": 8 }
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `COMMENT_NOT_FOUND` | 404 | 존재하지 않는 댓글 |
| `NOT_LIKED` | 400 | 좋아요하지 않은 댓글 |
