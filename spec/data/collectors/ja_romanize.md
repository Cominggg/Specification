# 로마자→한글 alias 자동 변환 (collectors/ja_romanize.py)

외부 API를 전혀 호출하지 않는 유일한 수집 모듈. DB에 이미 저장된 MusicBrainz `sort_name`(로마자 표기)을 규칙 기반으로 한글로 변환해 `artist_alias(locale='ko')`를 보완한다. 과거 Korean Wikipedia redirect 기반 alias 수집(`wikipedia.py`)을 대체한 잡이다.

## 배경: 왜 규칙 변환인가

MusicBrainz는 일본어 아티스트의 `sort_name`을 헵번식 로마자로 제공한다(예: `"Yonezu, Kenshi"`). 이를 일본어 외래어 표기법(국립국어원 규정)에 따라 결정적으로 한글 변환할 수 있어, 외부 API 호출이나 사전 매핑 없이 순수 규칙만으로 처리 가능하다.

## 함수별 로직

### `_is_japanese_script(text: str) -> bool`

문자열에 히라가나(`0x3040-0x309F`)·가타카나(`0x30A0-0x30FF`)·한자(`0x4E00-0x9FFF`) 중 하나라도 포함되면 `True`. `artist.name`이 이미 로마자 표기인 그룹(예: 영문 밴드명)은 변환 대상에서 제외하기 위한 판별.

### `_add_batchim(syllable: str, jong: int) -> str`

완성형 한글 음절(1글자)에 종성(받침)을 추가한다. KS X 1001 완성형 공식(`code = (초성*21+중성)*28+종성`, `AC00` 오프셋)을 이용해 종성이 이미 있는 음절(`code % 28 != 0`이 아닌 경우)이나 한글이 아닌 문자는 그대로 반환한다.

- `_JONG_N = 4` (받침 ㄴ) — 발음(ん) 표기용
- `_JONG_S = 19` (받침 ㅅ) — 촉음(っ) 표기용

### `_match_mora(word: str, pos: int) -> Optional[tuple]`

`word[pos:]`에서 3자·2자·1자 순으로 가장 긴 모라(음절 단위)를 `_ALTERNATING`/`_FIXED` 테이블에서 찾아 `(매칭된 키, 소비 길이)`를 반환. 최장 일치 우선이라 `"cha"`가 `"c"+"h"+"a"`로 잘못 쪼개지지 않는다.

### `_romanize_stream(text: str) -> Optional[str]`

전체 변환의 핵심 상태 머신. 공백으로 구분된 로마자 문자열을 앞에서부터 순회하며 아래 규칙을 적용한다.

1. **숫자**: 그룹명에 붙는 `46`, `96` 같은 숫자는 변환 없이 그대로 통과
2. **촉음(っ)**: `_SOKUON_PREFIXES = ("kk","ss","tt","pp","cch","tch")`로 시작하면, 직전 음절에 `_add_batchim(..., _JONG_S)`로 받침 ㅅ을 붙이고 중복 자음 중 1글자만 소비 (예: `"kekka"` 표기 규칙과 유사한 촉음 처리). 직전 음절이 없거나 공백이면 변환 불가(`None`)
3. **발음(ん)**: 단독 `n` 뒤에 모음/야행(`aiueoy`)이 오지 않으면 받침 비음으로 처리(`_add_batchim(..., _JONG_N)`). 뒤에 모음이 이어지면 일반 모라 매칭으로 넘어감
4. **청음 어두/어중 교체**(`_ALTERNATING`): `ka/ki/ku/ke/ko`, `kya/kyu/kyo`, `ta/te/to`, `chi`, `cha/chu/cho` 계열은 **단어 첫머리(`word_initial=True`)일 때 예사소리**(가·기·구...), **그 외 위치는 거센소리**(카·키·쿠...)로 표기 — 외래어 표기법 세칙(어두 무성 파열음/파찰음 규칙)
5. 그 외 모든 모라는 `_FIXED` 테이블 고정 표기(탁음·비음·유음·마찰음 등 위치 무관)
6. **공백을 넘어도 "어중"으로 유지** — `"Fujii Kaze"`처럼 성-이름이 공백으로 나뉘어도 두 번째 단어 첫 글자는 어두 규칙이 아니라 어중(거센소리) 규칙 적용 (`word_initial`은 공백에서 리셋되지 않음) → `"후지이 카제"`
7. 매칭 테이블에 없는 문자를 만나면 **그 시점에서 전체를 포기하고 `None` 반환** — 부분적으로 틀린 표기를 저장하지 않기 위한 전량 실패(all-or-nothing) 정책

### `romanize_to_korean(sort_name: str) -> Optional[str]`

전처리 후 `_romanize_stream()` 호출.

- `"성, 이름"`(MusicBrainz `sort_name` 관용 포맷)의 콤마는 제거만 하고 순서는 그대로 유지(성-이름 순) — 이름 재배열은 하지 않음
- 하이픈(`-`)은 단어 구분이 아니라 모라 경계 표시(예: `"Ko-en"`)로 보고 공백이 아닌 **삭제**로 처리
- `_NON_ROMAJI_RE`(`[^a-z0-9\s]`)로 로마자·숫자·공백 외 문자를 공백 치환 후 연속 공백을 정리
- 변환 실패(인식 불가 문자) 시 `logger.debug`로만 기록(경고 수준 아님 — 흔한 케이스이므로)

### `collect_ko_aliases(artists: list[dict]) -> list[dict]`

```
입력: [{"artist_id": int, "name": str, "sort_name": str}, ...]
출력: [{"artist_id": int, "name": str, "locale": "ko"}, ...]
```

`_is_japanese_script(name)`이 `False`인 아티스트(이름이 이미 로마자 표기)는 애초에 변환 대상에서 제외 — 원본 로마자 이름 자체가 매칭에 활용 가능하다고 보기 때문. 변환 성공 건만 결과에 포함하며, 처리 결과를 `logger.info("%d / %d건")`으로 요약 로깅한다.

## 예외 처리

외부 API가 없어 네트워크 예외 처리가 존재하지 않는다. 유일한 실패 모드는 "규칙 테이블에 없는 문자"이며, 이 경우 예외를 던지지 않고 함수가 `None`을 반환하는 방식으로 처리한다(호출부는 `if converted:` 체크만 하면 됨). 잘못된 한글 alias가 저장되는 것을 막기 위해 **부분 변환 결과를 저장하지 않는다**는 것이 이 모듈의 핵심 안전장치다.
