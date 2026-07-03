# MusicBrainz 아티스트 수집 (collectors/musicbrainz.py)

## 외부 API

| 호출 | 메서드 | 용도 |
|------|--------|------|
| `GET /ws/2/artist/` | 검색 | J-POP 아티스트 페이지네이션 검색 |
| `GET /ws/2/artist/{mbid}` | 상세 | alias·URL relations 조회 |

- Base URL: `https://musicbrainz.org/ws/2`
- Header: `User-Agent: {MUSICBRAINZ_USER_AGENT}`(환경변수 필수, 없으면 모듈 임포트 시 `ValueError`), `Accept: application/json`
- Rate limit: `_RATE_LIMIT_SLEEP = 1.1`초 — 모든 GET 요청 전에 sleep (`_get()` 공통 함수)
- Timeout: 30초

### 검색 요청 (`_search_artists`)

```
GET /ws/2/artist/?query=tag:j-pop AND country:JP AND (type:Group OR type:Person)&fmt=json&limit=100&offset={n}
```

응답: `{"count": int, "artists": [{"id": mbid, "name": str, ...}, ...]}`

### 상세 요청 (`_fetch_artist_detail`)

```
GET /ws/2/artist/{mbid}?inc=aliases+url-rels&fmt=json
```

응답: `{"id", "name", "sort-name", "aliases": [...], "relations": [...]}`

## 함수별 로직

### `_get(path: str, params: dict) -> dict`

공통 GET 래퍼. `time.sleep(1.1)` 후 요청, `response.raise_for_status()`로 4xx/5xx는 예외 전파.

### `_parse_aliases(raw_aliases: list) -> list[dict]`

- `type == "Search hint"`인 alias는 제외 (검색 편의용 내부 alias, 실제 표기 아님)
- `locale`(예: `"ko"`, `"en-US"`)의 `-` 앞부분만 추출해 `ko`/`en`/`ja` 3개 언어만 채택, 그 외 로케일은 버림
- 반환: `[{"name": str, "locale": "ko"|"en"|"ja"}, ...]`

### `_parse_url_rels(relations: list) -> list[dict]`

- `target-type == "url"`인 relation만 대상
- `type == "official homepage"`는 `"Official"` 타입으로 저장 (도메인 매칭보다 우선)
- 그 외는 URL의 netloc(`www.` 접두 제거)을 `_DOMAIN_TO_TYPE`으로 매핑: `instagram.com`→Instagram, `twitter.com`/`x.com`→Twitter, `youtube.com`→YouTube, `open.spotify.com`→Spotify, `music.apple.com`→AppleMusic
- `youtu.be`(영상 단축 URL)는 매핑 테이블에서 의도적으로 제외 — 아티스트 채널 URL로 부적합
- 매핑된 타입마다 `_URL_PATTERNS` 정규식으로 2차 검증(예: Spotify는 `open\.spotify\.com/artist/[A-Za-z0-9]+` 패턴 일치 필수) — 앨범·트랙 URL이 아티스트 URL로 잘못 저장되는 것 방지
- 타입당 URL 1개만 유지(`seen: dict[type, url]`) — 동일 타입 중복 relation은 첫 번째만 채택

### `search_artists(name: str) -> list[dict]`

```
GET /ws/2/artist/?query=artist:{name}&fmt=json&limit=10
```

어드민 검색용(재시도 없음, 실패 시 `logger.error` 후 빈 리스트 반환). 반환 필드: `mbid`, `name`, `country`, `type`, `url`(MusicBrainz 아티스트 페이지).

### `collect_single_artist(mbid: str) -> Optional[dict]`

어드민이 MBID를 직접 지정해 등록할 때 사용. `_fetch_artist_detail()` 1회 호출 → `_parse_artist()`. **Last.fm 인기도 필터·Spotify URL 필수 조건을 적용하지 않는다** (어드민이 의도적으로 등록하는 케이스이므로). 실패 시(`requests.RequestException`) `None` 반환.

### `collect_artists(skip_mbids: Optional[set] = None) -> list[dict]`

초기/배치 수집 메인 함수.

1. `offset=0`부터 페이지네이션 (`_PAGE_LIMIT=100`), `_MAX_ARTISTS=10_000` 도달 시 강제 중단(`logger.warning`)
2. 검색 페이지 조회 실패 시 최대 3회 재시도(`5*attempt`초 대기), 3회 모두 실패하면 지금까지 수집분을 반환하고 중단(전체 예외 전파하지 않음)
3. `mbid in skip_mbids`이면 상세 조회를 건너뛴다 — 재개 시 이미 DB에 있는 아티스트의 API 호출 비용 절약
4. 상세 조회도 최대 3회 재시도, 3회 실패 시 해당 아티스트만 `None` 처리하고 다음으로 진행(전체 중단 아님)
5. **Spotify URL 필터**: `_parse_artist()` 결과에 `url_rels`에 `type=="Spotify"`가 없으면 해당 아티스트를 저장 대상에서 제외 — Spotify 기반 릴리즈·이미지 수집이 불가능한 아티스트는 초기 단계에서 걸러낸다
6. 페이지 끝(`offset >= total`) 또는 `_MAX_ARTISTS` 도달 시 종료

> 주의: 이 함수 자체는 Last.fm 필터를 호출하지 않는다. 문서 갱신 시점 기준 코드에는 Last.fm 인기도 필터 로직이 존재하지 않으므로(과거 명세에 기술되었던 `LASTFM_MIN_LISTENERS` 필터는 현재 코드에 없음), 별도 확인 없이 명세에 기재하지 않는다.
