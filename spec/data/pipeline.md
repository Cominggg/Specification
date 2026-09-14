# 데이터 수집 파이프라인 명세

Python 별도 레포지토리(`coming-data`)로 구현. Spring 백엔드와 동일한 DB를 공유하며, 스키마 변경은 Spring 쪽 마이그레이션 도구로 단일 관리(DML만 사용).

이 문서는 파이프라인 전체 개요·스케줄 요약 역할을 한다. 세부 로직은 아래 문서를 참고한다.

- [scheduler.md](scheduler.md) — 잡별 cron 시각·진입점·의존관계·재시도/misfire 정책, CLI 커맨드
- [matchers.md](matchers.md) — 공연-아티스트 매칭 함수 로직
- [error-handling.md](error-handling.md) — 공통 예외 처리·재시도·체크포인트(재개) 컨벤션
- [collectors/musicbrainz.md](collectors/musicbrainz.md) — MusicBrainz 아티스트 수집 + Last.fm 인기도 필터
- [collectors/release.md](collectors/release.md) — Spotify 릴리즈(앨범·싱글) + 트랙·커버아트 수집
- [collectors/kopis.md](collectors/kopis.md) — KOPIS 공연 수집·상태 갱신
- [collectors/setlist.md](collectors/setlist.md) — setlist.fm 셋리스트 수집
- [collectors/ja_romanize.md](collectors/ja_romanize.md) — 로마자 표기 기반 한글 alias 자동 변환
- [collectors/artist_image.md](collectors/artist_image.md) — Spotify 아티스트 프로필 이미지 수집
- ERD: [erd.md](erd.md)

## 레포지토리 구조

```
coming-data/
├── collectors/
│   ├── musicbrainz.py     # MusicBrainz 아티스트 수집 + Last.fm 인기도 필터
│   ├── release.py         # Spotify 릴리즈(앨범·싱글) + 트랙·커버아트 수집
│   ├── kopis.py           # KOPIS 공연 수집·상태 갱신
│   ├── setlist.py         # setlist.fm 셋리스트 수집
│   ├── ja_romanize.py     # sort_name 로마자 표기 → 한글 alias 규칙 변환
│   ├── artist_image.py    # Spotify 아티스트 프로필 이미지 수집
│   └── spotify_client.py  # Spotify Client Credentials 토큰 발급·캐싱, 공통 GET 래퍼
├── matchers/
│   └── artist_matcher.py  # 공연명-아티스트 alias 구문 매칭
├── db/
│   ├── connection.py      # SQLAlchemy 엔진·세션
│   └── repository.py      # DB 저장·조회 함수 (DML)
├── tests/                 # pytest 단위 테스트
├── scheduler.py           # 잡 정의 + APScheduler 데몬 진입점 + CLI
└── api.py                 # 어드민 연동용 내부 FastAPI (X-Internal-Secret 인증)
```

## 수집 단계 개요

### ① 초기 구축 (1회성 · `python scheduler.py init`)

