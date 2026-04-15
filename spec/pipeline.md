# 데이터 수집 파이프라인 명세

Python 별도 레포지토리로 구현. Spring 백엔드와 동일한 DB를 공유하며, 스키마 변경은 Spring 쪽 마이그레이션 도구로 단일 관리.

## 레포지토리 구조

```
jpop-concert-collector/
├── collectors/
│   ├── kopis.py           # KOPIS API 수집
│   ├── musicbrainz.py     # MusicBrainz 아티스트·멤버 수집
│   ├── release.py         # MusicBrainz 릴리즈(앨범·싱글·EP) + 트랙·크레딧·커버 수집
│   ├── lastfm.py          # Last.fm 앨범 소개(wiki) 수집
│   └── setlist.py         # setlist.fm 셋리스트 수집
├── matchers/
│   └── artist_matcher.py  # alias 기반 매칭 로직 (rapidfuzz)
├── db/
│   └── repository.py      # DB 저장 (SQLAlchemy)
├── scheduler.py           # APScheduler 진입점
└── requirements.txt
```

## 수집 파이프라인 단계

### ① 초기 구축 (1회성)

| 작업 | 상세 |
|------|------|
| MusicBrainz 아티스트 수집 | JP 아티스트 목록 수집. country=JP, tag=j-pop 조건. 이름·alias(한/영/일)·url-rels·멤버 구성·데뷔일 포함 저장.<br>`GET /ws/2/artist/?query=tag:j-pop AND country:JP&limit=100&offset={n}&fmt=json`<br>개별 상세: `GET /ws/2/artist/{mbid}?inc=aliases+tags+url-rels+artist-rels&fmt=json`<br>응답의 `relations` 배열에서 `type: "member of band"` 항목을 파싱해 멤버 구성 저장. 전·현 멤버 구분은 `ended` 필드 기준.<br>데뷔일: `life-span.begin` 필드 저장. 값이 없는 경우 null 허용.<br>※ Rate Limit: 1 req/sec |
| MusicBrainz 릴리즈 수집 | 등록된 아티스트의 앨범·싱글·EP 초기 수집.<br>`GET /ws/2/release-group/?artist={mbid}&type=album%7Csingle%7Cep&limit=100&offset={n}&fmt=json`<br>취득 필드: `title`(제목), `first-release-date`(발매일), `primary-type`(Album·Single·EP).<br>▸ **트랙·크레딧 수집** (REL-02): release-group별 대표 release MBID 취득 후 개별 상세 호출.<br>`GET /ws/2/release/{release-mbid}?inc=recordings+artist-credits+labels&fmt=json`<br>수록곡(`media[].tracks`): 트랙 번호·제목·`length` 저장. 크레딧(`artist-credit`): 작곡·작사·편곡 저장. 레이블: `label-info[].label.name` 저장.<br>▸ **앨범 커버 URL** (REL-02): Cover Art Archive 존재 여부 확인 후 URL 컬럼 저장.<br>`GET https://coverartarchive.org/release-group/{release-group-mbid}/front` — 404 시 null 허용.<br>※ Rate Limit: 1 req/sec (MusicBrainz), Cover Art Archive 별도 1 req/sec |
| Last.fm 앨범 소개 수집 | 릴리즈별 소개 텍스트 초기 수집 (REL-02).<br>`GET https://ws.audioscrobbler.com/2.0/?method=album.getInfo&artist={artist}&album={title}&api_key={key}&format=json`<br>취득 필드: `album.wiki.summary`. 미존재 시 null 허용.<br>※ Rate Limit: 5 req/s. MusicBrainz 수집과 별도 배치로 분리 실행. |
| 관리자 등록 | MusicBrainz 미등록 아티스트를 관리자 UI로 직접 입력. |

### ② 주기적 수집 (스케줄)

| 작업 | 상세 |
|------|------|
| KOPIS 수집 | 내한 공연 후보 조회. visit=Y, genrenm=대중음악 조건. prfnm·prfcast·날짜·장소·updatedate 저장.<br>`GET /openApi/restful/pblprfr?service={key}&stdate=20200101&eddate={today}&shcate=GGGA&visit=Y&rows=100&cpage={n}`<br>상세: `GET /openApi/restful/pblprfr/{mt20id}?service={key}`<br>상세 API 응답의 `relates` 필드(예매처 링크 목록) 저장. 값이 없는 경우 빈 배열로 처리.<br>※ 주 1회 이상 권장 |
| 상태 갱신 | updatedate 변화 감지 시 prfstate DB 갱신. 매일 실행. |
| 매칭 ① | prfcast 기반 매칭 — 출연진 필드 → Artist DB alias 완전 일치. HIGH 신뢰도로 저장. 관리자 승인 없이 즉시 노출.<br>prfcast에 여러 아티스트가 포함된 경우(`,` · `·` 구분) 각각 개별 매칭 후 모두 `concert_artist`에 INSERT (합동 공연 지원). |
| 매칭 ② | prfnm 기반 매칭 — 공연명 문자열 내 alias 부분 검색. `rapidfuzz.fuzz.partial_ratio` 기준 85점 이상 시 매칭 성공. LOW 신뢰도로 저장. 관리자 승인 후 노출.<br>복수 아티스트 매칭 시 각각 별도 행으로 `concert_artist`에 INSERT. |
| 매칭 실패 | 두 매칭 모두 실패 시 검토 큐 등록. 승인 시 alias 학습 → 다음 사이클 자동 매칭률 향상. |
| 릴리즈 갱신 | 등록된 아티스트의 신보 감지. `first-release-date` 기준 DB에 없는 항목만 INSERT. 주 1회 실행.<br>`GET /ws/2/release-group/?artist={mbid}&type=album%7Csingle%7Cep&limit=100&offset={n}&fmt=json`<br>▸ 신규 릴리즈 감지 시 트랙·크레딧·커버·Last.fm 소개 수집도 연속 실행. |

### 공연-아티스트 관계 구조

공연과 아티스트는 **다대다(M:N)** 관계. 합동 공연(페스티벌·조인트 콘서트) 지원을 위해 `concert_artist` 중간 테이블로 관리.

| 컬럼 | 설명 |
|------|------|
| `concert_id` | 공연 FK |
| `artist_id` | 아티스트 FK |
| `confidence` | `HIGH` (prfcast 완전 일치) / `LOW` (prfnm 퍼지 매칭) |
| `matched_by` | `prfcast` / `prfnm` / `manual` |
| `approved` | LOW 매칭의 관리자 승인 여부. HIGH는 항상 `true`. |

> 단독 공연은 `concert_artist` 행이 1개, 합동 공연은 참여 아티스트 수만큼 행이 생성됨.

### ③ 셋리스트 수집 (스케줄)

| 작업 | 상세 |
|------|------|
| setlist.fm 수집 | prfstate=공연완료 대상. 공연 완료 후 1일 이내 실행.<br>`GET https://api.setlist.fm/rest/1.0/search/setlists?artistMbid={mbid}&countryCode=KR&p={n}`<br>Header: `x-api-key`, `Accept: application/json`<br>데이터 미존재 시 빈 상태 유지. |
