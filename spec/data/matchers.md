# 매칭 로직 명세 (matchers/artist_matcher.py)

공연명(`prfnm`/`title`)과 아티스트 alias 간 **구문 일치(phrase match) 단일 전략**만 사용한다. `prfcast`(출연진) 기반 완전일치·`rapidfuzz` 퍼지매칭 단계는 2026-06-30부로 폐기되었다 (ERD 변경이력 `concert_artist_candidate.matched_by` 컬럼 제거와 함께 정리됨). `rapidfuzz`는 이 모듈에서 더 이상 import되지 않는다.

## `_normalize(text: str) -> str`

```python
re.sub(r"\W+", " ", text.lower()).strip()
```

소문자화 후 비단어 문자(`\W`, 공백·기호 포함) 연속 구간을 공백 하나로 치환. 한글·영문·숫자·언더스코어만 토큰 경계 판단에 사용됨.

## `_phrase_match_title(title: str, aliases: list[dict]) -> list[dict]`

입력: `aliases = [{"artist_id": int, "name": str}, ...]` (alias + `artist.name` 통합 목록, `db.repository.get_all_aliases()` 참고)

로직:
1. `_MIN_ALIAS_LEN = 2` — 정규식으로 비단어 문자를 제거한 alias 길이가 2 미만이면 매칭 대상에서 제외 (1글자 alias의 오탐 방지)
2. `title_norm = _normalize(title)`, `title_tokens = set(title_norm.split())`, `title_padded = f" {title_norm} "`
3. alias별 매칭 기준을 단어 수로 분기:
   - **단일 단어 alias**: `name_norm in title_tokens` — 토큰 집합 완전 일치 (부분 문자열 매칭 아님)
   - **다중 단어 alias**: `f" {name_norm} " in title_padded` — 앞뒤에 공백을 덧대 구문 전체가 단어 경계 안에서 연속으로 등장하는지 확인 (예: `"IU"`가 `"IUNIVERSE"`의 부분 문자열이어도 매칭되지 않음)
4. 동일 `artist_id`에 여러 alias가 매칭되면 **가장 긴 이름**(`len(name)` 최대)만 채택 — 특이도가 높은(더 구체적인) alias를 우선
5. 서로 다른 `artist_id`가 모두 매칭되면 전부 반환 (합동 공연·페스티벌 지원)

## `has_match(concert_raw: dict, aliases: list[dict]) -> bool`

```python
has_match(concert_raw, aliases) -> bool
```

`concert_raw["prfnm"]`(KOPIS 원시 필드)에 대해 `_phrase_match_title()` 결과가 비어있지 않은지만 확인. **저장 전 필터링 전용** — `run_new_concert_collect`·`collect_and_save_concert`에서 alias 매칭이 되는 공연만 DB에 저장하기 위해 사용한다. 비매칭 공연은 이 시점에서 완전히 버려지며 별도 검토 큐로 가지 않는다.

## `match_concert(concert: dict, aliases: list[dict]) -> tuple[list[dict], list[dict]]`

입력: `concert = {"concert_id": int, "title": str}` (DB 저장된 공연, KOPIS 원시 데이터 아님 — `title` 컬럼명 사용)

```python
match_concert(concert, aliases) -> (matches, failures)
```

- `title`이 매칭되면 `matches = [{"concert_id": ..., "artist_id": ...}, ...]`(매칭된 모든 alias) 반환, `failures = []`
- 매칭이 없으면 `matches = []`, `failures = [{"concert_id": concert_id}]` 반환
- 매칭/실패 각각 `logger.debug`로 기록

호출부(`scheduler.run_new_concert_collect`, `scheduler.collect_and_save_concert`)는 `matches`를 각각 `save_concert_artist_candidates()` / `save_concert_artists()`에 전달한다. `failures`는 현재 별도 저장·알림 없이 로그로만 남는다 — 실패한 공연은 다음 잡 실행 시 `get_unmatched_concerts()`(매칭 없고 `status != EXCLUDED`)로 다시 조회되어 재시도된다.

## 예외 처리

이 모듈은 외부 API를 호출하지 않으므로 재시도·백오프 로직이 없다. 입력 딕셔너리에 필요한 키(`prfnm`/`title`)가 없을 경우 `.get(..., "")`로 빈 문자열 기본값을 사용해 `KeyError`를 방지한다 (`has_match`, `match_concert` 모두 동일).
