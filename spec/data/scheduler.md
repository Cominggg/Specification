# 스케줄러 명세 (scheduler.py)

`APScheduler`(`BackgroundScheduler`, `timezone="Asia/Seoul"`)로 구동. 데몬 실행 시(`python scheduler.py`, 인자 없음) 스케줄러와 내부 FastAPI(`api.py`)가 같은 프로세스에서 함께 뜬다 (`threading.Thread(daemon=True)`로 uvicorn 실행).

## 잡 등록 (`_build_scheduler`)

| 잡 | cron | 진입점 함수 | 역할 |
|----|------|-------------|------|
| 공연 상태 갱신 | `hour=4, minute=0` (매일) | `run_concert_status_update` | 활성 공연(UPCOMING/ONGOING) 상태 재조회, `is_coming` 동기화 |
| 신규 공연 탐지 | `hour=4, minute=30` (매일) | `run_new_concert_collect` | KOPIS 신규·수정 공연 증분 탐지 → alias 매칭 → `concert_artist_candidate` 저장 |
| 릴리즈 갱신 | `hour=5` (매일) | `run_release_update` | 내한 매칭 아티스트의 Spotify 신규 릴리즈 수집 |
| 아티스트 이미지 수집 | `day_of_week="thu", hour=2` | `run_artist_image_update` | `image_url` 미수집 아티스트의 Spotify 프로필 이미지 수집 |
| 로마자→한글 alias 변환 | `day_of_week="thu", hour=3` | `run_ja_romanize_collect` | `sort_name` 규칙 변환으로 `ko` alias 보완 (외부 API 미사용) |
| setlist 수집 | `hour=6` (매일) | `run_setlist_collect` | `ENDED` 상태 공연의 setlist.fm 셋리스트 수집 |

- 등록 순서와 실행 순서는 무관하다 — cron 표현식이 유일한 트리거이며 잡 간 명시적 선행 의존관계는 스케줄러 레벨에 없다. 단, **아티스트 이미지(목 02:00) → 로마자 변환(목 03:00) → 신규 공연 탐지(매일 04:30) → 릴리즈 갱신(매일 05:00)** 순으로 시각을 배치해 같은 날 실행 시 데이터 정합성이 자연히 맞도록 되어 있다(예: 신규 alias가 생긴 뒤 매칭이 돌도록).
- APScheduler misfire(스케줄러가 다운돼 있던 사이 지나간 실행 시각)에 대한 별도 `misfire_grace_time` 설정은 없다 — 기본값(다음 실행 시각까지 유효, 지나면 스킵) 적용.
- 잡 실행 중 예외가 발생해도 APScheduler는 다음 스케줄에 정상적으로 재실행한다 (각 `run_*` 함수 내부에서 개별 아이템 단위로 예외를 흡수하므로 잡 전체가 죽는 경우는 각 함수의 최상위 미처리 예외뿐).

## 각 잡의 상세 로직

### `run_initial_collect(...)` — 초기 수집 (CLI `init` 전용)

```
run_initial_collect(
    skip_artists=False, force_artists=False, skip_kopis=False,
    skip_ja_romanize=False, skip_releases=False,
    skip_artist_image=False, skip_setlist=False,
) -> None
```

순서: 아티스트 수집 → 로마자 alias 변환 → KOPIS 수집·매칭 → 릴리즈 수집 → 이미지 수집 → setlist 수집.

- 아티스트 수집: `force_artists=False`(기본)이면 `get_all_artist_mbids()`로 기존 MBID를 조회해 `musicbrainz.collect_artists(skip_mbids=...)`에 전달 — 재개 지원. `force_artists=True`면 빈 집합을 넘겨 전체 재수집.
- KOPIS 수집: `run_new_concert_collect(stdate="20230101", use_prfstate=True)` 호출 — 초기 수집은 검수 큐(PENDING) 없이 KOPIS `prfstate`를 그대로 반영해 저장한다.
- 릴리즈 수집: `_is_banned()`가 True면(Spotify 429 밴 유효) 전체 건너뜀. `get_matched_artists_with_spotify()`(내한 매칭 + Spotify URL 보유)만 대상. 체크포인트 키 `"initial_collect"`로 재개 지원.
- `SpotifyRateLimitError` 발생 시 해당 아티스트까지 진행 상황을 체크포인트에 저장하고 릴리즈 수집 루프를 `break` — 이미지·setlist 수집은 계속 진행된다.
- `for...else`: 루프가 `break` 없이 끝나면(전원 처리 완료) `_clear_progress("initial_collect")`로 체크포인트 삭제.

### `run_ja_romanize_collect()` — 로마자→한글 alias 변환

```
run_ja_romanize_collect() -> None
```

`get_artists_without_ko_alias()`로 `locale='ko'` alias가 없는 아티스트를 조회 → `ja_romanize.collect_ko_aliases()`로 변환 → `save_aliases()`. 외부 API 호출이 없으므로 실패·재시도 로직이 없다. 대상이 0건이면 즉시 종료.

