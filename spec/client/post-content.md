# 게시글 본문 스키마

게시글 `content`(Tiptap JSON)의 웹·앱 공통 계약입니다. 웹 에디터, 앱 블록 에디터, 양쪽 렌더러, BE 검증이 모두 이 문서를 기준으로 합니다. API 요청·응답 형태는 [posts.md](../api/posts.md)를 따릅니다.

---

## 허용 노드

| 노드 | 위치 | content | attrs |
|------|------|---------|-------|
| `doc` | 루트 | 최상위 블록 0개 이상 | — |
| `paragraph` | 최상위, `listItem`·`blockquote` 안 | 인라인 0개 이상 | — |
| `heading` | 최상위만 | 인라인 0개 이상 | `level`: `2` \| `3` |
| `bulletList` | 최상위만 | `listItem` 1개 이상 | — |
| `orderedList` | 최상위만 | `listItem` 1개 이상 | `start`: 정수 (기본 `1`) |
| `listItem` | 목록 안 | `paragraph` 1개 이상 | — |
| `blockquote` | 최상위만 | `paragraph` 1개 이상 | — |
| `entityMentionCard` | 최상위만 | 없음 (atom) | [언급 attrs](#언급-카드) |
| `text` | 인라인 | — | — |
| `hardBreak` | 인라인 | — | — |

- 최상위 블록: `paragraph`, `heading`, `bulletList`, `orderedList`, `blockquote`, `entityMentionCard`
- 중첩 목록은 없습니다. `listItem`·`blockquote` 안에는 `paragraph`만 둡니다
- `heading` level 2는 "큰 소제목", level 3은 "작은 소제목"입니다. 그 밖의 level은 허용하지 않습니다

## 허용 마크

`text` 노드에만 붙습니다.

| 마크 | attrs | 비고 |
|------|-------|------|
| `bold` | — | |
| `strike` | — | |
| `link` | `href` | 자동 링크만. `href`는 `http://`·`https://`로 시작해야 함 |

- 링크 버튼·수동 링크는 없습니다. 입력한 글자가 URL 형식일 때만 링크가 됩니다

## 언급 카드

`entityMentionCard`의 attrs입니다. 값은 `GET /api/mentions/search`의 `EntityCard`에서 채웁니다.

| attr | 타입 | 필수 | 출처 |
|------|------|------|------|
| `entityType` | String | Y | `type` — `CONCERT` \| `ARTIST` \| `RELEASE` \| `TRACK` |
| `entityId` | Long | Y | `id` |
| `title` | String | Y | `title` |
| `subtitle` | String? | N | `subtitle` |
| `thumbnailUrl` | String? | N | `thumbnailUrl` |
| `releaseGroupId` | Long? | N | `releaseGroupId` (`TRACK`만) |

- 고유 카드(`entityType`+`entityId`)는 10개 이하입니다. 같은 엔티티를 여러 번 넣는 것은 1개로 셉니다
- 작성 시 카드 바로 뒤에 빈 `paragraph`를 함께 넣습니다. 카드 뒤에 글을 이어 쓸 커서 위치를 보장하기 위함입니다
- 아티스트 언급도 카드입니다. 인라인 언급(`entityMentionChip`)은 없습니다

## 허용 밖 내용 처리

| 대상 | 처리 |
|------|------|
| 허용 밖 노드·마크, 위치·중첩 규칙 위반, 허용 밖 `heading` level, `http`·`https` 외 링크 | 400 `POST_CONTENT_INVALID_FORMAT` |
| 허용 밖 attrs 키 (예: `orderedList.type`) | BE가 저장 전에 지움 (오류 아님) |

## 입력 규칙

웹·앱 에디터가 같게 동작해야 하는 규칙입니다.

**단축 입력** (줄 맨 앞에서 입력)

| 입력 | 결과 |
|------|------|
| `# ` | 큰 소제목 (`heading` level 2) |
| `## ` | 작은 소제목 (`heading` level 3) |
| `- ` | 글머리 목록 |
| `1. ` | 번호 목록 (`start` = 입력한 숫자) |
| `> ` | 인용 |
| `**글**` | 굵게 |
| `~~글~~` | 취소선 |

**붙여넣기**
- 외부에서 온 내용은 일반 텍스트로 받습니다. 줄바꿈은 문단으로 나뉘고, URL 글자는 자동 링크가 됩니다
- 웹은 같은 에디터에서 복사한 내용의 서식을 유지합니다. 앱은 블록마다 입력칸이 따로라 항상 일반 텍스트로 받습니다
- 어느 경우든 결과는 이 문서의 스키마를 지킵니다

**메뉴 라벨** (웹 슬래시 메뉴·앱 툴바 공통): 큰 소제목 · 작은 소제목 · 글머리 목록 · 번호 목록 · 인용 · 공연 · 아티스트 · 음악 · 트랙

## 표시 규칙

| 대상 | 값 |
|------|----|
| 큰 소제목 | 20 / 700 |
| 작은 소제목 | 17 / 700 |
| 굵게 | 700 |
| 취소선 | 선만 긋고 글자색은 유지 |
| 언급 카드 | 안쪽 여백 12, 모서리 12. 아티스트는 48 원형 사진과 이름(14/600)만. 카드 전체 높이 약 72 (사진 48 + 위아래 여백 12) |

크기 단위는 웹 px, 앱 sp/dp입니다. 본문 크기는 플랫폼마다 다릅니다 (웹 15, 앱 16).

## 길이 제한

| 대상 | 한도 | 오류 |
|------|------|------|
| 본문 글자 수 (아래 계산식) | 10,000자 | 400 `POST_CONTENT_TOO_LONG` |
| `content` JSON 직렬화 길이 | 50,000자 | 400 `POST_CONTENT_TOO_LONG` |

**본문 글자 수 계산식** (BE 기준, 웹·앱 카운터도 같은 식을 씀)
1. 문서를 앞에서부터 돌며 `text` 노드의 글자만 이어 붙입니다. 같은 블록 안의 인접 `text` 노드는 공백 없이 붙고, 마크는 무시합니다
2. `text`가 아닌 노드는 자식을 다 돈 뒤, 그 사이 글자가 하나라도 늘었으면 공백 1자를 붙입니다. 겹친 모든 블록에 각각 적용됩니다 (`bulletList > listItem > paragraph`면 3자, `doc`의 공백은 끝이라 trim됨)
3. `text` 자손이 없는 노드(`hardBreak`, `entityMentionCard`, 빈 `paragraph`)는 0자이고 공백도 붙지 않습니다
4. 마지막에 앞뒤 공백을 지운(trim) 길이를 셉니다. `text` 안의 내부 공백은 남습니다
5. 길이 단위는 UTF-16 코드 유닛입니다 (Java `String.length()`, JS `.length`와 같음). 이모지 😀는 2자입니다
6. 10,000자 **초과**면 거부합니다. 정확히 10,000자는 통과입니다

| # | content | 추출 문자열 | 글자 수 |
|---|---------|-------------|--------|
| N1 | `p "a"` | `a` | 1 |
| N2 | `p "a"`, `p "b"` | `a b` | 3 |
| N3 | `blockquote > p "a"`, `p "b"` | `a␣␣b` | 4 |
| N4 | `bulletList > listItem > p "a"`, `p "b"` | `a␣␣␣b` | 5 |
| N5 | `bulletList > [listItem > p "a", listItem > p "b"]` | `a␣␣b` | 4 |
| N6 | `orderedList > [listItem > p "a", listItem > p "b"]`, `p "c"` | `a␣␣b␣␣␣c` | 8 |
| N7 | `heading(2) "ab"`, `p "c"` | `ab c` | 4 |
| N8 | `p ["a", hardBreak, "b"]` | `ab` | 2 |
| N9 | `p "a"`, `entityMentionCard`, `p "b"` | `a b` | 3 |
| N10 | `entityMentionCard`만 | (빈 문자열) | 0 |
| N11 | `p "a"`, 빈 `p`, `p "b"` | `a b` | 3 |
| N12 | `p ["ab", "c"]` (마크로 나뉜 text) | `abc` | 3 |
| N13 | `p "␣␣a␣␣b␣␣"` | `a␣␣b` | 4 |
| N14 | `p "😀"` | `😀` | 2 |

**경계값** (`"가"×n`은 n자 문자열)

| # | content | 글자 수 | 결과 |
|---|---------|--------|------|
| B1 | `p "가"×9999` | 9,999 | 통과 |
| B2 | `p "가"×10000` | 10,000 | 통과 |
| B3 | `p "가"×10001` | 10,001 | 거부 |
| B4 | `blockquote > p "가"×9997`, `p "나"` | 10,000 | 통과 |
| B5 | `blockquote > p "가"×9998`, `p "나"` | 10,001 | 거부 |
| B6 | `bulletList > listItem > p "가"×9996`, `p "나"` | 10,000 | 통과 |
| B7 | `bulletList > listItem > p "가"×9997`, `p "나"` | 10,001 | 거부 |

기준값은 BE `TiptapTextExtractor`(develop 517fe7d)로 뽑았습니다. 웹·앱·BE 테스트에 같은 케이스를 넣습니다.

## 오류 코드

| 코드 | HTTP | 원인 |
|------|------|------|
| `POST_CONTENT_INVALID_FORMAT` | 400 | [허용 밖 내용 처리](#허용-밖-내용-처리) 참고 |
| `POST_CONTENT_TOO_LONG` | 400 | [길이 제한](#길이-제한) 초과 |
