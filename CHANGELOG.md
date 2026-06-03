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
