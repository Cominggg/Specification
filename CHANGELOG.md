$'# Changelog\n\n'## 2026-05-21 (2)

- `spec/admin.md` — ADM-01 비고에 현재 BE 구현 필드 범위 명시 (`mbid`·`name`·`sortName`·`debutDate` 지원, alias·소속사·이미지 미구현)


## 2026-05-21

- `spec/api/concerts.md` — `GET /api/concerts` 쿼리 파라미터 정리 (`dateFrom`, `dateTo`, `artistId`, `region` 제거, `status` 유지)
- `spec/api/concerts.md` — `GET /api/concerts` 기본 정렬 `startDate` 오름차순 → 내림차순으로 변경
- `spec/api/concerts.md` — `status` 요청·응답 값 한국어 → enum 문자열 (`UPCOMING` \| `ONGOING` \| `ENDED` \| `CANCELLED`) 로 수정
- `spec/api/concerts.md` — `GET /api/concerts/{id}` `ticketLinks[].id` 타입 `String` → `Long` 수정
- `spec/api/concerts.md` — `GET /api/concerts/following` `status` 쿼리 파라미터 추가, "예정 공연"에서 전체 공연 조회로 변경 (`status` 미전달 시 전체 반환)

- `spec/api/calendar.md` — `GET /api/calendar` 응답에 `isInCalendar` 필드 추가 (비인증 시 `false`, 인증 시 실제 값)
- `spec/api/calendar.md` — `GET /api/calendar/my` 비고에 `isInCalendar` 항상 `true` 명시
- `spec/api/calendar.md` — `POST /api/calendar/{concertId}` 에러 테이블에 `ALREADY_IN_CALENDAR` (409) 추가
- `spec/api/calendar.md` — `DELETE /api/calendar/{concertId}` 에러 코드 `CONCERT_NOT_FOUND` → `NOT_IN_CALENDAR` (400) 수정

## 2026-05-19

- `spec/api/artists.md` — `POST /api/artists/{id}/follow` 에러 테이블에 `ALREADY_FOLLOWING` (409) 추가
- `spec/api/artists.md` — `DELETE /api/artists/{id}/follow` 에러 테이블에 `NOT_FOLLOWING` (400) 추가

## 2026-05-18

- `spec/api/admin.md` — review-queue 3개 섹션 제거 (`GET /api/admin/review-queue`, `approve`, `reject`) — `matching_review_queue` 테이블 제거(2026-05-07) 및 `concert_artist.approved` 컬럼 제거(2026-05-08)에 따른 반영
- `spec/api/_index.md` — 관리자 엔드포인트 목록에서 review-queue 3개 행 제거
- `spec/features.md` — ADM-02(매칭 검토 큐 관리) 전체 제거; CON-06 "관리자 승인 없이" 문구 삭제
- `spec/pipeline.md` — 매칭 테이블 ② 노출 기준을 "미노출 (confidence=HIGH만 노출)"로 수정; 매칭 테이블 ①③ "관리자 승인 없이" 문구 삭제; `concert_artist.approved` 컬럼 설명 제거

명세서 변경 이력입니다. 날짜 기준으로 관리합니다.

---

## 2026-05-09

- `erd.md` — DB ERD 문서 신규 추가 (DDL 전문, 테이블 관계 요약, 변경 이력)

- `pipeline.md` — 레포 이름 `jpop-concert-collector` → `coming-data`, `requirements.txt` → `pyproject.toml`, `tests/` 추가
- `pipeline.md` — `collectors/wikipedia.py` 추가 — Korean Wikipedia redirect 기반 한국어 alias 수집 단계 신설
- `pipeline.md` — 아티스트 수집 조건에 `type:Group OR type:Person` 필터 추가
- `pipeline.md` — 매칭 로직 전면 수정: ② prfcast 퍼지(token_set_ratio), ③ prfnm 구문 일치(HIGH 폴백) — 기존 ② prfnm partial_ratio에서 변경
- `pipeline.md` — KOPIS 수집 저장 필드에 `poster_url`·`venue_address`·`price` 추가
- `pipeline.md` — 커버아트 갱신 잡 추가 (수요일 05시, 미수집 건만 보완)
- `pipeline.md` — `artist.is_coming` 동기화 섹션 추가
- `pipeline.md` — 관리자 단건 수집 함수(`collect_and_save_*`) 섹션 추가
- `pipeline.md` — 스케줄 실행 시간 명시 (KOPIS 월 03시, 상태갱신 매일 04시, 릴리즈 화 05시 등)
- `features.md` — ART-06: `artist.is_coming` 배지 노출 조건을 HIGH confidence 공연 기준으로 명시, 동기화 주기(월/매일) 추가
- `features.md` — CON-06: 갱신 실행 시간(매일 04시), `is_coming` 연동, 3단계 매칭 기준 명시
- `features.md` — ADM-02: "실패 목록" 문구 제거 → LOW confidence 매칭 공연만 검토 대상
- `features.md` — ADM-03: "수동 등록·수정" → "수동 등록" (수정 엔드포인트 미정의)
- `admin.md` — ADM-02: "자동 매칭 실패 공연 목록" 문구 제거, ADM-03 동일 정정
- `api.md` — `GET /api/admin/review-queue` 비고: "LOW 매칭 + 실패 목록" → "LOW confidence 매칭 목록"
- `features.md` — ADM-07 공연 정보 수정, ADM-08 공연 삭제 기능 신규 추가
- `admin.md` — ADM-07·ADM-08 상세 명세 추가 (수정 가능 필드, cascade 삭제 대상 명시); ADM-04 concert_status_log 명시
- `api.md` — `PUT /api/admin/concerts/{id}` (공연 수정), `DELETE /api/admin/concerts/{id}` (공연 삭제) 추가

