# API 명세 (Spring 백엔드)

인증이 필요한 엔드포인트는 `Authorization: Bearer {token}` 헤더를 요구합니다. ADMIN 권한 필요 엔드포인트는 일반 사용자 접근 시 403 반환.

## 공통 응답 규칙

| 항목 | 규칙 |
|------|------|
| 날짜 포맷 | `YYYY-MM-DD` (ISO 8601) 통일. FE에서 `YYYY.MM.DD` 표시 변환 처리 |
| 필드명 | DB 컬럼 snake_case → 응답 camelCase 변환. 주요 매핑: `venue_name`→`venue`, `start_date`→`startDate`, `end_date`→`endDate`, `poster_url`→`posterUrl`, `first_release_date`→`releaseDate`, `cover_url`→`coverUrl`, `profile_image_url`→`avatarUrl` |
| `isFollowing` | 인증된 사용자 요청 시 `user_follow_artist` 기준으로 포함. 비인증 요청 시 `false` 고정 반환 |
| `isInCalendar` | 인증된 사용자 요청 시 `user_concert_calendar` 기준으로 포함. 비인증 요청 시 `false` 고정 반환 |

## 인증

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/auth/login/{provider}` | OAuth 2.0 소셜 로그인 리다이렉트 | 불필요 | provider: `google`\|`kakao` |
| GET | `/api/auth/callback/{provider}` | OAuth 콜백 처리 및 JWT 발급 | 불필요 | |
| POST | `/api/auth/refresh` | Access Token 재발급 (Refresh Token 사용) | 불필요 | HttpOnly Cookie에서 Refresh Token 추출 |
| POST | `/api/auth/logout` | 로그아웃 (토큰 무효화) | 필요 | |
| DELETE | `/api/auth/withdraw` | 회원 탈퇴 | 필요 | |
| GET | `/api/auth/me` | 내 프로필 조회 | 필요 | 응답: `{ id, nickname, avatarUrl, role }` |
| PUT | `/api/auth/me` | 내 프로필 수정 (닉네임, 이미지) | 필요 | multipart/form-data |

## 아티스트

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/artists` | 아티스트 목록 조회 (검색) | 불필요 | 파라미터: `name`, `page`, `size`(기본 25). 응답 항목당 `isFollowing` 포함(공통 규칙 적용). `imageUrl`은 artist 테이블에 이미지 컬럼 없으므로 항상 `null` 반환 |
| GET | `/api/artists/{id}` | 아티스트 상세 정보 조회 | 불필요 | `followerCount`(int) — `user_follow_artist` COUNT 쿼리로 산출. `isFollowing` 포함(공통 규칙 적용). `links[]` — `artist_url` 테이블의 `type`을 BE에서 label로 변환해 응답: `{ "id": "{type}", "label": "Spotify", "url": "..." }` 형태 |
| GET | `/api/artists/{id}/concerts` | 아티스트 내한 공연 목록 | 불필요 | 파라미터: `tab`(`all`\|`upcoming`\|`past`), `page`, `size`(기본 10) |
| GET | `/api/artists/{id}/releases` | 아티스트 디스코그래피 (앨범·싱글·EP + 수록곡) | 불필요 | 파라미터: `type`(`ALBUM`\|`SINGLE`\|`EP`, 복수 허용), `page`, `size`(기본 10). 응답 항목당 `tracks[]` 포함: `{ position, title, length_ms }` (`setlist_track`이 아닌 `track` 테이블) |
| GET | `/api/releases` | 음악 전체 목록 조회 (REL-03) | 불필요 | 파라미터: `artistId`, `type`(`ALBUM`\|`SINGLE`\|`EP`\|`기타`), `page`, `size`(기본 20). `기타`는 ALBUM·SINGLE·EP 외 타입 전체 포함. 정렬: `releaseDate` 내림차순 고정. 홈 "새 앨범·싱글" 섹션은 이 API를 `size=8`로 재사용 |
| GET | `/api/releases/{id}` | 릴리즈 상세 조회 (앨범·싱글·EP 공통) | 불필요 | 트랙리스트·커버 포함. REL-02 |
| POST | `/api/artists/{id}/follow` | 관심 아티스트 추가 | 필요 | |
| DELETE | `/api/artists/{id}/follow` | 관심 아티스트 제거 | 필요 | |
| GET | `/api/artists/following` | 팔로우한 아티스트 목록 | 필요 | 마이페이지 "관심 아티스트" 탭(MY-02)에서 재사용. 응답 항목당 `hasUpcomingConcert`(`artist.is_coming`) 포함 |

