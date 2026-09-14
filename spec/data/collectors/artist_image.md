# 아티스트 프로필 이미지 수집 (collectors/artist_image.py, spotify_client.py)

## 인증 및 공통 클라이언트 (`spotify_client.py`)

Spotify Web API **Client Credentials Flow** 사용 (사용자 인증 불필요, 앱 단위 토큰).

### `_get_access_token() -> str`

```
POST https://accounts.spotify.com/api/token
Header: Authorization: Basic {base64(client_id:client_secret)}
Body:   grant_type=client_credentials
```

- `SPOTIFY_CLIENT_ID`/`SPOTIFY_CLIENT_SECRET` 환경변수 필수 — 없으면 호출 시점에 `ValueError`
- 모듈 레벨 `dict` 캐시(`_token_cache`)에 `access_token`과 만료 시각을 저장, **만료 60초 전**부터 미리 재발급 (경계 조건에서 만료된 토큰으로 요청하는 것을 방지)
- Timeout 10초

### `spotify_get(path: str, params: Optional[dict] = None) -> dict`

```
GET https://api.spotify.com/v1{path}
Header: Authorization: Bearer {token}
```

- 요청 전 `time.sleep(_REQUEST_INTERVAL=2.0)` — Spotify의 30초 롤링 윈도우 rate limit 대비, 초당 0.5req(30초당 15req)로 제한해 **공식 한도 대비 50% 안전 마진**을 둔다
- **`429` 수신 시 재시도하지 않고 즉시 `SpotifyRateLimitError` 발생** — 응답 헤더 `Retry-After`(초)를 `retry_after` 속성에 담아 상위(`scheduler.py`)로 전파. 호출부가 밴 상태를 기록하고 잡을 중단하도록 강제하는 설계 (상세: [error-handling.md](../error-handling.md))
- 그 외 4xx/5xx는 `response.raise_for_status()`로 일반 `requests.HTTPError` 전파(재시도는 호출부 책임)
- Timeout 10초

`SpotifyRateLimitError(requests.HTTPError)`: `retry_after: int` 속성을 가진 커스텀 예외. "호출부는 수집된 데이터를 저장하고 처리를 중단해야 한다"는 계약을 코드 주석으로 명시.

## artist_image.py 함수별 로직

### `_get_mb_artist_info(mbid: str) -> (artist_name, spotify_id)`

```
GET https://musicbrainz.org/ws/2/artist/{mbid}?fmt=json&inc=url-rels
Header: User-Agent: {MUSICBRAINZ_USER_AGENT 또는 "coming/1.0" 기본값}
```

- MusicBrainz rate limit 준수를 위해 `time.sleep(1.1)`
- `404`면 `(None, None)` 반환(예외 아님)
- `relations[]`에서 `open.spotify.com/artist/` 포함 URL을 찾아 마지막 path segment를 `spotify_id`로 추출, 첫 매치에서 중단

### `_search_spotify_artist(name: str) -> Optional[str]`

```
GET /v1/search?q={name}&type=artist&limit=1
```

이름 검색 fallback. 결과 없으면 `None`.

### `_get_artist_image_url(spotify_id: str) -> Optional[str]`

```
GET /v1/artists/{spotify_id}
```

`images[0]`(Spotify가 큰 순으로 정렬해 반환)의 `url`. `404`(아티스트 없음)는 `None` 반환, 그 외 HTTP 오류는 재전파.

### `spotify_artist_url(spotify_id: str) -> str`

`f"https://open.spotify.com/artist/{spotify_id}"` 문자열 조합만 수행(API 호출 없음). `run_artist_image_update`가 fallback으로 새로 찾은 Spotify 연결을 `artist_url`에 백필할 때 URL 생성에 사용.

### `collect_artist_image(mbid, spotify_url=None, name=None) -> (image_url, spotify_id)`

Spotify ID를 확보하는 3가지 경로를 **입력 우선순위대로** 시도해 불필요한 API 호출을 줄인다.

1. `spotify_url`이 있으면 URL에서 ID만 파싱(API 호출 0회)
2. 없고 `name`이 있으면 `_search_spotify_artist(name)` 호출(Spotify API 1회, MusicBrainz 호출 생략)
3. 둘 다 없으면 `_get_mb_artist_info(mbid)`로 MusicBrainz URL relations 조회 → 거기서 Spotify ID를 못 찾으면 MusicBrainz가 반환한 아티스트명으로 `_search_spotify_artist()` fallback

반환값 규약:
- `spotify_id`를 전혀 못 찾으면 `(None, None)`
- `spotify_id`는 찾았지만 이미지가 없으면 `(None, spotify_id)` — **이 경우에도 `spotify_id`는 반환** → 호출부(`scheduler.run_artist_image_update`)가 fallback으로 새로 발견한 Spotify 연결을 `artist_url` 테이블에 백필할 수 있도록 함
- 정상: `(image_url, spotify_id)`

## 예외 처리 요약

| 상황 | 처리 |
|------|------|
| Spotify 429 | `SpotifyRateLimitError` 즉시 전파 (재시도 없음) |
| MusicBrainz 404 | `(None, None)` 반환, 예외 아님 |
| Spotify 아티스트 404 | `_get_artist_image_url`이 `None` 반환 |
| Spotify ID 미발견(3경로 모두 실패) | `(None, None)` 반환, `scheduler.py`가 `image_failed` 체크포인트에 실패 기록 |
| access_token 발급 실패 | `response.raise_for_status()`로 예외 전파 — 이미지 수집 잡 전체가 중단됨(토큰 없이는 어떤 요청도 불가하므로 개별 항목 단위 흡수가 의미 없음) |
