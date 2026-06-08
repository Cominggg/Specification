# 관리자 기능 명세

관리자 전용 대시보드 (별도 React 라우트 `/admin`). 관리자 권한(`ROLE_ADMIN`) 보유 계정만 접근 가능. 일반 사용자 접근 시 403 페이지 표시.

| ID | 기능명 | 설명 | 우선순위 | 비고 |
|----|--------|------|----------|------|
| ADM-01 | 아티스트 수동 등록·수정 | MusicBrainz 미등록 아티스트를 직접 입력. 이름(한/영/일), alias, 소속사, 데뷔일, 이미지 업로드. | P0 | 초기 데이터 구축 필수. **현재 BE 구현**: 수정(`PUT /api/admin/artists/{id}`)만 지원. 신규 아티스트 등록은 ADM-09 MBID 기반 수집 트리거(`POST /api/admin/data/collect/artists`)로 대체. alias·소속사·이미지 미구현. |
| ADM-02 | PENDING 공연 검토 큐 관리 | 파이프라인이 수집·매칭한 PENDING 상태 공연 목록 조회. 후보 아티스트(`concert_artist_candidate`) 확인 후 승인 / 거절 처리. 승인 시 `concert_artist`로 이동하고 날짜 기반으로 status 자동 계산. 거절 시 `EXCLUDED` 처리. 아티스트 직접 지정(`POST /api/admin/concerts/{id}/artists`)으로 후보 보완 가능. | P0 | **BE 구현 완료** |
| ADM-03 | 공연 수동 등록 | KOPIS 미등록 소규모 공연 직접 입력. 날짜·장소·아티스트 매핑·예매처 URL. | P1 | **BE 구현 제거.** KOPIS ID 기반 공연 수집 트리거(ADM-09 `POST /api/admin/data/collect/concerts`)로 대체. |
| ADM-04 | 공연 강제 상태 변경 | prfstate를 관리자가 직접 변경 가능 (긴급 정정용). `concert_status_log`에 변경 이력 기록. | P1 | **BE 구현 완료** |
| ADM-07 | 공연 정보 수정 | 기존 공연의 내용 필드 수정. 수정 가능 필드: 공연명(`title`)·출연진(`cast`)·시작일·종료일·공연장명·공연장 주소·포스터 URL·가격(`price`). 예매처 링크(`concert_booking_link`) 추가·수정·삭제 포함. 상태 변경은 ADM-04에서만 처리. | P1 | **BE 구현 완료** |
| ADM-08 | 공연 삭제 | 기존 공연을 영구 삭제. 삭제 전 확인 모달 표시(복구 불가 안내). 연관 데이터 cascade 삭제: `concert_booking_link`, `concert_artist`, `user_concert_calendar`, `setlist`·`setlist_track`, `concert_status_log`. | P1 | **BE 구현 제거.** |
| ADM-05 | 회원 관리 | 회원 목록 조회, 닉네임·이메일 검색, 계정 정지 처리. | P2 | |
| ADM-06 | 문의 목록 조회 및 처리 | 유저가 등록한 데이터 문의 목록 조회. 유형·상태별 필터링. 문의 상세 확인 후 처리 상태를 IN_PROGRESS → RESOLVED / REJECTED로 변경. 반려 시 반려 사유 입력 필수. | P1 | INQ-01~03 연동 |
| ADM-09 | Data 파이프라인 검색·수집 트리거 | 관리자가 직접 Data 파이프라인 검색 및 단건 수집을 트리거. **검색 2종**: MusicBrainz 아티스트 검색(`GET /api/admin/data/search/artists`), KOPIS 공연 검색(`GET /api/admin/data/search/concerts`). **수집 트리거 4종**: MBID 기반 아티스트 초기 수집(`POST /api/admin/data/collect/artists`), KOPIS ID 기반 공연 수집(`POST /api/admin/data/collect/concerts`), 아티스트 릴리즈 수집(`POST /api/admin/data/collect/artists/{id}/releases`), 공연 셋리스트 수집(`POST /api/admin/data/collect/concerts/{id}/setlist`). WebClient(`X-Internal-Secret` 헤더) 연동. | P1 | **BE 구현 완료** |
