# 데이터 수집 파이프라인 명세

Python 별도 레포지토리로 구현. Spring 백엔드와 동일한 DB를 공유하며, 스키마 변경은 Spring 쪽 마이그레이션 도구로 단일 관리.

## 레포지토리 구조

```
coming-data/
├── collectors/
│   ├── kopis.py           # KOPIS API 수집
│   ├── musicbrainz.py     # MusicBrainz 아티스트 수집
│   ├── release.py         # MusicBrainz 릴리즈(앨범·싱글·EP) + 트랙·커버 수집
│   ├── setlist.py         # setlist.fm 셋리스트 수집
│   └── wikipedia.py       # Korean Wikipedia redirect 기반 한국어 alias 수집
├── matchers/
│   └── artist_matcher.py  # alias 기반 매칭 로직 (rapidfuzz)
├── db/
│   └── repository.py      # DB 저장 (SQLAlchemy)
├── tests/                 # pytest 단위 테스트
├── scheduler.py           # APScheduler 진입점
└── pyproject.toml
```

## 수집 파이프라인 단계

### ① 초기 구축 (1회성 · `python scheduler.py init`)

| 작업 | 상세 |
|------|------|
| MusicBrainz 아티스트 수집 | JP 아티스트 목록 수집. `tag:j-pop AND country:JP AND (type:Group OR type:Person)` 조건. 이름·alias(한/영/일)·url-rels 저장.<br>개별 상세: `GET /ws/2/artist/{mbid}?inc=aliases+url-rels&fmt=json`<br>url-rels: Instagram·Twitter·YouTube·Spotify·Apple Music 등 허용 도메인만 저장.<br>**Last.fm 인기도 필터**: 상세 조회 후 `GET https://ws.audioscrobbler.com/2.0/?method=artist.getInfo&mbid={mbid}` 로 월간 리스너 수 확인. `LASTFM_MIN_LISTENERS` 임계값(기본 1,000) 미달 시 저장 건너뜀. 조회 실패 시 저장 진행.<br>재개 지원: 이미 DB에 저장된 MBID는 상세 조회 건너뜀.<br>※ Rate Limit: 1 req/sec (MusicBrainz), Last.fm은 MusicBrainz sleep으로 자연 제어 |
| Wikipedia 한국어 alias 수집 | Korean Wikipedia redirect 기반으로 아티스트 한국어 alias를 보완 수집.<br>`GET https://ko.wikipedia.org/w/api.php?action=query&prop=redirects&titles={name}&rdlimit=500`<br>리다이렉트 중 한글 포함 제목만 `artist_alias`에 저장. 실패 시 지수 백오프(최대 3회) 재시도.<br>※ Rate Limit: 0.5초 간격 |
| MusicBrainz 릴리즈 수집 | HIGH confidence 매칭 아티스트의 앨범·싱글·EP 우선 수집. 나머지는 주간 배치가 점진적으로 채운다.<br>`GET /ws/2/release-group/?artist={mbid}&type=album%7Csingle%7Cep&limit=100&offset={n}&fmt=json`<br>취득 필드: `title`(제목), `first-release-date`(발매일), `primary-type`(Album·Single·EP).<br>**저장 순서**: Album → EP → Single 순으로 정렬 후 저장. `track.mbid UNIQUE` 제약 하에 앨범 수록곡이 앨범 release_group에 우선 연결됨.<br>▸ **트랙 수집**: release-group별 대표 release MBID 취득 후 개별 상세 호출.<br>`GET /ws/2/release/{release-mbid}?inc=recordings+labels&fmt=json`<br>수록곡(`media[].tracks`): 트랙 번호·제목·`length` 저장. 레이블: `label-info[].label.name` 저장.<br>▸ **앨범 커버 URL**: Cover Art Archive 별도 주간 잡(`run_cover_art_update`)에서 미수집 건만 보완.<br>※ Rate Limit: 1 req/sec (MusicBrainz), Cover Art Archive 별도 1 req/sec |
| KOPIS 수집·매칭 | 초기 수집 중 KOPIS 공연 조회 + alias 매칭 실행 → 내한 확정 아티스트 파악. `--skip-kopis` 플래그로 건너뜀 가능. |
| 관리자 등록 | MusicBrainz 미등록 아티스트를 관리자 UI로 직접 입력. |

