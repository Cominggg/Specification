## 2026-09-27

- `spec/policy/auth-policy.md` — "다중 기기 로그인 정책" 섹션 신설: 동시 로그인 허용, 사용자당 최대 세션 5개(초과 시 가장 오래된 세션 제거), Refresh Token 회전 시 세션 식별자 유지·원자적 검증·교체(유예 구간 없음), 로그아웃은 현재 기기 세션만 종료, 탈퇴 시 전체 세션 삭제, 전체 기기 로그아웃 미제공. 로그아웃 행의 "서버 Refresh Token 블랙리스트 등록"을 실제 동작(Access Token 블랙리스트 + Refresh Token 삭제)으로 정정. 기존 구현이 사용자당 Refresh Token 1개만 저장해 새 기기 로그인 시 다른 기기가 로그아웃되는 문제를 FE 조사 중 발견해 정책을 확정함 (BE 반영 예정)
- `spec/api/auth.md` — `POST /api/auth/refresh`에 회전·세션 단위 검증·동시 요청 처리 비고 추가, `POST /api/auth/logout`을 현재 기기 세션만 종료하도록 정정(Cookie 없어도 200), `DELETE /api/auth/withdraw`에 전체 세션 삭제 명시

## 2026-09-21

- `spec/api/concerts.md` — 공연 별점 등록·수정(`PUT /api/concerts/{id}/rating`), 내 별점 조회(`GET /api/concerts/{id}/rating/me`), 별점 취소(`DELETE /api/concerts/{id}/rating`) 엔드포인트 신규 문서화. `CONCERT_NOT_ENDED`(400, 공연 상태가 ENDED가 아니면 등록·수정 거부) 에러 반영. `GET /api/concerts`·`GET /api/concerts/{id}` 응답에 `averageRating`·`ratingCount` 필드 추가
- `spec/api/releases.md` — 릴리즈 별점 등록·수정(`PUT /api/releases/{id}/rating`), 내 별점 조회(`GET /api/releases/{id}/rating/me`), 별점 취소(`DELETE /api/releases/{id}/rating`) 엔드포인트 신규 문서화 (상태 제약 없이 항상 등록 가능). `GET /api/releases`·`GET /api/releases/{id}` 응답에 `averageRating`·`ratingCount` 필드 추가
- `spec/api/_index.md` — 위 공연·릴리즈 별점 엔드포인트 6건을 전체 목록에 추가
- `spec/features.md` — CON-10(공연 별점 평가), REL-05(음악 별점 평가) 신규 추가 (모두 P1)

## 2026-09-19

- `spec/api/posts.md` — 인기글 카테고리(`GET /api/posts/popular-board`), 게시글·댓글 신고(`POST /api/reports`), 공지사항 공개 조회(`GET /api/notices`, `GET /api/notices/{id}`) 엔드포인트 신규 문서화
- `spec/api/admin.md` — 관리자 공지사항 CRUD(`GET/POST /api/admin/notices`, `GET/PATCH/DELETE /api/admin/notices/{id}`), 관리자 신고 처리(`GET /api/admin/reports`, `GET /api/admin/reports/{id}`, `PATCH /api/admin/reports/{id}/status`) 엔드포인트 신규 문서화. `deleteTarget=true` 시 신고 대상 게시글(물리 삭제)·댓글(소프트 삭제) 강제 삭제 동작 명시
- `spec/api/_index.md` — 위 8개 엔드포인트 행 추가 (게시판 섹션 3건, 관리자 섹션 8건 — 신고 생성 포함)
- `spec/features.md` — 게시판 섹션 상단 "신고 기능은 없다" 문구를 "게시글·댓글 신고 기능을 포함한다"로 정정. POST-10(인기글 카테고리)·POST-11(게시글·댓글 신고)·POST-12(공지사항 노출) 신규 추가 (모두 P1)
- `spec/admin/features.md` — ADM-12(공지사항 관리)·ADM-13(신고 목록 조회 및 처리) 신규 추가 (P1). 유저 계정 정지는 이번 기능 범위 밖(ADM-05로 유지, 별도 이슈)
- `spec/api/admin.md` — FE에서 신고자 식별 불가 문제 보고에 따라 `GET /api/admin/reports`·`GET /api/admin/reports/{id}` 응답에 `reporterNickname` 필드 추가

