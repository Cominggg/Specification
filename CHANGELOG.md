## 2026-05-23

- `spec/admin.md` — ADM-03·04·07·08 BE 구현 완료 표시 추가
- `spec/api/admin.md` — `bookingLinks[].name`·`bookingLinks[].url` 필수 여부 Y로 수정; `bookingLinks` null/빈 배열 정책 명시

## 2026-05-22

- `spec/api/artists.md` — `tracks[].length_ms` → `tracks[].lengthMs` (camelCase 통일)
- `spec/api/releases.md` — `tracks[].length_ms` → `tracks[].lengthMs` (camelCase 통일)
- `spec/api/artists.md` — `GET /api/artists` name 검색에 alias 포함 명시; `GET /api/artists/{id}/concerts` status 영어 통일 (`공연예정` → `UPCOMING` 등)
- `spec/api/admin.md` — `POST /api/admin/concerts`, `PUT /api/admin/concerts/{id}/state` status 영어 통일
