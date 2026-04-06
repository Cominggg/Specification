# Coming — 기획 명세서

> Jpop 아티스트 내한 공연 정보 통합 웹 플랫폼

| 항목 | 내용 |
|------|------|
| 최신 명세 | 2026-04-06 |
| 상태 | In Review |
| 프론트엔드 | React 19 + Vite, JavaScript (JSX) |
| 백엔드 | Java 17 (Spring Boot 3.x) |

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [기능 명세](#2-기능-명세)
3. [에러 처리 및 UX 정책](#3-에러-처리-및-ux-정책)
4. [인증 및 토큰 정책](#4-인증-및-토큰-정책)
5. [공통 UI/UX 정책](#5-공통-uiux-정책)
6. [데이터 수집 파이프라인 명세](#6-데이터-수집-파이프라인-명세)
7. [관리자 기능 명세](#7-관리자-기능-명세)
8. [API 명세 (Spring 백엔드)](#8-api-명세-spring-백엔드)
9. [제약 사항 및 리스크](#9-제약-사항-및-리스크)

---

## 1. 프로젝트 개요

**Coming**은 Jpop 아티스트 및 내한 공연 정보를 통합 관리·공유하는 웹 플랫폼입니다. KOPIS 공공 API와 MusicBrainz를 연동해 공연·아티스트 데이터를 수집하고, 사용자에게 공연 정보·아티스트 탐색·캘린더 기능을 제공합니다.

### 1.1 기술 스택

| 구분 | 상세 |
|------|------|
| 프론트엔드 | React 19 + Vite, JavaScript (JSX) — 라우터: React Router v6, 상태관리: Zustand (전역) + React Query (서버), HTTP: Axios |
| 백엔드 | Java 17 + Spring Boot 3.x |
| 데이터 수집 | Python (별도 레포지토리) — APScheduler 기반 스케줄링 |
| 인증 | OAuth 2.0 (Google / Kakao) + JWT (Access / Refresh Token) |
| 외부 API | KOPIS OpenAPI, MusicBrainz API, setlist.fm API |
| 이미지 CDN | 미정 (S3 + CloudFront 또는 외부 CDN 검토 중) |

### 1.2 우선순위 기준

| 등급 | 의미 | 목표 릴리스 |
|------|------|-------------|
| P0 | MVP 핵심 기능 — 없으면 서비스 불가 | v1.0 |
| P1 | 주요 기능 — 빠른 시일 내 제공 필요 | v1.1 |
| P2 | 부가 기능 — 추후 개발 | v2.0 이후 |

---

## 2. 기능 명세

### 회원 관리

| ID | 기능명 | 설명 | 우선순위 | 비고 |
|----|--------|------|----------|------|
| AUTH-01 | OAuth 2.0 소셜 로그인 | Google / Kakao OAuth 2.0 연동. 로그인 성공 시 온보딩 없이 홈으로 바로 이동.<br>▸ 비로그인 상태에서 인증 필요 기능 접근 시: 로그인 유도 모달 표시 후 현재 URL 유지 (redirect_uri 파라미터 전달)<br>▸ Access Token 만료(30분) 시 Refresh Token(7일)으로 자동 재발급. Refresh Token 만료 시 재로그인 요구. | P0 | Spring Security |
| AUTH-02 | 회원 프로필 조회·수정 | 닉네임, 프로필 이미지 관리. 프로필 이미지는 파일 업로드(jpg·png·webp, 최대 5MB) 방식. 업로드 실패 시 현재 이미지 유지. | P1 | |
| AUTH-03 | 회원 탈퇴 | 계정 및 관련 데이터(캘린더, 팔로우 등) 삭제. 탈퇴 전 확인 다이얼로그 표시. 탈퇴 완료 후 홈으로 이동. | P1 | |

### 아티스트

| ID | 기능명 | 설명 | 우선순위 | 비고 |
|----|--------|------|----------|------|
| ART-01 | 아티스트 목록 조회 | 이름 검색, 장르·데뷔년도 필터링 지원.<br>▸ 전체 목록 표시 (페이지네이션 없음). 필터 변경 시 목록 초기화. | P0 | Artist DB 기반 |
| ART-02 | 아티스트 상세 정보 조회 | 프로필, 멤버 구성, 소속사, 데뷔일. 관련 링크는 MusicBrainz url-rels 데이터(공식 홈페이지, 스트리밍 서비스 등)만 노출. | P0 | MusicBrainz 데이터 |
| ART-03 | 아티스트 내한 공연 내역 조회 | 2020년 이후 전체 내한 공연 목록. 과거·예정 탭 구분. 전체 표시. | P0 | Concert DB 연동 |
| ART-04 | 관심 아티스트 추가·제거 | 로그인 사용자의 팔로우 기능. 팔로우한 아티스트 목록은 마이페이지에서 관리. 비로그인 시 로그인 유도 모달 표시. | P0 | |
| ART-05 | 아티스트 게시판 | 팬 커뮤니티 게시글 작성·조회·댓글. 신고 및 모더레이션 기능 포함 필요. ※ 운영 리소스(모더레이션) 부담으로 론칭 후 트래픽 확인 뒤 도입 권고. | P2 | MVP 이후 개발 |
| ART-06 | 내한 예정 아티스트 배지 | 예정된 내한 공연이 존재하는 아티스트에 'Coming' 배지 표시. 아티스트 목록·상세 페이지 공통 적용.<br>▸ 배지 노출 조건: 오늘 날짜 이후 예정 공연이 1건 이상 존재하는 아티스트. CON-06 파이프라인과 동기화. | P1 | Concert DB 연동 |

### 공연 정보

| ID | 기능명 | 설명 | 우선순위 | 비고 |
|----|--------|------|----------|------|
| CON-01 | 내한 공연 목록 조회 | 날짜·아티스트·지역 필터. 기본 정렬은 공연 시작일 오름차순.<br>▸ 전체 목록 표시 (페이지네이션 없음). | P0 | |
| CON-02 | 공연 상세 정보 조회 | 날짜, 장소, 포스터, 가격, 예매처 링크(KOPIS relates 필드), 공연 상태 배지. updatedate 기반 일별 갱신.<br>▸ 공연 상태 값(prfstate): 공연예정 / 공연중 / 공연완료 / 공연취소 (KOPIS 원문 값 그대로 사용)<br>▸ 배지 색상: 공연예정=Blue, 공연중=Green, 공연완료=Gray, 공연취소=Red<br>▸ 포스터 이미지: KOPIS poster 필드 URL. 로드 실패 시 플레이스홀더 이미지(Coming 기본 이미지) 표시. | P0 | KOPIS 데이터 |
| CON-03 | 인기 내한 공연 조회 | 조회수 기반 상위 공연 목록. 홈 화면 노출 **4건(1행)**. 이후 전체보기로 유도. | P1 | |
| CON-04 | 관심 아티스트 공연 조회 | 팔로우한 아티스트의 예정 공연만 필터링해 표시. 비로그인 시 로그인 유도 배너 표시. | P0 | |
| CON-05 | 셋리스트 조회 | 공연 종료 후 setlist.fm API 데이터 연동. 데이터 미존재 시 '아직 등록된 셋리스트가 없습니다' empty state UI 표시. | P1 | 공연 완료 후 표시 |
| CON-06 | 공연 상태 자동 갱신 | 파이프라인이 KOPIS updatedate 변화 감지 시 DB의 prfstate 동기화. 공연 카드·상세 배지 자동 업데이트. | P1 | 파이프라인 연동 |
| CON-07 | 티켓팅 임박 공연 | 티켓 오픈일 기준 D-day를 표시한 공연 목록. 홈 화면 노출 최대 **6건**, 수평 스크롤.<br>▸ D-day 표시: D-0(당일), D-1, D-2… 형식. 오픈일 지난 공연 제외.<br>▸ 티켓 오픈일 정보 미등록 공연은 목록에서 제외. | P1 | |

### 캘린더 / 일정

| ID | 기능명 | 설명 | 우선순위 | 비고 |
|----|--------|------|----------|------|
| CAL-01 | 내한 공연 캘린더 조회 | 월별 캘린더 뷰로 공연 일정 시각화. 날짜 셀에 공연 도트 인디케이터 표시. 클릭 시 해당 날짜 공연 목록 표시.<br>▸ 멀티데이 공연: 첫날~마지막날 모두 도트 인디케이터 표시.<br>▸ 한 날짜에 공연 2건 초과 시: '외 N건' 텍스트로 축약 표시. | P0 | |
| CAL-02 | 내 캘린더에 공연 추가·제거 | 로그인 사용자가 공연을 개인 일정에 저장·삭제. 내 캘린더 뷰에서 저장한 공연만 필터링해 표시. | P0 | |

### 마이페이지

| ID | 기능명 | 설명 | 우선순위 | 비고 |
|----|--------|------|----------|------|
| MY-01 | 공연 히스토리 | 내 캘린더에 추가한 공연 중 공연일이 지난 항목을 '다녀온 공연' 탭으로 분리해 기록. 아티스트명·날짜·장소 표시. | P2 | |

### 검색

| ID | 기능명 | 설명 | 우선순위 | 비고 |
|----|--------|------|----------|------|
| SRC-01 | 통합 검색 | 아티스트명·공연명 통합 검색. 결과는 아티스트/공연 탭으로 구분.<br>▸ 검색 결과 정렬: 아티스트 탭 – 이름 오름차순 / 공연 탭 – 공연 시작일 오름차순.<br>▸ 결과 없을 때: '검색 결과가 없습니다. 다른 검색어를 입력해 보세요.' empty state UI 표시. | P1 | |
| SRC-02 | 아티스트명 자동완성 | 검색창 입력 시 Artist DB alias 기반 자동완성 드롭다운 표시. 최대 5건 노출. | P1 | 웹 적합성 검토 후 P2→P1 상향 |

### 데이터 문의

| ID | 기능명 | 설명 | 우선순위 | 비고 |
|----|--------|------|----------|------|
| INQ-01 | 데이터 문의 등록 | 셋리스트·공연·아티스트 정보가 누락되었거나 오류가 있을 때 유저가 직접 문의를 남길 수 있는 기능. 문의 유형(ARTIST / CONCERT / SETLIST), 대상 ID, 제목, 내용 입력.<br>▸ 대상 ID 입력 필드: 관련 페이지에서 '정보 문의' 버튼 클릭 시 자동 주입.<br>▸ 처리 상태: PENDING(접수) → IN_PROGRESS(처리중) → RESOLVED(완료) / REJECTED(반려)<br>▸ 등록 완료 시 '문의가 접수되었습니다.' 토스트 표시. 중복 방지: 동일 유형+대상 ID로 PENDING 상태인 문의 존재 시 재등록 불가. | P1 | 로그인 필요 |
| INQ-02 | 내 문의 내역 조회 | 로그인 사용자가 자신이 등록한 문의 목록과 처리 상태를 마이페이지에서 확인. 문의 유형·제목·상태·등록일 표시. | P1 | 마이페이지 내 탭 |

---

## 3. 에러 처리 및 UX 정책

모든 API 에러는 React의 Axios interceptor에서 중앙 처리합니다.

| 상황 | 영향 범위 | 처리 방식 |
|------|-----------|-----------|
| API 서버 장애 (5xx) | 전체 페이지 | '일시적인 오류가 발생했습니다. 잠시 후 다시 시도해 주세요.' 토스트 메시지 + 재시도 버튼 |
| KOPIS API 장애 | 공연 목록·상세 | 마지막 성공 캐시 데이터 표시. 상단에 '현재 최신 데이터를 불러오지 못하고 있습니다.' 배너 표시 |
| MusicBrainz API 장애 | 아티스트 상세 링크 영역 | 링크 섹션 숨김 처리. 나머지 프로필 정보는 정상 표시 |
| 이미지 로드 실패 | 공연 포스터 / 아티스트 프로필 | Coming 기본 플레이스홀더 이미지로 대체 (React onError 핸들러 사용) |
| 인증 토큰 만료 (401) | 인증 필요 API 전체 | Refresh Token으로 자동 재발급 시도. 실패 시 로그인 페이지로 이동 후 이전 URL redirect |
| 네트워크 오프라인 | 전체 페이지 | 브라우저 navigator.onLine 감지 → '인터넷 연결을 확인해 주세요.' 전역 배너 표시 |
| 검색 결과 없음 | SRC-01 검색 | '검색 결과가 없습니다. 다른 검색어를 입력해 보세요.' empty state UI 표시 |
| 셋리스트 미등록 | CON-05 셋리스트 | '아직 등록된 셋리스트가 없습니다.' empty state UI. setlist.fm 링크 제공 |
| 팔로우 / 캘린더 — 비로그인 | ART-04, CAL-02 | 로그인 유도 모달 표시 (현재 URL을 redirect_uri로 포함) |
| 문의 등록 — 비로그인 | INQ-01 | 로그인 유도 모달 표시 (현재 URL을 redirect_uri로 포함) |
| 문의 중복 등록 | INQ-01 | '이미 접수된 문의가 있습니다.' 토스트 메시지 표시. 동일 유형+대상 ID로 PENDING 상태 문의 존재 시 |

> React Query 도입 시 stale-while-revalidate 전략으로 캐시 fallback 처리 권장.

---

## 4. 인증 및 토큰 정책

| 항목 | 내용 |
|------|------|
| Access Token 유효시간 | 30분. HTTP `Authorization: Bearer {token}` 헤더로 전달. |
| Refresh Token 유효시간 | 7일. HttpOnly Cookie로 저장 (XSS 방어). |
| 자동 재발급 | Axios interceptor에서 401 응답 감지 → `POST /api/auth/refresh` 호출. 성공 시 원래 요청 재시도. 실패 시 로그인 페이지 이동. |
| 로그아웃 | `POST /api/auth/logout` → 서버 Refresh Token 블랙리스트 등록 + 클라이언트 Cookie 삭제. |
| 비로그인 접근 | 인증 필요 페이지 접근 시 로그인 모달 표시. 로그인 완료 후 원래 URL로 redirect. React Router의 PrivateRoute 컴포넌트로 구현. |
| 소셜 로그인 플로우 | `GET /api/auth/login/{provider}` → OAuth 리다이렉트 → `GET /api/auth/callback/{provider}` → JWT 발급 → 홈으로 이동. provider: `google`\|`kakao` |

---

## 5. 공통 UI/UX 정책

### 반응형

- **데스크탑 우선** (max-width 기준)
- 브레이크포인트: `max-width: 1279px` (tablet) / `max-width: 900px` (tablet-sm) / `max-width: 767px` (mobile)
- 예외: 앨범·수평 스트립 → `min-width: 768px`에서 그리드 전환

### 목록 표시

| 항목 | 방식 |
|------|------|
| 아티스트 목록 (ART-01) | 전체 표시 (페이지네이션 없음) |
| 공연 목록 (CON-01) | 전체 표시 (페이지네이션 없음) |
| 아티스트 공연 내역 (ART-03) | 전체 표시 (2020년~) |
| 자동완성 드롭다운 | 최대 5건 |

### 홈 섹션 구성

**섹션 순서**

1. 히어로 캐러셀 (주요 공연·릴리즈 하이라이트)
2. 티켓팅 임박 (CON-07, D-day 순, 최대 6건, 수평 스크롤)
3. 인기 공연 (CON-03, 조회수 순, 4건·1행, 전체보기 유도)
4. 새 앨범·싱글 (그리드, 데스크탑 768px+)
5. 관심 아티스트 공연 (비로그인 시 로그인 유도)

**원칙**

- 로그인 유도 섹션은 콘텐츠 섹션을 모두 소비한 **맨 마지막**에 배치
- 비로그인 접근 버튼 클릭 시 로그인 모달 표시 (페이지 이동 없음)
- 홈의 목록은 "맛보기" 수준으로 제한 후 전체보기로 유도

### 로딩 상태

- 데이터 로딩 중: 스켈레톤 UI 표시 (Spinner 지양)
- 버튼 클릭 후 처리 중: 버튼 disabled + 로딩 인디케이터

### 공연 상태 배지 색상

| 상태 | 색상 | 헥스 |
|------|------|------|
| 공연예정 | Blue | `#1565C0` |
| 공연중 | Green | `#2E7D32` |
| 공연완료 | Gray | `#757575` |
| 공연취소 | Red | `#C62828` |

### 이미지 처리 정책

| 이미지 종류 | 소스 | 실패 시 |
|-------------|------|---------|
| 공연 포스터 | KOPIS poster URL 직접 참조 | `/assets/poster-placeholder.png` |
| 아티스트 이미지 | MusicBrainz 제공 또는 관리자 업로드 | `/assets/artist-placeholder.png` |
| 프로필 이미지 (AUTH-02) | jpg·png·webp, 최대 5MB. 서버에서 리사이즈(최대 400×400px) 후 CDN 저장 | — |

### 매칭 신뢰도 노출 정책

- **HIGH 매칭**: 관리자 승인 없이 즉시 프론트 노출
- **LOW 매칭**: 관리자 승인 후 노출. 승인 대기 중 공연은 목록에서 제외

---

## 6. 데이터 수집 파이프라인 명세

Python 별도 레포지토리로 구현. Spring 백엔드와 동일한 DB를 공유하며, 스키마 변경은 Spring 쪽 마이그레이션 도구로 단일 관리.

### 6.1 레포지토리 구조

```
jpop-concert-collector/
├── collectors/
│   ├── kopis.py           # KOPIS API 수집
│   ├── musicbrainz.py     # MusicBrainz 아티스트 수집
│   └── setlist.py         # setlist.fm 셋리스트 수집
├── matchers/
│   └── artist_matcher.py  # alias 기반 매칭 로직 (rapidfuzz)
├── db/
│   └── repository.py      # DB 저장 (SQLAlchemy)
├── scheduler.py           # APScheduler 진입점
└── requirements.txt
```

### 6.2 수집 파이프라인 단계

#### ① 초기 구축 (1회성)

| 작업 | 상세 |
|------|------|
| MusicBrainz 수집 | JP 아티스트 목록 수집. country=JP, tag=j-pop 조건. 이름·alias(한/영/일)·url-rels 포함 저장.<br>`GET /ws/2/artist/?query=tag:j-pop AND country:JP&limit=100&offset={n}&fmt=json`<br>개별 상세: `GET /ws/2/artist/{mbid}?inc=aliases+tags+url-rels&fmt=json`<br>※ Rate Limit: 1 req/sec |
| 관리자 등록 | MusicBrainz 미등록 아티스트를 관리자 UI로 직접 입력. |

#### ② 주기적 수집 (스케줄)

| 작업 | 상세 |
|------|------|
| KOPIS 수집 | 내한 공연 후보 조회. visit=Y, genrenm=대중음악 조건. prfnm·prfcast·날짜·장소·updatedate 저장.<br>`GET /openApi/restful/pblprfr?service={key}&stdate=20200101&eddate={today}&shcate=GGGA&visit=Y&rows=100&cpage={n}`<br>상세: `GET /openApi/restful/pblprfr/{mt20id}?service={key}`<br>※ 주 1회 이상 권장 |
| 상태 갱신 | updatedate 변화 감지 시 prfstate DB 갱신. 매일 실행. |
| 매칭 ① | prfcast 기반 매칭 — 출연진 필드 → Artist DB alias 완전 일치. HIGH 신뢰도로 저장. 관리자 승인 없이 즉시 노출. |
| 매칭 ② | prfnm 기반 매칭 — 공연명 문자열 내 alias 부분 검색. rapidfuzz 임계값 적용. LOW 신뢰도로 저장. 관리자 승인 후 노출. |
| 매칭 실패 | 두 매칭 모두 실패 시 검토 큐 등록. 승인 시 alias 학습 → 다음 사이클 자동 매칭률 향상. |

#### ③ 셋리스트 수집 (스케줄)

| 작업 | 상세 |
|------|------|
| setlist.fm 수집 | prfstate=공연완료 대상. 공연 완료 후 1일 이내 실행.<br>`GET https://api.setlist.fm/rest/1.0/search/setlists?artistMbid={mbid}&countryCode=KR&p={n}`<br>Header: `x-api-key`, `Accept: application/json`<br>데이터 미존재 시 빈 상태 유지. |

---

## 7. 관리자 기능 명세

관리자 전용 대시보드 (별도 React 라우트 `/admin`). 관리자 권한(`ROLE_ADMIN`) 보유 계정만 접근 가능. 일반 사용자 접근 시 403 페이지 표시.

| ID | 기능명 | 설명 | 우선순위 | 비고 |
|----|--------|------|----------|------|
| ADM-01 | 아티스트 수동 등록·수정 | MusicBrainz 미등록 아티스트를 직접 입력. 이름(한/영/일), alias, 소속사, 데뷔일, 이미지 업로드. | P0 | 초기 데이터 구축 필수 |
| ADM-02 | 매칭 검토 큐 관리 | LOW 매칭 및 자동 매칭 실패 공연 목록 조회. 각 항목에 대해 '승인 / 거부 / alias 추가' 액션 제공. | P0 | 파이프라인 연동 |
| ADM-03 | 공연 수동 등록·수정 | KOPIS 미등록 소규모 공연 직접 입력. 날짜·장소·아티스트 매핑·예매처 URL. | P1 | |
| ADM-04 | 공연 강제 상태 변경 | prfstate를 관리자가 직접 변경 가능 (긴급 정정용). | P1 | 변경 이력 로그 필요 |
| ADM-05 | 회원 관리 | 회원 목록 조회, 닉네임·이메일 검색, 계정 정지 처리. | P2 | |
| ADM-06 | 문의 목록 조회 및 처리 | 유저가 등록한 데이터 문의 목록 조회. 유형·상태별 필터링. 문의 상세 확인 후 처리 상태를 IN_PROGRESS → RESOLVED / REJECTED로 변경. 반려 시 반려 사유 입력 필수. | P1 | INQ-01 연동 |

---

## 8. API 명세 (Spring 백엔드)

인증이 필요한 엔드포인트는 `Authorization: Bearer {token}` 헤더를 요구합니다. ADMIN 권한 필요 엔드포인트는 일반 사용자 접근 시 403 반환.

### 인증

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/auth/login/{provider}` | OAuth 2.0 소셜 로그인 리다이렉트 | 불필요 | provider: `google`\|`kakao` |
| GET | `/api/auth/callback/{provider}` | OAuth 콜백 처리 및 JWT 발급 | 불필요 | |
| POST | `/api/auth/refresh` | Access Token 재발급 (Refresh Token 사용) | 불필요 | HttpOnly Cookie에서 Refresh Token 추출 |
| POST | `/api/auth/logout` | 로그아웃 (토큰 무효화) | 필요 | |
| DELETE | `/api/auth/withdraw` | 회원 탈퇴 | 필요 | |
| GET | `/api/auth/me` | 내 프로필 조회 | 필요 | |
| PUT | `/api/auth/me` | 내 프로필 수정 (닉네임, 이미지) | 필요 | multipart/form-data |

### 아티스트

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/artists` | 아티스트 목록 조회 (검색·필터) | 불필요 | query, genre, debutYear |
| GET | `/api/artists/{id}` | 아티스트 상세 정보 조회 | 불필요 | |
| GET | `/api/artists/{id}/concerts` | 아티스트 내한 공연 목록 | 불필요 | tab: past\|upcoming |
| POST | `/api/artists/{id}/follow` | 관심 아티스트 추가 | 필요 | |
| DELETE | `/api/artists/{id}/follow` | 관심 아티스트 제거 | 필요 | |
| GET | `/api/artists/following` | 팔로우한 아티스트 목록 | 필요 | |

### 공연

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/concerts` | 내한 공연 목록 조회 (필터) | 불필요 | date, artistId, region |
| GET | `/api/concerts/{id}` | 공연 상세 정보 조회 | 불필요 | 조회수 +1 처리 |
| GET | `/api/concerts/{id}/setlist` | 셋리스트 조회 | 불필요 | 공연 완료 후 제공 |
| GET | `/api/concerts/popular` | 인기 공연 목록 (조회수 기반) | 불필요 | |
| GET | `/api/concerts/ticketing-soon` | 티켓팅 임박 공연 목록 (D-day 순) | 불필요 | |
| GET | `/api/concerts/following` | 관심 아티스트 예정 공연 목록 | 필요 | |

### 캘린더

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/calendar` | 전체 공연 캘린더 목록 | 불필요 | year, month 파라미터 |
| GET | `/api/calendar/my` | 내 캘린더 공연 목록 | 필요 | |
| POST | `/api/calendar/{concertId}` | 내 캘린더에 공연 추가 | 필요 | |
| DELETE | `/api/calendar/{concertId}` | 내 캘린더에서 공연 제거 | 필요 | |

### 마이페이지

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/my/history` | 다녀온 공연 목록 (공연 히스토리) | 필요 | 공연일 지난 캘린더 항목 |

### 검색

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| GET | `/api/search` | 아티스트·공연 통합 검색 | 불필요 | q, type(artist\|concert\|all) |

### 데이터 문의

| 메서드 | 엔드포인트 | 설명 | 인증 | 비고 |
|--------|------------|------|------|------|
| POST | `/api/inquiries` | 데이터 문의 등록 | 필요 | type(ARTIST\|CONCERT\|SETLIST), targetId, title, content |
| GET | `/api/inquiries/my` | 내 문의 내역 조회 | 필요 | page, size, status 필터 |
| GET | `/api/inquiries/my/{id}` | 내 문의 상세 조회 | 필요 | 처리 결과·반려사유 포함 |

### 관리자 (ROLE_ADMIN)

| 메서드 | 엔드포인트 | 설명 | 비고 |
|--------|------------|------|------|
| GET | `/api/admin/review-queue` | 매칭 검토 큐 목록 조회 | LOW 매칭 + 실패 목록 |
| POST | `/api/admin/review-queue/{id}/approve` | 매칭 승인 | alias 학습 포함 |
| POST | `/api/admin/review-queue/{id}/reject` | 매칭 거부 | |
| POST | `/api/admin/artists` | 아티스트 수동 등록 | |
| PUT | `/api/admin/artists/{id}` | 아티스트 정보 수정 | |
| POST | `/api/admin/concerts` | 공연 수동 등록 | |
| PUT | `/api/admin/concerts/{id}/state` | 공연 상태 강제 변경 | 변경 이력 로그 |
| GET | `/api/admin/inquiries` | 문의 목록 조회 | type, status, page, size 필터 |
| GET | `/api/admin/inquiries/{id}` | 문의 상세 조회 | 유저 정보·대상 데이터 링크 포함 |
| PATCH | `/api/admin/inquiries/{id}/status` | 문의 처리 상태 변경 | status(IN_PROGRESS\|RESOLVED\|REJECTED), rejectReason |

---

## 9. 제약 사항 및 리스크

| 항목 | 내용 | 대응 방안 |
|------|------|-----------|
| KOPIS prfcast 공백 | 출연진 필드가 비어 있는 공연 다수 존재 | prfnm 부분 매칭(LOW) + 관리자 검토 큐 병행 |
| 아티스트명 다국어 불일치 | 한/영/일 표기 혼재로 자동 매칭 실패 가능 | alias DB 지속 확충 + rapidfuzz 퍼지 매칭 |
| MusicBrainz Rate Limit | 초당 1 req 제한으로 초기 수집 시간 소요 | 배치 작업 분리, APScheduler 간격 조정 |
| KOPIS 미등록 공연 | 소규모 내한 공연 일부가 KOPIS에 등록되지 않을 수 있음 | 관리자 수동 등록(ADM-03) + 유저 문의(INQ-01) 채널 활용 |
| 공연 상태 갱신 지연 | KOPIS 데이터 갱신이 늦을 경우 일시적 부정확 표시 가능 | 갱신 주기 명시(일 1회), 마지막 동기화 일시 공연 상세 하단 노출 |
| setlist.fm 데이터 누락 | 커뮤니티 기반 입력으로 등록되지 않은 공연 다수 존재 | 빈 상태 UI + INQ-01 문의 버튼 제공. ADM-06 통해 수동 입력 |
| Bandsintown / Songkick API 사용 불가 | 이용약관 제한 또는 신규 발급 불가 | KOPIS 단독 사용으로 대체 확정 |
| LOW 매칭 공연 노출 지연 | 관리자 승인 전 미노출로 데이터 공백 발생 가능 | 관리자 검토 SLA 정의 권고 (수집 후 48시간 내 처리) |
| 문의 어뷰징 | 동일 데이터에 대한 중복·허위 문의 다수 접수 가능 | 동일 사용자·유형·엔티티 중복 문의 1건 제한. 반려 시 사유 메모 기록 |

---

> 본 명세서는 기획 검토 결과를 반영한 문서입니다. 구현 과정에서 변경될 수 있습니다.