## 2026-04-30

- `pipeline.md` — MusicBrainz 아티스트 상세 수집에서 멤버 구성(artist-rels) 관련 내용 제거

## 2026-04-25

- `ux-policy.md` — 홈 섹션 순서 변경: 이달 통계 배너 제거 → 인기 공연을 2번째로 상향, 다가오는 공연을 3번째로 이동
- `ux-policy.md` — 다가오는 공연 섹션에 이달 건수 칩 및 D-day 칩 규칙 추가



## 2026-04-24

- `api.md` — `GET /api/concerts/stats` 신규 추가 — 홈 이달 공연 통계 배너용. year·month 파라미터, `{ "concertCount": N }` 응답
- `api.md` — `GET /api/concerts` 파라미터에 `dateFrom`, `dateTo` 추가 — 홈 다가오는 공연 섹션용 날짜 범위 필터. 기존 `date` 파라미터 하위 호환 유지
- `erd.md` — `artist_member` 테이블 제거 — 그룹 멤버 관계를 DB에서 관리하지 않기로 결정
- `features.md` — **ART-02** 아티스트 상세 조회 설명에서 "멤버 구성" 문구 제거
- `features.md` — **INQ-02** 아티스트 정보 문의 대상 목록에서 "멤버 구성" 제거


## 2026-04-24

- `api.md` — `GET /api/artists` 쿼리 파라미터에서 `genre` 제거 — 수집 대상이 j-pop으로 고정되어 장르 필터 실효성 없음
- `features.md` — **ART-01** 설명에서 "장르 필터링" 문구 제거, 파라미터 초기화 트리거를 "필터 변경"에서 "검색어 변경"으로 수정

## 2026-04-15

- `features.md` — **REL-02** 앨범·싱글·EP 상세 조회 신규 추가 (P2) — MusicBrainz + Last.fm 기반 릴리즈 상세 페이지
- `pipeline.md` — 레포 구조에 `lastfm.py` 추가, 릴리즈 수집에 트랙·크레딧·커버·Last.fm 소개 수집 명세 추가
- `api.md` — `GET /api/releases/{id}` 엔드포인트 신규 추가 (REL-02)

## 2026-04-11 (5)

- `ux-policy.md` — 홈 섹션에 이달 공연 통계 배너·다가오는 공연 섹션 추가 (섹션 순서 재정렬)
- `ux-policy.md` — 에러 처리 테이블에 셋리스트 미구현(CON-05 P2) 기간 처리 정책 추가
- `features.md` — **CON-05** P2 미구현 기간 중 공연완료 시 INQ-03 버튼 노출 정책 추가
- `features.md` — **INQ-03** 진입 경로 2단계(CON-05 구현 전·후) 명세 추가

## 2026-04-11 (4)

- `features.md` — **ART-02** 아티스트 상세에 D-day 카운트다운 및 디스코그래피 섹션(앨범·싱글·EP + 수록곡) 추가
- `features.md` — **REL-01** 아티스트 상세 페이지 디스코그래피 섹션 데이터 공유 명시 / 홈 팔로우 아티스트 우선 명시
- `api.md` — `/api/artists/{id}/releases` 신규 추가 — 디스코그래피 + 수록곡 조회

## 2026-04-11 (3)

- `ux-policy.md` / `features.md` / `api.md` — 목록 표시 방식 전체 표시 → 페이지네이션 전환. 아티스트 목록 24건, 공연 목록 20건, 아티스트 공연 내역·마이페이지 목록 10건/페이지

