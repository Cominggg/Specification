# setlist.fm 셋리스트 수집 (collectors/setlist.py)

## 외부 API

```
GET https://api.setlist.fm/rest/1.0/search/setlists
    ?artistMbid={mbid}&countryCode=KR&p={page}
```

- Header: `x-api-key: {SETLISTFM_API_KEY}`(환경변수 필수, 없으면 모듈 임포트 시 `ValueError`), `Accept: application/json`
- Timeout: 30초
- 응답: `{"setlist": [{"id", "eventDate"(dd-MM-yyyy), "url", "sets": {"set": [{"song": [{"name","info"}]}]}}], "total": int, "itemsPerPage": int}`
- 요청 성공 시 `time.sleep(1.0)` 후 반환 (rate limit 예방)
- 페이지네이션 상한: `_MAX_PAGES = 10`

## 함수별 로직

### `_get(path, params) -> dict`

공통 GET. 재시도 정책:
- `404`는 **즉시 재전파**(재시도하지 않음) — "해당 아티스트 셋리스트 없음"을 의미
- 그 외 `requests.HTTPError`/`requests.RequestException`은 최대 3회 재시도, `time.sleep(5 * attempt)` 백오프

### `_parse_tracks(sets_data: dict) -> list[dict]`

`sets_data["set"]` 배열(공연 중 여러 세트 — 본공연/앙코르 등)을 순회하며 `song[]`을 1부터 시작하는 연속 `position`으로 평탄화. `info`(예: "encore")는 그대로 보존.

### `_event_date_in_range(event_date_str, prfpdfrom, prfpdto) -> bool`

setlist.fm의 `eventDate`(`dd-MM-yyyy`)를 파싱해 공연 기간(`prfpdfrom`~`prfpdto`, `YYYY-MM-DD`) 안에 있는지 확인. 날짜 포맷이 다르므로 각각 다른 `strptime` 패턴 사용. 파싱 실패(`ValueError`/`TypeError`) 시 `logger.warning` 후 `False`(매칭 실패로 처리 — 잘못된 셋리스트를 공연에 잘못 연결하지 않기 위한 안전한 기본값).

### `collect_for_concert(concert: dict) -> Optional[dict]`

단건 API 트리거용(`scheduler.collect_and_save_setlist`). `concert = {"concert_id", "artist_mbid", "start_date", "end_date"}`.

- 페이지를 순회하며 `artistMbid` 검색 결과 중 `_event_date_in_range()`를 통과하는 첫 셋리스트를 반환
- `404` 발생 시 조용히 `None` 반환(로그 없음). 그 외 예외는 `logger.warning` 후 `None`
- 반환: `{"concert_id", "setlist_fm_id", "attribution_url", "tracks"}`

### `collect() -> list[dict]`

배치 수집 메인 함수(`scheduler.run_setlist_collect`). `db.repository.get_completed_concerts()`(setlist 미수집·미시도 또는 7일 경과한 `ENDED` 공연)를 대상으로 순회.

- `seen_concert_ids`로 같은 회차 내 중복 처리 방지
- 페이지별로 `_event_date_in_range()`를 통과하는 첫 셋리스트를 찾으면 결과에 추가하고 해당 공연 순회 종료
- **모든 경로(404·빈 결과·매칭 성공·페이지 상한 도달)에서 `update_concert_fetch_attempted(concert_id)`를 호출** — `concert.fetch_attempted_at`을 갱신해 다음 조회 대상에서 7일간 제외 (동일 공연에 대한 API 재호출 낭비 방지)
- `_MAX_PAGES(10)` 도달 시에도 조용히 다음 공연으로 진행

## 예외 처리 요약

| 상황 | 처리 |
|------|------|
| `404`(셋리스트 없음) | 재시도 없이 즉시 처리 종료, `fetch_attempted_at` 갱신 |
| 기타 HTTP/네트워크 오류 | 최대 3회 재시도(`5*attempt`초), 3회 실패 시 `logger.warning` 후 해당 공연 스킵 + `fetch_attempted_at` 갱신 |
| 날짜 파싱 실패 | 매칭 실패로 간주(`False`), 셋리스트 오귀속 방지 |
| 대상 없음(`get_completed_concerts()` 빈 배열) | 즉시 종료, API 호출 없음 |