플래그:
- `--skip-artists`: 아티스트 수집 건너뜀
- `--skip-kopis`: KOPIS 수집·매칭 건너뜀
- `--skip-wikipedia`: Wikipedia alias 수집 건너뜀
- `--force-artists`: 기존 DB 아티스트를 건너뛰지 않고 전체 재수집

### ② 주기적 수집 (스케줄)

| 작업 | 스케줄 | 상세 |
|------|--------|------|
| KOPIS 수집·매칭 | 월요일 03시 | 내한 공연 후보 조회. `visit=Y, genrenm=GGGA(대중음악)` 조건. `prfnm`·`prfcast`·날짜·장소·`poster_url`·`venue_address`·`price`·`updatedate` 저장.<br>`GET /openApi/restful/pblprfr?service={key}&stdate=20200101&eddate={today}&shcate=GGGA&visit=Y&rows=100&cpage={n}`<br>상세: `GET /openApi/restful/pblprfr/{mt20id}?service={key}`<br>상세 API 응답의 `relates` 필드(예매처 링크 목록) 저장. 값이 없는 경우 빈 배열로 처리.<br>저장 전 `has_match()`로 alias 매칭 공연만 필터링 — 비매칭 공연은 DB에 저장하지 않음.<br>매칭 공연은 `status=PENDING`으로 저장하고, 매칭 결과를 `concert_artist_candidate`에 INSERT (어드민 검토 후 `concert_artist`로 확정).<br>※ 주 1회 이상 권장 |
| 공연 상태 갱신 | 매일 04시 | KOPIS 재조회 후 `updatedate` 변화 감지 시 `status`·`kopis_update_date` 갱신. `artist.is_coming` 동기화. |
| 릴리즈 갱신 | 화요일 05시 | 등록된 전체 아티스트의 신보 감지. `first-release-date` 기준 DB에 없는 항목만 INSERT. 신규 릴리즈 감지 시 트랙·레이블도 연속 수집. |
| 커버아트 수집 | 수요일 05시 | `cover_url`이 없는 release_group만 대상으로 Cover Art Archive 호출. 404 시 null 유지. |
| Wikipedia alias 수집 | 목요일 03시 | 전체 아티스트 대상 Korean Wikipedia redirect 기반 한국어 alias 보완. |
| setlist 수집 | 매일 06시 | `status=공연완료` 대상. 공연 완료 후 1일 이내 실행. |

#### 매칭 상세

| 단계 | 대상 | 기준 | 결과 |
|------|------|------|------|
| 매칭 ① | `prfcast` 각 이름 | Artist alias **완전 일치** | `concert_artist_candidate` INSERT, concert = PENDING |
| 매칭 ② | `prfcast` 각 이름 | `rapidfuzz.fuzz.token_set_ratio` ≥ 85 | `concert_artist_candidate` INSERT, concert = PENDING |
| 매칭 ③ | `prfnm` (제목) | 구문 일치(`_phrase_match_title`): 단어 경계 exact 또는 다중 단어 구문 포함 | `concert_artist_candidate` INSERT, concert = PENDING (매칭 ①② 실패 시 폴백) |

- 매칭 ③은 `prfcast` 기반 매칭 ①②가 모두 실패한 경우에만 폴백으로 실행된다.
- 매칭된 모든 공연은 `PENDING` 상태로 저장되어 어드민 검토 큐에 진입한다.
- `prfcast` 구분자: `,` `·` `&` `×` `・` `/` — feat/featuring/ft 표기 자동 제거.
- `prfcast`에 여러 아티스트 포함 시 각각 개별 매칭 후 모두 `concert_artist_candidate`에 INSERT.