## 2026-04-11 (2)

- `ux-policy.md` — 홈 히어로 캐러셀 선정 기준 추가 — 예정 공연(캘린더 추가 수 기준, fallback: 조회수) + 최신 릴리즈 자동 선정

## 2026-04-11

- `features.md` — **MY-01** 우선순위 P2 → P1 상향 — INQ-04(내 문의 내역)와 마이페이지 동선 연계로 v1.1 동시 제공 필요
- `features.md` — **CON-05** 우선순위 P1 → P2 하향 — setlist.fm 데이터 공백 리스크 크고 초기 사용 빈도 낮음. INQ-03 문의 채널로 대체 운영
- `features.md` — 마이페이지 상단 프로필 영역 및 탭 구성(관심 아티스트·예정 공연·다녀온 공연·내 문의) 명세 추가
- `features.md` — **MY-01** 기능명 '공연 히스토리' → '다녀온 공연' 변경
- `features.md` — **MY-02** 관심 아티스트 목록 신규 추가 (P1) — ART-04 팔로우 목록 마이페이지 탭으로 명세 독립
- `features.md` — **MY-03** 예정 공연 목록 신규 추가 (P1) — CAL-02 내 캘린더 중 미래 공연 탭

## 2026-04-10

- `features.md` — **REL-01** 새 앨범·싱글 조회 기능 추가 (P1) — 홈 화면 수평 스크롤, ALBUM/SINGLE 타입 배지
- `features.md` — 검색 기능(SRC-01, SRC-02) 및 `/search` 라우트 제거 — 각 페이지 내 검색으로 대체
- `features.md` — **CON-07** 티켓팅 임박 공연 제거 — 티켓 오픈일 데이터 취득 경로 없음
- `pipeline.md` — MusicBrainz 아티스트 수집에 `artist-rels` inc 파라미터 추가 — 멤버 구성(전·현 멤버) 수집 반영
- `pipeline.md` — 릴리즈 수집 단계 신규 추가 — 초기 구축: `release.py` 컬렉터, 주기적 수집: 주 1회 신보 감지 및 INSERT


## 2026-04-08

- `features.md` — **ART-01** 목록 필터에서 데뷔년도 제거 (이름·장르만 유지)
- `features.md` — **ART-02** 상세 프로필에서 소속사 제거 (멤버 구성·데뷔일만 유지)
- `features.md` — **ART-03** 공연 내역 탭을 `과거·예정` → `전체·예정·과거` 3탭으로 변경

## 2026-04-06

- 프론트엔드 TypeScript → **JavaScript (JSX)** 확정
- 상태관리 **Zustand** 확정 (전역: 인증·필터 / 서버: React Query)
- **CON-07** 티켓팅 임박 공연 신규 추가
- 반응형 브레이크포인트 2단계 → 3단계 세분화 (`1279px` / `900px` / `767px`)
- 목록 표시 방식: 무한 스크롤 → 전체 표시
- 홈 섹션 구성 순서·원칙 명세 추가
- 인기 공연 홈 노출: 10건 → **4건 (1행)**

## 2026-04-03 (2)

- OAuth Naver 제거 (Google·Kakao만 유지)
- **INQ-01·INQ-02** 데이터 문의 신규 추가
- **ADM-06** 문의 목록 조회 및 처리 신규 추가
- 문의 관련 API 엔드포인트 추가
- SRC-02 우선순위 P2 → P1 상향
- ART-05 운영 리소스 경고 추가

## 2026-04-03 (1)

- setlist.py 추가 / 파이프라인 ③ setlist.fm 수집 단계 신규 추가
- MusicBrainz·KOPIS 수집 API URL 상세 보강
- Bandsintown 제거 확정

## 2026-03-31 (2)

- 에러 처리 섹션 신규
- 인증·토큰 정책 섹션 신규
- 공통 UI/UX 정책 섹션 신규
- 관리자 기능 명세 섹션 신규
- AUTH-02 이미지 업로드 방식 명세
- ART-03 페이지네이션 방식 명세
- ART-06 배지 조건 구체화
- CON-02 prfstate 값 열거 및 배지 색상 정의
- CAL-01 멀티데이 처리 방식 추가
- SRC-01·02 empty state 및 정렬 기준 추가
- API 명세 `/api/auth/refresh`·관리자 엔드포인트 추가

## 2026-03-31 (1)

- AUTH-01 수정
- ART-02·03 수정
- CON-02 수정
- CAL-03·04 삭제
- **ART-06·CON-06·MY-01** 신규 추가
- 리스크 항목 보강

## 2026-03-31 — 최초 작성