## 2026-09-18

- `spec/legal/terms-of-service.md` — 커뮤니티(게시글·댓글 작성/추천/신고), 공연·릴리스 평점 기능 도입 반영. 제2조에 "게시물"·"평점" 정의 추가, 제6조 이용자 의무에 명예훼손·불법정보 게시·허위 신고·평점 조작 금지 항목 추가, 제7조에 게시물 조치와 계정 제재 병과 근거 추가, 제8조(게시물의 관리 — UGC 저작권 귀속 및 이용허락 범위)·제9조(신고 및 처리) 신설, 제11조(저작권)·제12조(면책 조항)에 게시물·평점 관련 내용 반영. 부칙으로 개정 사유·시행일(공지일로부터 30일 후인 2026-10-18)·시행 전 임시 배포 시 적용 예외 명시
- `spec/legal/privacy-policy.md` — 위 기능 도입에 따라 수집 방법(제1조)에 커뮤니티·평점 관련 정보 처리 근거 추가, 제2조 3호에 "신고의 접수·처리" 목적 추가, 제3조 보유기간에 신고 접수·처리 기록(처리 완료일로부터 1년) 예외 항목 추가. 부칙으로 개정 사유·시행일(2026-10-18) 명시
- `spec/api/admin.md` — 정책 버전 등록 엔드포인트(`POST /api/admin/policies`) 신규 문서화. 등록 커밋 후 정책 알림 배치 자동 실행, `POLICY_VERSION_DUPLICATE`(409) 에러, 회원가입 시 시행 중인 정책 미등록 상태면 `POLICY_NOT_FOUND`로 실패하는 운영 제약 반영
- `spec/api/_index.md` — 전체 엔드포인트 목록의 관리자 섹션에 `POST /api/admin/policies` 행 추가
- `spec/admin/features.md` — ADM-11 "정책 변경 이메일 고지" 신규 추가 (P1). 정책 등록 → 배치 트리거 → 메일 발송·재시도, 회원가입 시 동의 이력 기록까지의 파이프라인 요약
- `spec/api/admin.md` — `POST /api/admin/policies`의 `requiresReconsent` 필드 제거. 기존 회원에 대한 정책 변경 고지는 하드 게이트(로그인 시 재동의 강제) 없이 `terms-of-service.md` 제3조 3항의 묵시적 동의 원칙만으로 처리하기로 확정 — 소비 로직 없는 필드였으며, 능동적 동의가 실제로 필요한 개정이 생기기 전까지는 재동의 플로우를 미구현 상태로 유지
- `spec/legal/privacy-policy.md` — 부칙에 적용례(3항) 신설. 신고·평점 기능은 아직 미구현 상태로, 해당 기능 관련 개인정보 처리 목적(제2조 3호)·보유기간(제3조)은 기능이 실제로 제공되는 시점부터 적용됨을 명시. 시행일(2026-10-18) 이전에 기능이 조기 제공될 경우 종전 방침의 일반 목적·보유기간을 따르도록 하여, `terms-of-service.md` 부칙 3항과 동일하게 법적 문서가 미구현 기능을 앞서 규율하는 공백을 보완

## 2026-09-15