### 공연-아티스트 관계 구조

공연과 아티스트는 **다대다(M:N)** 관계. 합동 공연(페스티벌·조인트 콘서트) 지원을 위해 `concert_artist` 중간 테이블로 관리.

파이프라인 매칭 결과는 먼저 `concert_artist_candidate`에 임시 저장되고, 어드민 승인 후 `concert_artist`로 이동한다.

**concert_artist** (어드민 승인 후 확정된 관계):

| 컬럼 | 설명 |
|------|------|
| `concert_id` | 공연 FK |
| `artist_id` | 아티스트 FK |

**concert_artist_candidate** (파이프라인 매칭 후 어드민 검토 대기):

| 컬럼 | 설명 |
|------|------|
| `concert_id` | 공연 FK |
| `artist_id` | 아티스트 FK |
| `matched_by` | `prfcast` / `prfnm` / `manual` |

> 단독 공연은 `concert_artist_candidate` 행이 1개, 합동 공연은 참여 아티스트 수만큼 행이 생성됨.

### ③ 셋리스트 수집 (스케줄 · 매일 06시)

| 작업 | 상세 |
|------|------|
| setlist.fm 수집 | `status=공연완료` 대상. 공연 완료 후 1일 이내 실행.<br>`GET https://api.setlist.fm/rest/1.0/search/setlists?artistMbid={mbid}&countryCode=KR&p={n}`<br>Header: `x-api-key`, `Accept: application/json`<br>데이터 미존재 시 빈 상태 유지. |

### ④ is_coming 동기화

`artist.is_coming` 필드는 오늘 이후 `UPCOMING` 또는 `ONGOING` 상태 공연이 `concert_artist`에 존재하는지 여부로 자동 갱신된다.

- 갱신 시점: 공연 상태 갱신 완료 후, 어드민이 PENDING 공연을 승인할 때 (BE)
- 실제로 변경된 행만 UPDATE (불필요한 쓰기 I/O 최소화)

### ⑤ 관리자 검색·단건 수집 (API 트리거)

관리자 UI 또는 백엔드에서 직접 호출하는 검색 및 단건 수집 함수.

**검색 함수** (동기 응답):

| 함수 | 설명 |
|------|------|
| `search_artists(name)` | MusicBrainz 아티스트명 검색. 최대 10건 반환. |
| `search_concerts(title)` | KOPIS 공연명 검색. 오늘~2년 후 범위, 최대 20건 반환. |

**수집 함수** (동기 처리 — 결과 반환):

| 함수 | 반환 | 설명 |
|------|------|------|
| `collect_artist_initial(mbid)` | `CollectArtistResult` | MBID 기반 아티스트 수집 (정보·이미지). 릴리즈는 미포함. |
| `collect_and_save_concert(kopis_id)` | `CollectConcertResult` | 단건 KOPIS 공연 수집 → alias 매칭 → DB 저장. 내한 공연 아니거나 매칭 없으면 `success: false`. |
| `collect_and_save_setlist(concert_id)` | `CollectSetlistResult` | 단건 공연 셋리스트 수집 → DB 저장. 데이터 미존재 시 `success: false`. |

**수집 트리거 함수** (비동기 처리 — 결과 미반환):

| 함수 | 설명 |
|------|------|
| `collect_and_save_release_group(release_group_mbid, artist_mbid)` | 단건 릴리즈 그룹 수집 (트랙·레이블 포함) → DB 저장. |
| `collect_and_save_cover_art(release_group_mbid)` | 단건 릴리즈 그룹 커버아트 수집 → DB 갱신. |

BE에서는 `/api/admin/data/*` 엔드포인트를 통해 Data 파이프라인에 HTTP 요청을 보낸다. (`X-Internal-Secret` 헤더로 인증) → 상세 명세: [spec/api/pipeline.md](../api/pipeline.md)
