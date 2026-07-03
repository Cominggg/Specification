# 릴리즈(앨범·싱글) 수집 (collectors/release.py)

Spotify Web API 단독으로 앨범·싱글·트랙·커버아트를 수집한다. **MusicBrainz release-group API·Cover Art Archive는 사용하지 않는다** (2026-06-04 ERD 변경 — `release_group`/`track`에 `spotify_id` 추가, `mbid` NOT NULL 제약 제거로 Spotify 단독 수집 허용). 인증·공통 GET은 [artist_image.md](artist_image.md)와 공유하는 `collectors/spotify_client.py`를 사용한다.

## 외부 API

| 호출 | 엔드포인트 | 용도 |
|------|-----------|------|
| 앨범 목록 | `GET /v1/artists/{spotify_artist_id}/albums` | 아티스트의 앨범·싱글 ID 페이지네이션 조회 |
| 앨범 상세 | `GET /v1/albums/{album_id}` | 트랙·레이블·커버 이미지 포함 상세 |

### 앨범 목록 요청 (`_fetch_album_ids`)

```
GET /v1/artists/{spotify_artist_id}/albums
    ?include_groups=album,single&limit=10&market=JP&offset={n}
```

- `_PAGE_LIMIT=10` — 레이트리밋 예방을 위해 페이지당 10개로 제한 (Spotify 기본 최대 50보다 작게 설정)
- `compilation`, `appears_on`은 `include_groups`에서 제외 — 컴필레이션·피처링 앨범 미수집
- 응답: `{"total": int, "items": [{"id": str, ...}, ...]}`

### 앨범 상세 요청 (`_fetch_albums_individual`)

```
GET /v1/albums/{album_id}?market=JP
```

응답 필드 사용: `album_type`(`album`/`single`), `name`, `release_date`, `images[]`, `label`, `total_tracks`, `tracks.items[]`(트랙 목록), `tracks.next`(페이지네이션 존재 여부)

## 함수별 로직

### `_parse_release_date(raw: Optional[str]) -> Optional[str]`

`YYYY-MM-DD` 정규식(`^\d{4}-\d{2}-\d{2}$`)에 정확히 일치할 때만 유효로 인정. Spotify가 부분 날짜(`YYYY`, `YYYY-MM`)를 반환하는 경우 `None`으로 처리 — DB `date` 컬럼에 부정확한 값이 들어가지 않도록 함.

### `_parse_tracks(items: list) -> List[dict]`

`spotify_id`(트랙 `id`)가 없는 항목은 건너뜀. 각 트랙: `spotify_id`, `title`(`name`), `position`(`track_number`), `disc_number`, `length_ms`(`duration_ms`), `explicit`(기본 `False`).

### `_fetch_album_ids(spotify_artist_id, cached_total=None) -> (album_ids, spotify_total)`

- 페이지네이션 루프: `offset=0`부터 `items`가 빈 배열이거나 `offset >= spotify_total`이 될 때까지 반복
- **1콜 최적화**: 첫 페이지(`offset==0`) 응답의 `total`이 `cached_total`(DB `artist.spotify_album_total`)과 같으면 즉시 `([], spotify_total)` 반환 — 신보가 없다는 뜻이므로 나머지 페이지를 조회하지 않음
- `SpotifyRateLimitError`는 즉시 상위로 재전파 (`raise`)
- 그 외 `requests.RequestException`은 `logger.warning` 후 루프를 `break`(재시도 없음) — 지금까지 모은 `album_ids`는 유지한 채 반환

### `_fetch_albums_individual(album_ids: List[str]) -> List[dict]`

앨범 ID별로 순차 조회(병렬 아님). 개별 실패는 `logger.warning` 후 결과 리스트에서 제외하고 계속 진행. `SpotifyRateLimitError`는 즉시 상위로 재전파.

### `collect_releases(spotify_artist_id, skip_spotify_ids=None, cached_total=None) -> (releases, spotify_total)`

메인 함수.

1. `_fetch_album_ids()` 호출 — 결과가 비어있고 `spotify_total==0`이면 "앨범 없음"으로 로그, 아니면(1콜 최적화로 스킵된 경우) 조용히 반환
2. `skip_spotify_ids`(DB에 이미 저장된 `spotify_id` 집합)에 포함된 앨범은 상세 조회 대상에서 제외 — 중복 API 호출 방지
3. `_fetch_albums_individual()`로 상세 조회
4. `album_type`이 `_ALBUM_TYPE_MAP`(`album`→`Album`, `single`→`Single`)에 없는 타입(예: `compilation`)은 건너뜀 — 목록 조회 단계에서 `include_groups`로 걸렀지만 방어적으로 한 번 더 확인
5. 트랙이 50개를 초과해 `tracks.next`가 존재하면(Spotify 앨범 상세 API의 트랙 임베드는 최대 50개) `logger.warning`으로 "일부 누락 가능" 기록 — 추가 페이지 조회는 하지 않음(구현상 알려진 한계)
6. `cover_url`은 `images[0]`(가장 큰 이미지)의 `url`, 이미지가 없으면 `None`
7. 반환값 `spotify_total==0`은 "오류 또는 앨범 없음"을 의미하므로, 호출부는 `spotify_total > 0`일 때만 `artist.spotify_album_total`을 갱신한다 (`scheduler.py`)

## 저장 순서

호출부(`scheduler.py`)의 `_sort_releases()`가 `_RELEASE_TYPE_ORDER = {"Album": 0, "Single": 1}` 기준으로 정렬 후 `save_releases()`를 호출 — Album을 Single보다 먼저 저장한다.

## 예외 처리 요약

| 상황 | 처리 |
|------|------|
| Spotify 429 | `SpotifyRateLimitError` 즉시 전파 → `scheduler.py`가 밴 저장 + 재스케줄 (상세: [error-handling.md](../error-handling.md)) |
| 앨범 목록 조회 네트워크 오류 | 재시도 없이 `break`, 그때까지 모은 결과 반환 |
| 앨범 상세 조회 개별 실패 | 해당 앨범만 건너뜀, 나머지 계속 진행 |
| 트랙 50개 초과 | 경고 로그만 남기고 첫 50개만 저장 (페이지네이션 미구현) |
| 부분 발매일(`YYYY`, `YYYY-MM`) | `first_release_date = None`으로 저장 |
