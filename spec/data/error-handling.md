# 공통 예외 처리 컨벤션

파이프라인 전반에서 반복되는 예외 처리·재시도·복구 패턴을 정리한다. 개별 API별 에러 케이스는 각 `collectors/*.md` 참고.

## 원칙

1. **한 건 실패가 전체를 막지 않는다.** 아티스트·릴리즈·공연 등 항목 단위 반복 루프는 개별 `try/except`로 감싸고 실패 항목만 로그 후 건너뛴다 (`db/repository.py`의 `save_artists`, `save_releases`, `save_aliases`; `scheduler.py`의 릴리즈 수집 루프 등).
2. **일시적 네트워크 오류는 재시도, 확정적 오류는 즉시 포기.** `requests.RequestException`(타임아웃·연결 오류·5xx)은 재시도 대상, `404`(리소스 없음) 등 확정적 응답은 즉시 `None`/빈 리스트 반환으로 처리한다.
3. **Rate limit(429)은 재시도하지 않고 즉시 중단 후 재스케줄한다.** 짧은 대기 후 재시도가 아니라, 밴 해제 시각까지 해당 잡 자체를 미룬다.
4. **DB 저장 실패는 로깅만 하고 다음 항목으로 진행한다.** `SQLAlchemyError`를 잡아 해당 레코드만 건너뛰며, 전체 배치 트랜잭션을 롤백하지 않는다 (레코드별 독립 `with get_session()` 블록).

## 재시도 정책 (모듈별)

| 모듈 | 대상 | 재시도 횟수 | 백오프 | 예외 유형 |
|------|------|------|--------|-----------|
| `musicbrainz.py` `collect_artists` | 검색 페이지·아티스트 상세 조회 | 3회 | `time.sleep(5 * attempt)` (5s/10s) | `requests.RequestException` |
| `kopis.py` `collect` | 목록 페이지 조회 | 3회 | `time.sleep(2 ** attempt)` (2s/4s/8s) | `requests.HTTPError`. 3회 실패 + `status==400`이면 마지막 페이지로 간주하고 정상 종료 |
| `kopis.py` `_fetch_and_merge` | 상세 API 병합 | 3회 | `time.sleep(5 * attempt)` | `requests.RequestException`. 3회 실패 시 예외를 던지지 않고 `_DETAIL_FALLBACK`(모든 필드 `None`/빈 값)으로 대체 |
| `setlist.py` `_get` | 검색 API | 3회 | `time.sleep(5 * attempt)` | `requests.HTTPError`(404 제외)·`requests.RequestException`. `404`는 즉시 raise(재시도 안 함) |
| `release.py`, `artist_image.py` (Spotify) | 앨범 목록·상세 조회 | 재시도 없음 | - | `SpotifyRateLimitError`는 즉시 상위로 전파, 그 외 `requests.RequestException`은 로그 후 해당 항목만 건너뜀 |
| `ja_romanize.py` | - | 해당 없음 (외부 API 미사용) | - | - |

## Spotify 429 밴 처리 (`scheduler.py`)

Spotify Web API는 30초 롤링 윈도우 rate limit을 사용하며, `collectors/spotify_client.py`의 `spotify_get()`이 `429` 수신 시 응답의 `Retry-After` 헤더를 읽어 `SpotifyRateLimitError(retry_after=...)`를 즉시 발생시킨다 (재시도하지 않음).

호출부(`scheduler.py`) 공통 처리:

1. `except SpotifyRateLimitError as e:` — 현재까지 처리한 항목을 체크포인트(`_save_progress`)에 저장
2. `_save_ban(e.retry_after)` — `retry_after`가 양수면 `now + retry_after + 5초 안전마진`, 아니면 고정 `_BAN_MARGIN_HOURS=25시간` 후를 `banned_until`로 계산해 `checkpoints/spotify_ban.json`에 저장
3. 데몬 모드(`_scheduler`가 초기화된 경우)라면 `_reschedule_on_ban_lift(job_func, banned_until)`로 밴 해제 시각에 동일 잡을 1회성(`trigger="date"`) 재실행 예약. `job_id=f"resume_{job_func.__name__}"`로 고정해 재밴 발생 시에도 `replace_existing=True`로 중복 등록 방지
4. `run_artist_image_update`, `run_release_update`는 재스케줄 대상. `run_missing_release_update`(CLI `recover` 전용 1회성)는 재스케줄하지 않음
5. 이후 밴이 유효한 동안 실행되는 다른 Spotify 관련 잡들은 `_is_banned()` 체크로 즉시 스킵 (밴 해제 전 불필요한 429 재유발 방지)

## 체크포인트 기반 재개(resume)

장시간 실행되는 배치(아티스트 초기 수집, 릴리즈 수집)는 `artist_id` 단위로 완료 여부를 JSON 파일에 기록해, 프로세스가 중단되더라도 다음 실행 시 완료분을 건너뛴다.

- 저장: 처리 성공한 `artist_id`를 `set`에 누적, 429나 예외로 중단 시점까지의 집합을 저장
- 삭제: `for...else` 패턴으로 루프가 `break` 없이 끝나면(전원 처리 완료) 체크포인트 파일 삭제 — 다음 실행은 처음부터 전체 스캔
- 손상 시 폴백: JSON 파싱 실패·파일 없음은 모두 빈 `set()`으로 폴백하고 `logger.warning`만 남김 (`_load_progress`, `_load_last_collect_date`, `_load_failed_image_artists` 공통 패턴)

## 로깅 규칙

- `logging`만 사용, `print` 금지 (`CLAUDE.md` 코딩 규칙)
- 포맷: `%(asctime)s [%(levelname)s] %(name)s — %(message)s`
- 각 잡은 `=== {잡 이름} 시작/완료 ===` 로그로 경계를 명시
- 실패 로그는 항상 식별자(`mbid`, `kopis_id`, `artist_id`, `concert_id`)를 포함해 재현·추적 가능하게 남긴다
- CLI 실행은 `logs/{command}_{timestamp}.log`에 파일 핸들러를 추가 연결, 데몬 모드는 `logs/scheduler.log`에 자정 로테이션(30일 보관) 핸들러 사용

## 내부 API(`api.py`) 동시 실행 가드

동일 리소스(아티스트·공연·셋리스트)에 대한 중복 요청은 `(리소스타입, id)` 키를 메모리 `set` + `threading.Lock`으로 관리해 거부한다.

- 동기 엔드포인트: 이미 실행 중이면 `409 Conflict`
- 비동기 엔드포인트(`/collect/artist/{id}/releases`): 이미 실행 중이면 `202` 대신 `{"accepted": false, "reason": "already running"}` 반환 (에러 아님)
- 작업 종료 시 `finally`(동기) 또는 `_wrap()`의 `finally`(비동기)로 반드시 키를 해제 — 예외 발생 시에도 락이 영구히 남지 않도록 보장