### `run_concert_status_update()` — 공연 상태 갱신

```
run_concert_status_update() -> None
```

1. `get_active_concerts()`로 `status IN ('UPCOMING','ONGOING')` 공연 조회
2. `ThreadPoolExecutor(max_workers=8)`로 `kopis.collect_by_id(kopis_id)` 병렬 호출
3. 개별 future 예외는 `logger.warning`으로 흡수하고 계속 진행 (`future.result()`를 개별 try/except로 감쌈 — 한 건 실패가 전체를 막지 않음)
4. 성공 응답만 모아 `update_concert_status()` 일괄 호출 — `updatedate` 변화 감지 시에만 `status`·`kopis_update_date` UPDATE
5. `update_artist_is_coming()` 항상 호출

### `run_new_concert_collect(stdate=None, use_prfstate=False)` — 신규 공연 탐지·매칭

```
run_new_concert_collect(stdate: Optional[str] = None, use_prfstate: bool = False) -> None
```

1. `_load_last_collect_date()`로 마지막 성공 수집일(`YYYYMMDD`)을 로드 — 있으면 `kopis.collect(afterdate=last_date)`로 증분 스캔, 없으면 `stdate`(기본 `20250101`) 기준 전체 스캔
2. `has_match()`로 alias 매칭되는 신규 공연만 필터링 (`kopis_id not in existing_ids`)
3. `save_concerts(new_concerts, use_prfstate=use_prfstate)` — 스케줄러 잡(기본 `use_prfstate=False`)은 무조건 `status='PENDING'`으로 저장해 검수 큐로 보냄
4. `get_unmatched_concerts()`(매칭 없고 `status != EXCLUDED`인 공연)에 대해 `match_concert()` 재실행 → `save_concert_artist_candidates()`로 후보 저장
5. `update_artist_is_coming()` 호출 후 `_save_last_collect_date(오늘)`로 체크포인트 갱신 — **예외 여부와 무관하게 함수 끝에서 항상 갱신**되므로, KOPIS 응답이 일부 누락된 채 실패해도 다음 실행은 그 시점부터 증분 스캔한다 (놓친 구간은 KOPIS `afterdate`가 갱신 기준 필드이므로 후속 수정분은 재탐지됨).

### `run_artist_image_update()` — 아티스트 이미지 수집

```
run_artist_image_update() -> None
```

1. `_is_banned()`면 즉시 종료
2. `get_artists_without_image()` 대상 중 `image_failed` 체크포인트(`{artist_id: 마지막 실패일}`)를 확인해, `_IMAGE_RETRY_DAYS`(30일) 이내 실패 기록이 있는 아티스트는 이번 회차 대상에서 제외 — 매주 동일 잔여 아티스트로 Spotify API를 반복 호출하지 않기 위함
3. 대상별로 `artist_image.collect_artist_image()` 호출. 이미지 획득 시 `update_artist_image()` + 실패 기록 제거. 실패 시 `image_failed[artist_id] = 오늘`
4. Spotify URL이 없었는데 fallback 검색으로 `spotify_id`를 찾은 경우 `upsert_artist_url()`로 DB에 백필
5. `SpotifyRateLimitError` 시 `_save_ban(e.retry_after)`로 밴 해제 시각을 계산하고, `_reschedule_on_ban_lift(run_artist_image_update, banned_until)`로 밴 해제 시각에 동일 잡을 1회성 재실행 예약 후 루프 중단

### `run_release_update()` / `run_missing_release_update()` — 릴리즈 갱신

```
run_release_update() -> None            # 매일 05:00, 내한 매칭 아티스트 전체 대상
run_missing_release_update() -> None    # 복구 전용, 릴리즈가 아예 없는 아티스트만 대상
```

공통 로직: `_is_banned()` 시 종료 → 체크포인트(`"release_update"` / `"missing_release"`)로 완료된 `artist_id` 건너뜀 → `release.collect_releases(spotify_id, skip_spotify_ids=기존 spotify_id 집합, cached_total=DB 캐시값)` 호출 → `spotify_total`이 DB 캐시와 다르면 `update_spotify_album_total()` 갱신 → 신규 릴리즈가 있으면 `_sort_releases()`(Album 우선, Single 후순)로 정렬 후 `save_releases()`.

`SpotifyRateLimitError` 처리는 `run_artist_image_update`와 동일하게 밴 저장 + 재실행 예약(단, `run_missing_release_update`는 CLI `recover` 전용 1회성 커맨드라 재스케줄하지 않는다).

### `run_setlist_collect()` — setlist 수집

```
run_setlist_collect() -> None
```

`setlist.collect()` → `save_setlists()`. 대상 선정과 재시도 로직은 `setlist.py` 내부([collectors/setlist.md](collectors/setlist.md)).

## 어드민 연동 내부 API (`api.py`)

FastAPI 앱(`docs_url=None`), 데몬 실행 시 별도 스레드로 uvicorn 구동 (`API_HOST`/`API_PORT` 환경변수, 기본 `0.0.0.0:8000`). 모든 엔드포인트는 `_verify_secret` 의존성으로 `X-Internal-Secret` 헤더를 `INTERNAL_SECRET` 환경변수와 비교 — 불일치·미설정 시 `401`.

