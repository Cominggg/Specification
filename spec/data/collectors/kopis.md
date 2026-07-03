# KOPIS 공연 수집 (collectors/kopis.py)

## 외부 API

- Base URL: `http://kopis.or.kr/openApi/restful/pblprfr`
- 인증: 쿼리 파라미터 `service={KOPIS_API_KEY}` (환경변수 필수, 없으면 모듈 임포트 시 `ValueError`)
- 응답 포맷: **XML** (`xml.etree.ElementTree`로 파싱, JSON 아님)
- Timeout: 30초
- Rate limit: 명시된 제한 없음 (문서상 "주 1회 이상 권장"). 상세 API는 `ThreadPoolExecutor(max_workers=8)`로 병렬 호출

### 목록 조회

```
GET /openApi/restful/pblprfr
    ?service={key}&genrenm=GGGA&rows=100&cpage={n}
    &stdate={YYYYMMDD}&eddate={YYYYMMDD}
    &afterdate={YYYYMMDD}   # 선택, 증분 스캔용
```

- `genrenm=GGGA`(대중음악)로 서버 필터링하지만, 응답에 다른 장르가 섞여 오는 경우가 있어 `_parse_concert()` 이전에 `<genrenm>대중음악</genrenm>`인 항목만 2차 필터링한다 (`collect()` 내부)
- `visit=Y`(내한) 필터는 목록 API가 아니라 **상세 API 응답의 `visit` 필드**로 판단 — 목록 조회 시에는 적용되지 않는다
- `afterdate`는 `stdate`/`eddate`와 AND 조건으로 동작 (KOPIS API 명세) — 등록·수정일 기준 증분 필터
- 응답 XML: `<dbs><db><mt20id/><prfnm/><prfcast/><prfpdfrom/><prfpdto/><fcltynm/><prfstate/><genrenm/><updatedate/><relates>...</relates></db>...</dbs>`

### 단건 상세 조회

```
GET /openApi/restful/pblprfr/{kopis_id}?service={key}
```

응답 XML: `<dbs><db>{목록 필드} + <poster/><pcseguidance/><visit/><styurls><styurl/>...</styurls></db></dbs>`

- `pcseguidance` → `price`, `poster` → `poster_url`, `visit` → 내한 여부(`"Y"`/그 외)
- `styurls/styurl[]` → `still_urls` (공연 스틸컷 이미지, 없으면 `[]`)

## 함수별 로직

### `_parse_kopis_date(raw) -> Optional[str]`

정규식 `^(\d{4})[-.](\d{2})[-.](\d{2})`로 앞부분만 추출해 `YYYY-MM-DD`로 정규화. `.`와 `-` 구분자, 뒤에 시각(`HH:MM:SS`)이 붙어도 허용. 매치 실패·빈 값은 `None`.

### `_get(params: dict) -> ET.Element`

목록 API 공통 GET. `ET.ParseError` 시 `logger.error` 후 예외 재전파(상위에서 재시도 판단).

### `_fetch_detail(kopis_id: str) -> dict`

단건 상세 조회(배치 병합용). **XML 파싱 실패 시 예외를 던지지 않고 `_DETAIL_FALLBACK`(모든 필드 `None`/빈 리스트)을 반환** — KOPIS 응답에 이스케이프되지 않은 특수문자(`&`, `<` 등)가 포함돼 파싱이 깨지는 사례에 대한 방어적 폴백.

### `search_concerts(title: str) -> list[dict]`

```
GET ...?shprfnm={title}&stdate=20200101&eddate={오늘+365일}&cpage=1&rows=20
```

어드민 검색용. 실패 시(`requests.RequestException`, `ET.ParseError`) `logger.error` 후 빈 리스트 반환(재시도 없음). 반환 필드: `kopis_id`, `title`, `start_date`, `end_date`, `venue`, `url`(KOPIS 상세 페이지).

### `collect_by_id(kopis_id: str) -> Optional[dict]`

단건 전체 필드(목록+상세 통합) 조회. 데이터 없음(`<db>` 태그 없음) 또는 `ET.ParseError` 시 `None` 반환(재시도 없음). `run_concert_status_update`(활성 공연 상태 갱신)와 `collect_and_save_concert`(단건 등록) 양쪽에서 사용.

### `_fetch_and_merge(concert: dict) -> Optional[dict]`

목록에서 파싱한 `concert` dict에 `_fetch_detail()` 결과를 `dict.update()`로 병합.

- 최대 3회 재시도, `time.sleep(5 * attempt)` 백오프
- 3회 모두 실패하면 `_DETAIL_FALLBACK` 값으로 병합(예외를 던지지 않음) — 상세 실패가 전체 배치 수집을 막지 않도록 함
- 병합 후 `visit != "Y"`이면 `None` 반환 — **내한 공연이 아닌 건은 이 시점에서 제외**

### `collect(stdate=None, eddate=None, afterdate=None) -> list[dict]`

메인 배치 수집 함수. 페이지 단위로 목록 조회 → 대중음악 필터링 → `ThreadPoolExecutor(max_workers=8)`로 상세 병렬 병합.

- `stdate` 기본값 `_DEFAULT_STDATE="20250101"`, `eddate` 기본값 오늘+`_DEFAULT_LOOKAHEAD_DAYS(365)`일
- 목록 조회 최대 3회 재시도, `time.sleep(2 ** attempt)`(2s/4s/8s) 백오프
- **3회 실패 + HTTP 400**이면 예외를 던지지 않고 "마지막 페이지"로 간주해 지금까지 결과를 반환하며 정상 종료(KOPIS는 데이터가 없는 페이지에 400을 반환하는 것으로 관찰됨). 400이 아닌 다른 상태 코드로 3회 실패하면 예외를 그대로 재전파
- 응답 `<db>` 개수가 `rows`(100) 미만이면 마지막 페이지로 판단해 루프 종료

## 예외 처리 요약

| 상황 | 처리 |
|------|------|
| 목록 조회 3회 실패, `status==400` | 마지막 페이지로 간주, 정상 종료 |
| 목록 조회 3회 실패, 그 외 상태 코드 | 예외 전파 (잡 실패) |
| 상세 조회(`_fetch_detail`) XML 파싱 실패 | 예외 없이 `_DETAIL_FALLBACK` 반환 |
| 상세 병합(`_fetch_and_merge`) 3회 실패 | 예외 없이 폴백 필드로 병합, `visit=None`이므로 이후 필터링에서 자동 제외 |
| `search_concerts`/`collect_by_id` 실패 | 재시도 없이 빈 값/`None` 반환 |