- `spec/api/posts.md` — 게시판 API 전체를 실제 구현(main 병합 완료) 기준으로 갱신: `[예정]`·설계 확정 배너 제거. `GET /api/mentions/search`에 `type=TRACK`(앨범 내 개별 트랙 멘션) 추가, `page` 파라미터 도입·응답을 페이지네이션 형태로 변경(`limit` 기본값 10→20), 응답 카드에 `releaseGroupId`(TRACK 전용) 필드 추가. `GET /api/posts/{id}`·`GET /api/posts` 등 응답의 `entityTags`(`EntityTag`)에도 `releaseGroupId` 반영. `GET /api/posts/popular`·`GET /api/posts/trending-tags` 신규 엔드포인트 문서화. 답글 있는 댓글 삭제 정책을 소프트 삭제로 확정 반영(비고의 "BE 설계 시 확정 필요" 문구 제거). `title`(255자)·`entityTags`(10개)·댓글(500자) 길이 제한, 목록류 `size` 최대 100, 검색어 최소 2자, `content` JSON 50000자 제한 등 누락됐던 검증 규칙 보강. `GET /api/entities/{type}/{id}/posts`가 엔티티 존재 여부를 검증하지 않는다는(404 없음) 실제 동작 정정
- `spec/features.md` — 게시판 섹션 설계 확정 배너 제거. POST-02(엔티티 인라인 멘션)에 TRACK 타입 분리(2026-09-15) 반영, POST-04·POST-05에 트랙 포함 명시, POST-07(댓글) 삭제 정책·API 경로 확정 반영, POST-08(인기 게시글)·POST-09(트렌딩 태그) 신규 추가
- `spec/api/_index.md` — 게시판 섹션 설계 확정 배너 제거, `GET /api/posts/popular`·`GET /api/posts/trending-tags` 엔드포인트 행 추가

## 2026-09-11

- `spec/features.md` — POST-07 "게시글 댓글" 신규 추가 (P1). 게시판 섹션 상단의 "댓글·신고 기능은 없다" 문구를 "신고 기능은 없다(댓글은 POST-07 참고)"로 정정 — 게시글 상세 페이지 리디자인 논의 중 댓글 기능을 이 게시판(POST-01~06)에 포함하기로 확정
- `spec/api/posts.md` — 댓글 API 5종 신규 추가: 목록 조회(`GET /api/posts/{id}/comments`, 답글 1단계 중첩), 작성(`POST /api/posts/{id}/comments`), 삭제(`DELETE /api/comments/{commentId}`), 좋아요(`POST`/`DELETE /api/comments/{commentId}/like`). `GET /api/posts/{id}` 응답에 `commentCount` 필드 추가. 답글 있는 댓글의 삭제 처리 방식(하드/소프트)은 비고에 BE 협의 필요 항목으로 명시
- `spec/api/_index.md` — 전체 엔드포인트 목록의 게시판 섹션에 댓글 API 5종 추가

## 2026-09-10

