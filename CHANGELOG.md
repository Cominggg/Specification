## 2026-06-15

- `spec/api/concerts.md` — `GET /api/concerts`에 `inCalendar` 파라미터 추가 및 `status` 우선순위 비고 반영; `GET /api/concerts/search` 엔드포인트 신규 등록 (공연명·아티스트명·alias LIKE 검색, `q` 필수)
- `spec/api/releases.md` — `GET /api/releases`에 `following` 파라미터 추가 (팔로우 아티스트 필터); 정렬 설명 `releaseDate DESC NULLS LAST` 갱신
- `spec/features.md` — CON-01 내 캘린더 필터(`inCalendar=true`) 추가; CON-08 공연 검색 기능 신규 추가 (P1); REL-03 `following=true` 필터 및 NULLS LAST 정렬 명세 반영

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
- `spec/api/artists.md` — `GET /api/artists/{id}/releases` type 표기 Pascal Case 통일; `tracks[]`에 `discNumber`·`explicit` 추가

## 2026-06-02

- `spec/api/artists.md` — `GET /api/artists/{id}` 응답에서 `debutDate` 필드 제거 (V6 DB 컬럼 DROP 반영)
- `spec/api/artists.md` — 아티스트 3개 API의 `imageUrl` 설명에서 "항상 null" 문구 제거 (`image_url` 컬럼 추가 반영)

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

- `spec/api/artists.md` — `tracks[].length_ms` → `tracks[].lengthMs` (camelCase 통일)
- `spec/api/releases.md` — `tracks[].length_ms` → `tracks[].lengthMs` (camelCase 통일)
- `spec/api/artists.md` — `GET /api/artists` name 검색에 alias 포함 명시; `GET /api/artists/{id}/concerts` status 영어 통일 (`공연예정` → `UPCOMING` 등)
- `spec/api/admin.md` — `POST /api/admin/concerts`, `PUT /api/admin/concerts/{id}/state` status 영어 통일