## 공연

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/concerts` | 내한 공연 목록 조회 (필터) | 불필요 | 파라미터: `dateFrom`(YYYY-MM-DD), `dateTo`(YYYY-MM-DD), `artistId`, `region`, `status`(`공연예정`\|`공연중`\|`공연완료`\|`공연취소`), `page`, `size`(기본 20). 기존 `date` 파라미터 하위 호환 유지. 홈 "다가오는 공연" 섹션은 이 API를 `dateFrom={오늘}&size=6` 조합으로 재사용 |
| GET | `/api/concerts/stats` | 이달 공연 건수 조회 | 불필요 | `year`(int), `month`(int) → `{ "concertCount": N }`. 홈 통계 배너 미사용으로 엔드포인트 유지만 |
| GET | `/api/concerts/{id}` | 공연 상세 정보 조회 | 불필요 | 조회수 +1 처리. `isInCalendar` 포함(공통 규칙 적용). `posterUrls[]` — DB `poster_url` 단일값을 1-element 배열로 래핑해 반환(없으면 빈 배열 `[]`). `thumbnailUrl`은 `poster_url` 원본값 그대로 반환. `ticketLinks[]` — `concert_booking_link` 테이블: `{ "id": "{name}", "label": "{name}", "url": "..." }` |
| GET | `/api/concerts/{id}/setlist` | 셋리스트 조회 | 불필요 | 공연 완료 후 제공. 응답 필드명: `setlist_track.position`→`order`, `setlist_track.song_name`→`title`. 형태: `{ "tracks": [{ "order": 1, "title": "곡명" }] }`. 데이터 없으면 `{ "tracks": [] }` 반환 |
| GET | `/api/concerts/popular` | 인기 공연 목록 (조회수 기반) | 불필요 | 고정 건수 반환(size 파라미터 없음). 홈 캐러셀은 이 API 상위 5건을 FE에서 슬라이싱해 재사용. 홈 "인기 공연" 섹션도 재사용(desktop 3건 / mobile 6건 FE 슬라이싱) |
| GET | `/api/concerts/following` | 관심 아티스트 예정 공연 목록 | 필요 | 홈 "관심 아티스트 공연" 탭 및 공연 목록 "관심 아티스트만" 필터에서 재사용. ConcertCard 응답 구조 동일 |

## 캘린더

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/calendar` | 전체 공연 캘린더 목록 | 불필요 | 파라미터: `year`, `month`. 응답 항목: `{ concertId, artistName, title, startDate, endDate, status, posterUrl, venue }` |
| GET | `/api/calendar/my` | 내 캘린더 공연 목록 | 필요 | 파라미터: `page`, `size`(기본 10). concert 전체 객체 응답(`/api/calendar`와 동일 구조). 마이페이지 "예정 공연" 탭(MY-03)은 이 API를 재사용하며, FE에서 `startDate >= 오늘` 항목만 표시 |
| POST | `/api/calendar/{concertId}` | 내 캘린더에 공연 추가 | 필요 | |
| DELETE | `/api/calendar/{concertId}` | 내 캘린더에서 공연 제거 | 필요 | |

## 마이페이지

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/my/history` | 다녀온 공연 목록 (공연 히스토리) | 필요 | 파라미터: `page`, `size`(기본 10). 내 캘린더에 저장한 공연 중 `startDate < 오늘`인 항목. 응답 구조: `{ id, artistName, title, startDate, endDate, venue, status }` |

## 데이터 문의

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| POST | `/api/inquiries` | 데이터 문의 등록 (INQ-01~03) | 필요 | Body: `{ type(ARTIST\|CONCERT\|SETLIST), targetId, title, content }`. 동일 `targetId`로 PENDING 상태 문의 존재 시 409 반환 |
| GET | `/api/inquiries/my` | 내 문의 내역 조회 | 필요 | 파라미터: `page`, `size`, `status` 필터. `admin_note` 컬럼을 `resultMessage`(status=RESOLVED), `rejectReason`(status=REJECTED)으로 분리해 응답. 해당 없는 필드는 `null` |
| GET | `/api/inquiries/my/{id}` | 내 문의 상세 조회 | 필요 | 처리 결과·반려사유 포함. `inquiries/my`와 동일한 분리 응답 구조 |

## 관리자 (ROLE_ADMIN)

| 메서드 | 엔드포인트 | 설명 | 비고 |
|--------|------------|------|------|
| GET | `/api/admin/review-queue` | 매칭 검토 큐 목록 조회 | LOW confidence 매칭 목록 |
| POST | `/api/admin/review-queue/{id}/approve` | 매칭 승인 | alias 학습 포함 |
| POST | `/api/admin/review-queue/{id}/reject` | 매칭 거부 | |
| POST | `/api/admin/artists` | 아티스트 수동 등록 | |
| PUT | `/api/admin/artists/{id}` | 아티스트 정보 수정 | |
| POST | `/api/admin/concerts` | 공연 수동 등록 | |
| PUT | `/api/admin/concerts/{id}` | 공연 정보 수정 | title·cast·날짜·장소·poster_url·price·예매처 링크. 상태 변경 불포함 |
| DELETE | `/api/admin/concerts/{id}` | 공연 삭제 | 연관 데이터 cascade 삭제. 복구 불가 |
| PUT | `/api/admin/concerts/{id}/state` | 공연 상태 강제 변경 | 변경 이력 로그 |
| GET | `/api/admin/inquiries` | 문의 목록 조회 | type, status, page, size 필터 |
| GET | `/api/admin/inquiries/{id}` | 문의 상세 조회 | 유저 정보·대상 데이터 링크 포함 |
| PATCH | `/api/admin/inquiries/{id}/status` | 문의 처리 상태 변경 | status(IN_PROGRESS\|RESOLVED\|REJECTED), rejectReason |