- `spec/api/posts.md` — 신규 추가: 게시판(Post) API 9종 명세 (⚠️ 설계 확정, 구현 예정 — 이슈 #116, 브랜치 `feat/#116-post-board`). 멘션 자동완성(`GET /api/mentions/search`), 게시글 CRUD(`/api/posts`), 추천(`/api/posts/{id}/recommend`), 엔티티별 백링크(`GET /api/entities/{type}/{id}/posts`), 통합 검색(`GET /api/search`)
- `spec/api/_index.md` — 도메인별 문서 표·전체 엔드포인트 목록에 "게시판" 섹션 추가 (posts.md 링크)
- `spec/features.md` — POST-01~06 게시판 기능 명세 신규 추가 (P1~P2); ART-05(아티스트 게시판) 비고에 신규 게시판과의 관계 명시
- `spec/api/posts.md` — FE 리뷰 반영: `GET /api/posts` 목록 응답(`PostSummary`)에 `viewCount` 추가(백링크·통합검색에도 동일 적용); `GET /api/posts/{id}` 응답에 `isAuthor: Boolean` 추가(닉네임 문자열 비교 대신 본인 여부 판별용); `PATCH /api/posts/{id}`에 `category` 수정 지원 추가(선택 필드, 검증은 최종 category 기준); `POST`/`DELETE /api/posts/{id}/recommend` 응답에 `recommendCount` 포함

## 2026-09-06

- `spec/api/concerts.md` — `GET /api/concerts`에 `q`·`followedOnly`·`ticketOpenPending` 파라미터 추가, `sort` 허용 필드(`startDate`·`ticketOpenAt`) 명시 및 위반 시 400 반영; `GET /api/concerts/search`·`GET /api/concerts/following`는 위 엔드포인트로 통합되어 제거; `inCalendar`·`status` 동시 사용 시 우선순위가 있다는 기존 오기재를 AND 조합으로 정정
- `spec/api/concerts.md` — `GET /api/concerts`, `GET /api/concerts/{id}` 응답의 아티스트 필드를 `artistName`(단일 문자열, `confidence` 기준)에서 `artists`(배열, `{artistId, name, koreanName}[]`)로 정정 — `confidence` 컬럼은 이미 제거되어 다중 아티스트 배열 구조로 전환된 상태였음
- `spec/api/releases.md` — `GET /api/releases`에 `q` 파라미터 추가 및 `type`을 `Album`·`Single`만 허용하도록 변경(그 외 값은 400); `GET /api/releases/search`는 위 엔드포인트로 통합되어 제거
- `spec/api/artists.md` — `GET /api/artists`에 `sort` 파라미터 추가(`sortName`·`followerCount`, 동시 지정 시 400), 응답에 `followerCount` 필드 추가
- `spec/features.md` — ART-01 정렬 옵션 추가; CON-01 통합 필터 및 기본 정렬 방향(내림차순) 오기재 정정; CON-08·REL-04 검색 API 경로를 통합 엔드포인트로 갱신; REL-03 `type` 필터를 `Album`·`Single`로 정정, 해소된 "BE 경로 충돌" 비고 제거

## 2026-08-11

- `spec/legal/terms-of-service.md` — 브랜드 표기 "Coming" → "커밍" 전환 (제1조·제2조, 프론트엔드 실제 약관 텍스트와 동기화)
- `spec/legal/privacy-policy.md` — 브랜드 표기 "Coming" → "커밍" 전환 (도입부, 프론트엔드 실제 방침 텍스트와 동기화)
- `spec/overview.md` — 프로젝트 소개 문구를 "**커밍**(Coming)"으로 갱신 (공식 표시 브랜드명 변경 반영, 기술 레포지토리명은 변경 없음)

## 2026-07-04

- `spec/api/admin.md` — `POST /api/admin/concerts/{id}/candidates`, `DELETE /api/admin/concerts/{id}/candidates/{artistId}` 신규 추가 (PENDING 공연 후보 아티스트 관리); `POST /api/admin/concerts/{id}/artists`, `DELETE /api/admin/concerts/{id}/artists/{artistId}`에 `CONCERT_IS_PENDING(400)` 에러 추가 및 PENDING 공연 사용 불가 비고 반영
- `spec/admin/features.md` — ADM-02 비고 갱신: 후보 아티스트 보완 API를 `/candidates` 엔드포인트로 변경

## 2026-07-03

- `spec/data/pipeline.md` — 함수별 로직·예외 처리·외부 API 상세를 하위 문서로 분리한 인덱스로 재구성; 릴리즈 수집을 MusicBrainz+Cover Art Archive에서 Spotify 단독 방식으로, Wikipedia alias 수집을 로마자→한글 변환(`ja_romanize`)으로 코드 실체에 맞게 갱신; `prfcast` 기반 매칭 단계 폐기(현재는 title 구문 매칭 단일 전략) 반영
- `spec/data/scheduler.md` — 신규 추가: 잡별 cron 시각·진입점·의존관계, 체크포인트·복구 파일, CLI 커맨드, 내부 API(`api.py`) 동시 실행 가드 명세
- `spec/data/matchers.md` — 신규 추가: `has_match`·`match_concert`·`_phrase_match_title` 함수별 로직 명세
- `spec/data/error-handling.md` — 신규 추가: 재시도/백오프 정책, Spotify 429 밴 처리, 체크포인트 기반 재개, 로깅 규칙 명세
- `spec/data/collectors/musicbrainz.md` — 신규 추가: MusicBrainz 아티스트 수집 함수·API 요청/응답 상세 (Last.fm 인기도 필터는 코드에서 제거되어 명세에 미포함)
- `spec/data/collectors/release.md` — 신규 추가: Spotify 릴리즈 수집 함수·API 상세
- `spec/data/collectors/kopis.md` — 신규 추가: KOPIS 공연 수집 함수·API 상세
- `spec/data/collectors/setlist.md` — 신규 추가: setlist.fm 수집 함수·API 상세
- `spec/data/collectors/ja_romanize.md` — 신규 추가: 로마자→한글 alias 변환 규칙·함수 로직 상세
- `spec/data/collectors/artist_image.md` — 신규 추가: Spotify 이미지 수집·Client Credentials 인증 함수 상세

## 2026-07-03

- `spec/api/pipeline.md` — `POST /api/admin/data/collect/artists`, `POST /api/admin/data/collect/concerts`, `POST /api/admin/data/collect/concerts/{id}/setlist` 동기 처리로 전환: 응답 바디 없음 → 수집 결과 DTO 반환; `PIPELINE_NOT_FOUND`(404)·`PIPELINE_CONFLICT`(409) 에러 추가
- `spec/data/pipeline.md` — ⑤ 관리자 수집 함수 동기/비동기 분리: 아티스트·공연·셋리스트 수집은 동기(결과 반환), 릴리즈·커버아트는 비동기 유지

## 2026-07-01

- `spec/api/auth.md` — `PATCH /api/auth/me/marketing` 응답을 `200 OK`에서 `204 No Content`로 수정

## 2026-07-01

- `spec/api/auth.md` — `GET /api/auth/me` 응답에 `agreedMarketing: Boolean` 추가; `PATCH /api/auth/me/marketing` 엔드포인트 신규 추가
- `spec/api/admin.md` — `GET /api/admin/artists/{id}` 신규 추가 (alias 포함 단건 조회); `PUT /api/admin/artists/{id}` 요청 바디에 `aliases` 필드 추가·`debutDate` 제거
- `spec/admin.md` — ADM-01 비고 갱신: alias 편집(`GET/PUT /api/admin/artists/{id}`) 구현 완료 반영

## 2026-07-01

- `spec/api/releases.md` — `GET /api/releases`, `GET /api/releases/search` content[] 응답에 `spotifyId: String?` 추가 (`release_group.spotify_id`); `GET /api/releases/{id}` 최상위 및 `tracks[]`에 각각 `spotifyId` 추가 (`track.spotify_id`)
- `spec/api/artists.md` — `GET /api/artists/{id}/releases` content[] 릴리즈 레벨 및 `tracks[]`에 `spotifyId: String?` 추가

## 2026-06-30

- `spec/api/artists.md` — `GET /api/artists` 목록 응답에 `spotifyUrl: String?` 필드 추가 (Spotify URL 없으면 `null`)
- `spec/api/my.md` — `GET /api/me/inquiries/exists` 엔드포인트 추가: 문의 제출 전 PENDING 중복 여부 사전 확인
- `spec/api/concerts.md` — `GET /api/concerts/{id}/setlist` 응답에 `sourceUrl: String?` 필드 추가 (setlist.fm attribution URL); 비고 섹션 추가 (null 시 `https://www.setlist.fm/` 폴백)
- `spec/erd.md` — `setlist.attribution_url(text)` 컬럼 추가 (V21 마이그레이션)
- `spec/erd.md` — `concert_artist_candidate.matched_by` 컬럼 제거 (V22 마이그레이션)

## 2026-06-27

- `spec/api/auth.md` — 콜백 응답을 리다이렉트(`isNewUser` 쿼리 파라미터) 방식으로 갱신; `GET /api/auth/me` 응답 `avatarUrl` → `birthYear` 교체·`PENDING` 역할 추가; `PUT /api/auth/me` multipart → Query Parameter 방식으로 변경; `POST /api/auth/register`(회원가입 완료), `GET /api/auth/check-nickname`(닉네임 중복 검사) 엔드포인트 신규 추가
- `spec/auth-policy.md` — PENDING 역할 정책 추가: 신규 가입·탈퇴 후 재가입·SUSPENDED 차단 처리 흐름 명세
- `spec/features.md` — AUTH-01 PENDING 역할 분기·SUSPENDED 차단·재가입 플로우 반영; AUTH-02 프로필 이미지 업로드 제거; AUTH-04 회원가입 완료(온보딩) 신규 추가 (P0)

## 2026-06-25

- `spec/api/my.md` — history `artistName` 설명에서 `confidence=HIGH` 기준 제거 (V15 마이그레이션으로 `concert_artist.confidence` 컬럼 삭제됨)
- `spec/features.md` — ART-06 배지 노출 조건, CON-06 매칭 기준에서 confidence 관련 설명 제거

## 2026-06-24

- `spec/api/artists.md` — `GET /api/artists`에 `isComing`·`following` 쿼리 파라미터 추가; 세 파라미터 조합 필터 지원 비고 반영
- `spec/api/my.md` — 마이페이지 API 전체 경로 `/me/*`로 변경: `GET /api/me/concerts/upcoming`(신규), `GET /api/me/concerts/history`, `GET /api/me/inquiries`, `GET /api/me/inquiries/{id}`
- `spec/api/calendar.md` — `GET /api/calendar/my` 섹션 제거 (`/api/me/concerts/upcoming`으로 이관)
- `spec/features.md` — MY-01 필터 기준 `endDate < 오늘`로 명시, MY-03 API 경로·ONGOING 포함 설명 갱신

## 2026-06-18

- `spec/api/admin.md` — `GET /admin/concerts/pending` 응답에 `ticketOpenAt`·`bookingLinks` 필드 추가
- `spec/api/admin.md` — `PUT /admin/concerts/{id}/approve` optional request body (`ticketOpenAt`, `bookingLinks`) 명세 추가
- `spec/admin.md` — ADM-02 PENDING 목록 응답 필드 및 승인 동시 설정 동작 명시

## 2026-06-17

- `spec/api/admin.md` — `PUT /api/admin/concerts/{id}` `ticketOpenAt` null 전달 시 기존 값 초기화(null-reset) 동작 명세 추가
- `spec/admin.md` — ADM-07 수정 가능 필드에 `ticketOpenAt` 추가 및 null-reset 동작 명시

## 2026-06-17

- `spec/api/concerts.md` — `ticketOpenAt` 필드 전체 응답에 추가; `GET /api/concerts/ticketing` 엔드포인트 신규 추가 (티켓 오픈 예정 공연 조회, `following` 파라미터 지원)
- `spec/api/calendar.md` — `CalendarEntry`에 `type`(`CONCERT`|`TICKETING`) 및 `ticketOpenAt` 필드 추가; 캘린더 응답에 티켓팅 일정 통합 설명 반영
- `spec/api/admin.md` — `GET /api/admin/concerts/{id}` 응답 및 `PUT /api/admin/concerts/{id}` 요청 바디에 `ticketOpenAt` 필드 추가
- `spec/features.md` — CON-09 티켓 오픈 예정 공연 조회 신규 추가; CAL-01 티켓팅 일정 통합 반영; ADM-07 `ticketOpenAt` 수정 지원 반영

## 2026-06-16

- `spec/api/releases.md` — `GET /api/releases/search` `q` 선택 파라미터로 변경; `type`(`Album`·`Single` 허용, 그 외 400), `following`(팔로우 아티스트 필터) 파라미터 추가; `following=true` 시 Bearer 토큰 필요 비고 추가
- `spec/api/_index.md` — 전체 엔드포인트 목록에 `GET /api/concerts/search`, `GET /api/releases/search` 추가
- `spec/features.md` — REL-04 `q` 선택 파라미터로 변경, `type`·`following` 조합 지원 명세 반영

## 2026-06-15
- `spec/api/releases.md` — `GET /api/releases/search` 엔드포인트 신규 등록 (릴리즈명·트랙명·아티스트명·alias 검색, `q` 필수)
- `spec/features.md` — REL-04 음악 검색 기능 신규 추가 (P1)

- `spec/api/concerts.md` — `GET /api/concerts`에 `inCalendar` 파라미터 추가 및 `status` 우선순위 비고 반영; `GET /api/concerts/search` 엔드포인트 신규 등록 (공연명·아티스트명·alias LIKE 검색, `q` 필수)
- `spec/api/releases.md` — `GET /api/releases`에 `following` 파라미터 추가 (팔로우 아티스트 필터); 정렬 설명 `releaseDate DESC NULLS LAST` 갱신
- `spec/features.md` — CON-01 내 캘린더 필터(`inCalendar=true`) 추가; CON-08 공연 검색 기능 신규 추가 (P1); REL-03 `following=true` 필터 및 NULLS LAST 정렬 명세 반영
- `spec/api/concerts.md` — `GET /api/concerts/search`에 `status` 파라미터 추가 (상태별 필터 지원)
- `spec/api/releases.md` — `GET /api/releases` `following=true` 비고 수정: `type` 필터 동시 적용 가능
- `spec/features.md` — REL-03 `following=true` 동작 명세 수정: `type` 필터 병행 적용 허용

## 2026-06-12

- `spec/api/admin.md` — `GET /api/admin/concerts/{id}` (EXCLUDED 포함 단건 조회), `DELETE /api/admin/concerts/{id}/artists/{artistId}` (아티스트 매핑 제거) 엔드포인트 신규 추가
- `spec/admin.md` — ADM-10 EXCLUDED 공연 관리 흐름 갱신: 공연 수정 폼 단건 조회 및 아티스트 제거 단계 추가

- `spec/api/admin.md` — `GET /api/admin/concerts/excluded` · `GET /api/admin/artists` 엔드포인트 신규 추가; `PUT /api/admin/concerts/{id}/state` status 허용값에 `EXCLUDED` 추가 및 비고 갱신; `POST /api/admin/concerts/{id}/artists` 비고에 EXCLUDED 공연 적용 가능 명시
- `spec/admin.md` — ADM-01 비고: DB 아티스트 검색 API 추가 반영; ADM-04 비고: 복원 시 `is_coming` 자동 갱신 명시; ADM-10 EXCLUDED 공연 관리 기능 신규 추가

## 2026-06-11

- `spec/api/admin.md` — `GET /api/admin/data/search/artists` · `GET /api/admin/data/search/concerts` 응답에 `url` 필드 추가
- `spec/api/pipeline.md` — `GET /api/admin/data/search/artists` · `GET /api/admin/data/search/concerts` 응답에 `url` 필드 추가

## 2026-06-08

- `spec/api/admin.md` — `POST /api/admin/artists`(수동 등록), `POST /api/admin/concerts`(수동 등록), `DELETE /api/admin/concerts/{id}`(삭제) 제거; `POST /api/admin/concerts/{id}/artists` 설명 수정 (`concert_artist_candidate` → `concert_artist` 직접 저장); Data 파이프라인 엔드포인트 전체 `spec/api/pipeline.md`로 분리
- `spec/api/pipeline.md` — **신규** BE↔Data 파이프라인 연동 API 명세 파일. 검색 2종(`GET /api/admin/data/search/artists`, `GET /api/admin/data/search/concerts`), 수집 트리거 4종(`POST /api/admin/data/collect/artists`, `POST /api/admin/data/collect/concerts`, `.../artists/{id}/releases`, `.../concerts/{id}/setlist`) 포함
- `spec/api/_index.md` — 관리자 엔드포인트 목록 정리(수동 등록·삭제 제거, pending/approve/reject/artists 추가); Data 파이프라인 연동 섹션 신규 추가; 도메인별 문서 표에 pipeline.md 추가
- `spec/api.md` — Data 파이프라인 연동 파일 링크 추가
- `spec/admin.md` — ADM-01 비고: 수동 등록 제거, 수집 트리거 방식으로 대체 반영; ADM-03 비고: BE 구현 제거; ADM-08 비고: BE 구현 제거; ADM-09 기능명·설명 확장 (검색 2종 + 수집 4종)
- `spec/pipeline.md` — 섹션 ⑤ 갱신: `search_artists`, `search_concerts`, `collect_artist_initial` 함수 추가; BE 엔드포인트 참조 링크 추가

## 2026-06-07

- `spec/erd.md` — `concert_artist.confidence·matched_by` 컬럼 제거; `concert_artist_candidate` 테이블 추가 (파이프라인 매칭 후 어드민 검토 큐)
- `spec/api/admin.md` — `GET /api/admin/concerts/pending`, `PUT .../approve`, `PUT .../reject`, `POST .../artists` 4개 PENDING 검토 엔드포인트 추가; Data 파이프라인 트리거 3종(`POST /api/admin/data/collect/*`) 추가
- `spec/admin.md` — ADM-02 PENDING 검토 큐 BE 구현 반영; ADM-09 Data 파이프라인 수집 트리거 기능 추가
- `spec/pipeline.md` — 매칭 흐름 PENDING 큐 방식으로 갱신 (`concert_artist_candidate` 임시 저장 → 어드민 승인 후 `concert_artist` 확정); `confidence` 기반 노출 기준 제거; `is_coming` 동기화 조건 갱신


## 2026-06-04

- `spec/erd.md` — release_group·track 테이블 V12 재구성 반영: spotify_id 추가, mbid NOT NULL 제약 제거, release_group.total_tracks·track.disc_number·track.explicit 컬럼 추가

## 2026-06-03

- `spec/api/releases.md` — type 표기 UPPER_CASE → Pascal Case(`Album`/`Single`) 통일; `GET /api/releases/{id}` 응답에 `totalTracks` 추가; `tracks[]`에 `discNumber`·`explicit` 추가

## 2026-06-02


## 2026-05-30

- `spec/erd.md` — fetch_attempted_at 컬럼 setlist → concert로 이동, 변경 이력 수정

## 2026-05-28

- `spec/api/concerts.md` — `GET /api/concerts/{id}` 응답 필드 갱신: `thumbnailUrl` 제거, `posterUrls: String[]` → `posterUrl: String?`, `imageUrls: String[]` 추가 (`concert_image` 테이블)
- `spec/erd.md` — artist.image_url 컬럼 추가, concert_image 테이블 추가 및 관계 요약·변경 이력 반영

## 2026-05-27

- `spec/pipeline.md` — `artist.debut_date` 컬럼 제거 반영 (MusicBrainz `life-span.begin` 신뢰도 문제)
- `spec/pipeline.md` — Last.fm 월간 리스너 기준 아티스트 필터 정책 추가 (`LASTFM_MIN_LISTENERS`, 기본 1,000)
- `spec/pipeline.md` — 릴리즈 저장 순서 Album → EP → Single 명시

## 2026-05-24

- `spec/api/my.md` — `GET /api/my/history` `artistName` 타입 `String` → `String?` 수정 및 `confidence=HIGH` 기준 비고 추가

## 2026-05-23

- `spec/api/my.md` — `GET /api/my/history` BE 구현 완료 (MY-01 다녀온 공연, `my` 패키지 신규)

## 2026-05-23

- `spec/api/concerts.md` — `GET /api/concerts/popular` BE 반환 건수 10건 명시; `GET /api/concerts/stats` month 범위 초과 시 400 반환 정책 추가

## 2026-05-23

- `spec/admin.md` — ADM-03·04·07·08 BE 구현 완료 표시 추가
- `spec/api/admin.md` — `bookingLinks[].name`·`bookingLinks[].url` 필수 여부 Y로 수정; `bookingLinks` null/빈 배열 정책 명시

## 2026-05-22

- `spec/api/releases.md` — `tracks[].length_ms` → `tracks[].lengthMs` (camelCase 통일)
- `spec/api/admin.md` — `POST /api/admin/concerts`, `PUT /api/admin/concerts/{id}/state` status 영어 통일
