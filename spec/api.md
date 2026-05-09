# API 명세 (Spring 백엔드)

인증이 필요한 엔드포인트는 `Authorization: Bearer {token}` 헤더를 요구합니다. ADMIN 권한 필요 엔드포인트는 일반 사용자 접근 시 403 반환.

## 인증

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/auth/login/{provider}` | OAuth 2.0 소셜 로그인 리다이렉트 | 불필요 | provider: `google`\|`kakao` |
| GET | `/api/auth/callback/{provider}` | OAuth 콜백 처리 및 JWT 발급 | 불필요 | |
| POST | `/api/auth/refresh` | Access Token 재발급 (Refresh Token 사용) | 불필요 | HttpOnly Cookie에서 Refresh Token 추출 |
| POST | `/api/auth/logout` | 로그아웃 (토큰 무효화) | 필요 | |
| DELETE | `/api/auth/withdraw` | 회원 탈퇴 | 필요 | |
| GET | `/api/auth/me` | 내 프로필 조회 | 필요 | |
| PUT | `/api/auth/me` | 내 프로필 수정 (닉네임, 이미지) | 필요 | multipart/form-data |

## 아티스트

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/artists` | 아티스트 목록 조회 (검색) | 불필요 | name, page, size(기본 25) |
| GET | `/api/artists/{id}` | 아티스트 상세 정보 조회 | 불필요 | 응답에 `followerCount`(int) 포함 — `user_follow_artist` 집계값. ERD 컬럼 추가 없이 COUNT 쿼리로 산출 |
| GET | `/api/artists/{id}/concerts` | 아티스트 내한 공연 목록 | 불필요 | tab: all\|upcoming\|past, page, size(기본 10) |
| GET | `/api/artists/{id}/releases` | 아티스트 디스코그래피 (앨범·싱글·EP + 수록곡) | 불필요 | type: ALBUM\|SINGLE\|EP (복수 허용), page, size(기본 10) |
| GET | `/api/releases/{id}` | 릴리즈 상세 조회 (앨범·싱글·EP 공통) | 불필요 | 트랙리스트·커버 포함. REL-02 |
| POST | `/api/artists/{id}/follow` | 관심 아티스트 추가 | 필요 | |
| DELETE | `/api/artists/{id}/follow` | 관심 아티스트 제거 | 필요 | |
| GET | `/api/artists/following` | 팔로우한 아티스트 목록 | 필요 | |

## 공연

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/concerts` | 내한 공연 목록 조회 (필터) | 불필요 | dateFrom(YYYY-MM-DD), dateTo(YYYY-MM-DD), artistId, region, page, size(기본 20). 기존 date 파라미터 하위 호환 유지 |
| GET | `/api/concerts/stats` | 이달 공연 건수 조회 | 불필요 | year(int), month(int) → `{ "concertCount": N }`. 홈 통계 배너 미사용으로 엔드포인트 유지만 |
| GET | `/api/concerts/{id}` | 공연 상세 정보 조회 | 불필요 | 조회수 +1 처리 |
| GET | `/api/concerts/{id}/setlist` | 셋리스트 조회 | 불필요 | 공연 완료 후 제공 |
| GET | `/api/concerts/popular` | 인기 공연 목록 (조회수 기반) | 불필요 | |
| GET | `/api/concerts/following` | 관심 아티스트 예정 공연 목록 | 필요 | |

## 캘린더

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/calendar` | 전체 공연 캘린더 목록 | 불필요 | year, month 파라미터 |
| GET | `/api/calendar/my` | 내 캘린더 공연 목록 | 필요 | page, size(기본 10) |
| POST | `/api/calendar/{concertId}` | 내 캘린더에 공연 추가 | 필요 | |
| DELETE | `/api/calendar/{concertId}` | 내 캘린더에서 공연 제거 | 필요 | |

## 마이페이지

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/my/history` | 다녀온 공연 목록 (공연 히스토리) | 필요 | page, size(기본 10) |

## 데이터 문의

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| POST | `/api/inquiries` | 데이터 문의 등록 (INQ-01~03) | 필요 | type(ARTIST\|CONCERT\|SETLIST), targetId, title, content |
| GET | `/api/inquiries/my` | 내 문의 내역 조회 | 필요 | page, size, status 필터 |
| GET | `/api/inquiries/my/{id}` | 내 문의 상세 조회 | 필요 | 처리 결과·반려사유 포함 |

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