| 엔드포인트 | 방식 | 동시 실행 가드 | 진입점 |
|-----------|------|----------------|--------|
| `GET /search/artists?name=` | 동기 | 없음 | `musicbrainz.search_artists` |
| `GET /search/concerts?title=` | 동기 | 없음 | `kopis.search_concerts` |
| `POST /collect/artist` (body: `mbid`) | 동기 | `("artist", mbid)` 키, 중복 시 `409` | `scheduler.register_artist_by_mbid` |
| `POST /collect/concert` (body: `kopis_id`) | 동기 | `("concert", kopis_id)` 키, 중복 시 `409` | `scheduler.collect_and_save_concert` |
| `POST /collect/concert/{id}/setlist` | 동기 | `("setlist", concert_id)` 키, 중복 시 `409` | `scheduler.collect_and_save_setlist` |
| `POST /collect/artist/{id}/releases` | 비동기(`202`, `BackgroundTasks`) | `("releases", artist_id)` 키, 중복 시 `{"accepted": false}` | `scheduler.collect_and_save_releases_for_artist` |

- 동시 실행 가드(`_running_tasks: set`, `threading.Lock`)는 프로세스 메모리 내 상태 — 멀티 프로세스로 확장 시 별도 락(Redis 등)으로 교체 필요.
- `_sync_response()`가 `scheduler.py`의 `{"status": "not_found"|"skipped"|"ok", ...}` discriminated dict를 HTTP 응답으로 변환: `not_found` → `404`, `skipped` → `{"success": false, "reason": ...}`, `ok` → `{"success": true, ...}`.
- 비동기 엔드포인트(`/collect/artist/{id}/releases`)는 작업 완료 후 `_wrap()`이 `finally`로 `_release_task()`를 호출해 락을 반드시 해제한다 — 예외 발생 시에도 동일.

## 체크포인트·복구 파일 (`scheduler.py` 내부 상태)

`checkpoints/` 디렉토리(스크립트 기준 상대경로)에 잡별 JSON 파일로 진행 상태를 저장한다. 각 파일의 읽기/쓰기 실패는 모두 `try/except Exception`으로 흡수하고 `logger.warning`만 남긴다(체크포인트 손상이 파이프라인 전체를 막지 않도록).

| 파일 | 용도 | 갱신 시점 |
|------|------|-----------|
| `spotify_ban.json` | Spotify 429 밴 해제 시각(`banned_until`) | `_save_ban()` — 429 수신 시 |
| `status_update.json` | 신규 공연 탐지 마지막 성공 수집일 | `_save_last_collect_date()` — `run_new_concert_collect` 종료 시 항상 |
| `image_failed.json` | `{artist_id: 마지막 실패일}` | `run_artist_image_update` 매 실행마다 |
| `<job_name>.json` (`initial_collect`/`release_update`/`missing_release`) | 완료된 `artist_id` 집합 | 각 릴리즈 수집 루프 진행 중, 정상 완료 시 삭제(`_clear_progress`) |

- `_migrate_legacy_checkpoint()`: 구버전 단일 파일 `release_sync.json`(밴 시각 + 완료 목록 통합)이 남아 있으면 `spotify_ban.json`과 `release_update.json`으로 분리 이전 후 삭제. `main()` 최초 진입 시 1회 실행.
- 상세 재시도/백오프 정책은 [error-handling.md](error-handling.md) 참고.

## CLI 커맨드 (`main()`)

| 커맨드 | 인자 | 동작 |
|--------|------|------|
| `init` | `--skip-artists` `--force-artists` `--skip-kopis` `--skip-ja-romanize` `--skip-releases` `--skip-artist-image` `--skip-setlist` `--log-file` | `run_initial_collect()` 실행 후 종료 |
| `recover` | `--skip-releases` `--skip-artist-image` | `run_recover()` — 이미지 미해결 + 릴리즈 0건 아티스트만 재수집 |
| `collect-release` | `--artist-id`(필수) | `collect_and_save_releases_for_artist()` 단건 실행 |
| `collect-setlist` | 없음 | `run_setlist_collect()` 실행 |
| `run-job` | `--job` (`concert-status-update`/`new-concert-collect`/`release-update`/`ja-romanize`/`artist-image`/`setlist`) | 지정 잡 함수를 즉시 1회 실행 |
| (인자 없음) | 없음 | 데몬 모드 — API 서버 스레드 시작 + `_build_scheduler().start()` 후 `Ctrl+C`까지 대기 |

각 CLI 실행은 `_attach_file_handler()`로 `logs/` 디렉토리에 타임스탬프 로그 파일을 별도 연결한다 (`--log-file` 지정 시 기존 파일에 이어쓰기). 데몬 모드는 `logs/scheduler.log`에 `TimedRotatingFileHandler`(자정 교체, 30일 보관)를 사용한다.