`run_initial_collect()`가 아래 순서로 실행한다 (각 단계는 `--skip-*` 플래그로 건너뛸 수 있음). 상세: [scheduler.md](scheduler.md#run_initial_collect)

1. MusicBrainz 아티스트 수집 (`--skip-artists`, `--force-artists`)
2. 로마자→한글 alias 변환 (`--skip-ja-romanize`)
3. KOPIS 수집·매칭 — `use_prfstate=True`로 검수 큐 없이 실제 KOPIS 상태 그대로 저장 (`--skip-kopis`)
4. 내한 매칭 아티스트 릴리즈 수집 (`--skip-releases`)
5. 아티스트 이미지 수집 (`--skip-artist-image`)
6. setlist.fm 수집 (`--skip-setlist`)

### ② 주기적 수집 (스케줄)

| 작업 | 스케줄 | 진입점 |
|------|--------|--------|
| 공연 상태 갱신 | 매일 04:00 | `run_concert_status_update` |
| 신규 공연 탐지·매칭 | 매일 04:30 | `run_new_concert_collect` |
| 릴리즈 갱신 | 매일 05:00 | `run_release_update` |
| 아티스트 이미지 수집 | 목요일 02:00 | `run_artist_image_update` |
| 로마자→한글 alias 변환 | 목요일 03:00 | `run_ja_romanize_collect` |
| setlist.fm 수집 | 매일 06:00 | `run_setlist_collect` |

cron 정의·의존관계·실패 시 정책은 [scheduler.md](scheduler.md) 참고.

### ③ 매칭

공연명(`prfnm`)과 아티스트 alias 간 구문 일치 단일 전략만 사용한다 (2026-06-30부로 `prfcast` 기반 완전일치·퍼지매칭 단계 폐기). 상세: [matchers.md](matchers.md)

- 매칭 성공 시 `concert_artist_candidate`에 INSERT (`(concert_id, artist_id)` 만 저장, `matched_by` 컬럼 없음 — title 단일 전략이라 값 자체가 의미 없어 제거됨)
- 어드민 승인 시 `concert_artist`로 이동, 거절 시 `concert.status = EXCLUDED`
- 저장 전 `has_match()` 통과 공연만 DB에 저장 (`run_new_concert_collect`) — 비매칭 공연은 저장하지 않음

### ④ 공연-아티스트 관계 구조

공연과 아티스트는 다대다(M:N) 관계. `concert_artist` 중간 테이블로 관리하며, 파이프라인 매칭 결과는 먼저 `concert_artist_candidate`에 임시 저장되고 어드민 승인 후 `concert_artist`로 이동한다.

| 테이블 | 컬럼 | 비고 |
|--------|------|------|
| `concert_artist` | `concert_id`, `artist_id` | 어드민 승인 후 확정된 관계. `UNIQUE(concert_id, artist_id)` |
| `concert_artist_candidate` | `concert_id`, `artist_id` | 파이프라인 매칭 후 검토 대기. `UNIQUE(concert_id, artist_id)` |

> 단독 공연은 후보 행이 1개, 합동 공연(페스티벌 등)은 참여 아티스트 수만큼 행이 생성된다.

### ⑤ is_coming 동기화

`update_artist_is_coming()` (`db/repository.py`) — `artist.is_coming`을 다음 조건의 존재 여부로 갱신한다.

```sql
concert_artist 존재
  AND concert.status NOT IN ('EXCLUDED', 'PENDING')
  AND concert.end_date >= CURRENT_DATE
```

- 호출 시점: `run_concert_status_update` 완료 후, `collect_and_save_concert` 단건 수집 성공 후 (어드민이 PENDING 공연을 `concert_artist`로 승인할 때는 BE 쪽에서 별도 처리)
- `IS DISTINCT FROM` 조건으로 값이 실제로 바뀌는 행만 UPDATE

### ⑥ 관리자 검색·단건 수집 (내부 API 트리거)

BE는 `X-Internal-Secret` 헤더로 `api.py`(FastAPI, 기본 포트 8000)를 호출한다. 엔드포인트별 동기/비동기 여부와 동시 실행 가드는 [scheduler.md#api](scheduler.md#어드민-연동-내부-api-apipy) 참고.

| 엔드포인트 | 처리 방식 | 진입점 함수 |
|------------|-----------|-------------|
| `GET /search/artists` | 동기 | `musicbrainz.search_artists` |
| `GET /search/concerts` | 동기 | `kopis.search_concerts` |
| `POST /collect/artist` | 동기 | `scheduler.register_artist_by_mbid` |
| `POST /collect/concert` | 동기 | `scheduler.collect_and_save_concert` |
| `POST /collect/concert/{id}/setlist` | 동기 | `scheduler.collect_and_save_setlist` |
| `POST /collect/artist/{id}/releases` | 비동기(202 + BackgroundTasks) | `scheduler.collect_and_save_releases_for_artist` |

BE의 `/api/admin/data/*` 엔드포인트 명세: [spec/api/pipeline.md](../api/pipeline.md)
