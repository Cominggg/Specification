# Coming DB ERD

> FK 제약은 V2에서 제거했다. 논리 참조만 존재하며 concert_image, concert_artist_candidate만 실제 FK를 가진다.

```sql
CREATE TABLE "user" (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    provider varchar(20) NOT NULL,              -- GOOGLE / KAKAO / APPLE
    provider_id varchar(255) NOT NULL,          -- 탈퇴 시 'withdrawn:{id}'로 치환
    email varchar(255),
    nickname varchar(50),                       -- 가입 완료 전·탈퇴 후 NULL
    role varchar(20) NOT NULL,                  -- USER / ADMIN / PENDING(가입 미완료)
    status varchar(20) NOT NULL,                -- ACTIVE / INACTIVE(탈퇴) / SUSPENDED
    birth_year integer,
    agreed_marketing boolean,
    apple_refresh_token varchar(512),
    suspended_until timestamp,                  -- SUSPENDED인데 NULL이면 영구 정지
    suspension_reason varchar(20),              -- SPAM / ABUSE / SEXUAL / ILLEGAL / COPYRIGHT / PRIVACY / ETC
    suspension_detail text,
    withdrawn_at timestamp,
    created_at timestamp,
    updated_at timestamp,
    UNIQUE (provider, provider_id)
);
CREATE UNIQUE INDEX user_nickname_unique ON "user"(nickname) WHERE nickname IS NOT NULL;

CREATE TABLE artist (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    mbid varchar(36) NOT NULL UNIQUE,
    name varchar(255) NOT NULL,
    sort_name varchar(255),
    is_coming boolean NOT NULL DEFAULT false,
    image_url text,
    spotify_album_total int  -- [Data 전용] Spotify 앨범 총 개수 캐시
);

CREATE TABLE artist_alias (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    artist_id bigint NOT NULL,
    name varchar(255) NOT NULL,
    locale varchar(10),
    UNIQUE (artist_id, name)
);

CREATE TABLE artist_url (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    artist_id bigint NOT NULL,
    type varchar(100) NOT NULL,
    url text NOT NULL,
    UNIQUE (artist_id, type)  -- 아티스트당 링크 타입별 1개
);

CREATE TABLE user_follow_artist (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id bigint NOT NULL,
    artist_id bigint NOT NULL,
    created_at timestamp,
    UNIQUE (user_id, artist_id)
);
CREATE INDEX idx_user_follow_artist_artist_id ON user_follow_artist (artist_id);

CREATE TABLE concert (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    kopis_id varchar(50) UNIQUE,                -- 어드민이 직접 등록하는 공연은 KOPIS 연동 없이 저장(NULL)
    title varchar(500) NOT NULL,
    "cast" text,
    start_date date NOT NULL,
    end_date date NOT NULL,
    venue_name varchar(255) NOT NULL,
    poster_url text,
    price text,
    status varchar(20) NOT NULL,                -- UPCOMING / ONGOING / ENDED / CANCELLED / EXCLUDED / PENDING
    view_count bigint NOT NULL DEFAULT 0,
    kopis_update_date date NOT NULL,
    fetch_attempted_at timestamptz,
    ticket_open_at timestamp
);
CREATE INDEX idx_concert_date_range ON concert (start_date, end_date, status);
CREATE INDEX idx_concert_ticket_open_at ON concert (ticket_open_at, status);

CREATE TABLE concert_booking_link (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    concert_id bigint NOT NULL,
    name varchar(100) NOT NULL,
    url text NOT NULL,
    UNIQUE (concert_id, url)
);

CREATE TABLE concert_image (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    concert_id bigint NOT NULL REFERENCES concert(id),
    url        text   NOT NULL,
    position   int    NOT NULL DEFAULT 0
);

CREATE TABLE concert_artist (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    concert_id bigint NOT NULL,
    artist_id bigint NOT NULL,
    UNIQUE (concert_id, artist_id)
);
CREATE INDEX idx_concert_artist_artist_id ON concert_artist (artist_id);

CREATE TABLE concert_artist_candidate (
    id         bigserial PRIMARY KEY,  -- 다른 테이블과 달리 IDENTITY가 아닌 시퀀스
    concert_id bigint NOT NULL REFERENCES concert(id),
    artist_id  bigint NOT NULL REFERENCES artist(id),
    UNIQUE (concert_id, artist_id)
);

CREATE TABLE user_concert_calendar (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id bigint NOT NULL,
    concert_id bigint NOT NULL,
    created_at timestamp,
    UNIQUE (user_id, concert_id)
);

CREATE TABLE setlist (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    concert_id bigint NOT NULL,
    setlist_fm_id varchar(50) NOT NULL UNIQUE,
    collected_at timestamp NOT NULL,
    attribution_url text
);
CREATE INDEX idx_setlist_concert_id ON setlist (concert_id);

CREATE TABLE setlist_track (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    setlist_id bigint NOT NULL,
    position int NOT NULL,
    song_name varchar(255) NOT NULL,
    info text
);
CREATE INDEX idx_setlist_track_setlist_id ON setlist_track (setlist_id);

CREATE TABLE release_group (
    id                 bigint       GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    mbid               varchar(36)  UNIQUE,
    spotify_id         varchar(22)  UNIQUE,
    artist_id          bigint       NOT NULL,
    title              varchar(500) NOT NULL,
    type               varchar(20),
    first_release_date date,
    cover_url          text,
    label              varchar(255),
    total_tracks       int
);
CREATE INDEX idx_release_group_artist_id ON release_group (artist_id);

CREATE TABLE track (
    id               bigint       GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    release_group_id bigint       NOT NULL,
    mbid             varchar(36)  UNIQUE,
    spotify_id       varchar(22)  UNIQUE,
    title            varchar(500) NOT NULL,
    position         int          NOT NULL,
    length_ms        int,
    disc_number      int,
    explicit         boolean
);
CREATE INDEX idx_track_release_group_id ON track (release_group_id);

CREATE TABLE inquiry (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id bigint NOT NULL,
    type varchar(20) NOT NULL,                  -- CONCERT / ARTIST / SETLIST / DATA_REQUEST / FEEDBACK
    target_id bigint,                           -- CONCERT·SETLIST=concert.id, ARTIST=artist.id, DATA_REQUEST·FEEDBACK=NULL
    title varchar(255) NOT NULL,
    content text NOT NULL,
    status varchar(20) NOT NULL,                -- PENDING / IN_PROGRESS / RESOLVED / REJECTED
    admin_note text,
    created_at timestamp,
    updated_at timestamp
);
CREATE INDEX idx_inquiry_user_id ON inquiry (user_id);
CREATE INDEX idx_inquiry_target_id ON inquiry (target_id);

CREATE TABLE post (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id bigint NOT NULL,
    category varchar(20) NOT NULL,              -- REVIEW / INFO / FREE
    title varchar(255) NOT NULL,
    content jsonb NOT NULL,                     -- TipTap JSON
    content_text text NOT NULL,                 -- 검색용 평문
    recommend_count bigint NOT NULL DEFAULT 0,
    view_count bigint NOT NULL DEFAULT 0,
    comment_count bigint NOT NULL DEFAULT 0,    -- 삭제되지 않은 댓글·답글 수 (V45 재집계)
    cover_concert_id bigint,                    -- 대표 공연 (V44)
    created_at timestamp,
    updated_at timestamp
);
CREATE INDEX idx_post_user_id ON post (user_id);
CREATE INDEX idx_post_category ON post (category);

CREATE TABLE post_entity_tag (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    post_id bigint NOT NULL,
    entity_type varchar(20) NOT NULL,           -- CONCERT / ARTIST / RELEASE / TRACK
    entity_id bigint NOT NULL,
    created_at timestamp
);
CREATE INDEX idx_post_entity_tag_post_id ON post_entity_tag (post_id);
CREATE INDEX idx_post_entity_tag_entity_type_entity_id ON post_entity_tag (entity_type, entity_id);

CREATE TABLE post_recommend (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id bigint NOT NULL,
    post_id bigint NOT NULL,
    created_at timestamp,
    UNIQUE (user_id, post_id)
);

CREATE TABLE comment (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    post_id bigint NOT NULL,
    user_id bigint NOT NULL,
    parent_comment_id bigint,                   -- NULL이면 댓글, 값이 있으면 답글
    content text NOT NULL,
    like_count bigint NOT NULL DEFAULT 0,
    is_deleted boolean NOT NULL DEFAULT false,  -- 소프트 삭제
    created_at timestamp
);
CREATE INDEX idx_comment_post_id ON comment (post_id);
CREATE INDEX idx_comment_parent_comment_id ON comment (parent_comment_id);

CREATE TABLE comment_like (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id bigint NOT NULL,
    comment_id bigint NOT NULL,
    created_at timestamp,
    UNIQUE (user_id, comment_id)
);

CREATE TABLE policy_document (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    type varchar(20) NOT NULL,                  -- TERMS / PRIVACY
    version varchar(50) NOT NULL,
    effective_date date NOT NULL,
    change_summary text NOT NULL,
    detail_url varchar(500) NOT NULL,
    created_at timestamp,
    UNIQUE (type, version)
);

CREATE TABLE user_policy_agreement (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id bigint NOT NULL,
    policy_id bigint NOT NULL,
    agreed_at timestamp NOT NULL,
    UNIQUE (user_id, policy_id)
);

CREATE TABLE policy_notification_target (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    policy_id bigint NOT NULL,
    user_id bigint NOT NULL,
    status varchar(20) NOT NULL DEFAULT 'PENDING',  -- PENDING / SENT / FAILED
    sent_at timestamp,
    retry_count integer NOT NULL DEFAULT 0,
    created_at timestamp,
    UNIQUE (policy_id, user_id)
);

CREATE TABLE notice (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id bigint NOT NULL,
    title varchar(255) NOT NULL,
    content jsonb NOT NULL,                     -- TipTap JSON (V46에서 text → jsonb)
    active boolean NOT NULL DEFAULT true,
    created_at timestamp,
    updated_at timestamp
);
CREATE INDEX idx_notice_active_created_at ON notice (active, created_at);

CREATE TABLE report (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    reporter_id bigint NOT NULL,
    target_type varchar(20) NOT NULL,           -- POST / COMMENT
    target_id bigint NOT NULL,
    reason varchar(20) NOT NULL,                -- SPAM / ABUSE / SEXUAL / ILLEGAL / COPYRIGHT / PRIVACY / ETC
    detail text,
    status varchar(20) NOT NULL,                -- PENDING / RESOLVED / REJECTED
    admin_note text,
    created_at timestamp,
    updated_at timestamp,
    UNIQUE (reporter_id, target_type, target_id)
);
CREATE INDEX idx_report_status ON report (status);
CREATE INDEX idx_report_target ON report (target_type, target_id);

CREATE TABLE rating (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id bigint NOT NULL,
    target_type varchar(20) NOT NULL,           -- CONCERT / RELEASE
    target_id bigint NOT NULL,
    score numeric(2,1) NOT NULL,
    created_at timestamp NOT NULL,
    updated_at timestamp NOT NULL,
    UNIQUE (user_id, target_type, target_id)
);
CREATE INDEX idx_rating_target ON rating (target_type, target_id);

-- Spring Batch 메타 테이블 6개는 V36(공식 schema-postgresql.sql)로 관리, 생략
```

## 테이블 관계 요약

| 테이블 | 설명 |
|--------|------|
| `user` | 서비스 유저 |
| `artist` | MusicBrainz 기반 아티스트 |
| `artist_alias` | 아티스트 별칭 (한/영/일) |
| `artist_url` | 아티스트 외부 링크 (SNS 등) |
| `user_follow_artist` | 유저 아티스트 팔로우 |
| `concert` | KOPIS 수집 공연. `status=PENDING`은 관리자 검토 대기 상태 |
| `concert_booking_link` | 예매처 링크 |
| `concert_image` | 공연 스틸컷 이미지 URL 목록 (position 오름차순) |
| `concert_artist` | 공연-아티스트 매핑 (어드민 승인 후 확정된 관계) |
| `concert_artist_candidate` | 파이프라인 매칭 후 관리자 검토 대기 후보 (승인 시 concert_artist로 이동, 거절 시 concert=EXCLUDED). 어드민 수동 추가·삭제 가능 |
| `user_concert_calendar` | 유저 공연 일정 저장 |
| `setlist` | setlist.fm 수집 셋리스트 (Data 소유) |
| `setlist_track` | 셋리스트 트랙 목록 (Data 소유) |
| `release_group` | 앨범·싱글·EP (MusicBrainz 또는 Spotify 기반) (Data 소유) |
| `track` | 릴리즈 트랙 (MusicBrainz 또는 Spotify 기반) (Data 소유) |
| `inquiry` | 유저 문의 |
| `post` | 커뮤니티 게시글 (category REVIEW/INFO/FREE, 본문 TipTap JSON) |
| `post_entity_tag` | 게시글 멘션 태그 (공연·아티스트·발매·트랙) |
| `post_recommend` | 게시글 추천 (유저당 1회) |
| `comment` | 댓글·답글 (parent_comment_id, 소프트 삭제) |
| `comment_like` | 댓글 좋아요 (유저당 1회) |
| `policy_document` | 약관·개인정보처리방침 버전 |
| `user_policy_agreement` | 유저별 약관 동의 이력 |
| `policy_notification_target` | 약관 개정 안내 메일 발송 대상 (Spring Batch) |
| `notice` | 공지사항 |
| `report` | 게시글·댓글 신고 |
| `rating` | 공연·발매 별점 (0.5 단위 numeric(2,1)) |
| `batch_*` | Spring Batch 메타 테이블 (V36) |

## 변경 이력

| 날짜 | 내용 |
|------|------|
| 2026-04-23 | venue 테이블 제거, concert에 venue_name/venue_address 컬럼으로 통합 |
| 2026-04-24 | artist_member 테이블 제거 — 그룹 멤버 관계 미관리 결정 |
| 2026-04-24 | user.is_deleted → user.status varchar(20) 로 변경 |
| 2026-04-24 | inquiry에 title varchar(255) 컬럼 추가 |
| 2026-04-24 | release_group.lastfm_summary 컬럼 제거 — API·파이프라인 미사용 |
| 2026-04-30 | concert.price(text) 컬럼 추가 — KOPIS pcseguidance 수집 |
| 2026-04-30 | concert_booking_link에 UNIQUE(concert_id, url) 제약 추가 — 중복 예매 링크 방지 |
| 2026-04-30 | 전체 테이블 id 컬럼에 GENERATED ALWAYS AS IDENTITY 추가 — INSERT 시 null 위반 방지 |
| 2026-05-04 | 전 컬럼 NOT NULL 명시 및 DEFAULT 추가 — 파이프라인 INSERT 쿼리와 정합성 검증 완료 |
| 2026-05-04 | concert_artist에 UNIQUE(concert_id, artist_id) 추가 — ON CONFLICT 대상 제약 명시 |
| 2026-05-04 | artist_alias에 UNIQUE(artist_id, name) 추가 — 동일 alias 중복 INSERT 방지 |
| 2026-05-04 | artist_url에 UNIQUE(artist_id, url) 추가 — 동일 URL 중복 INSERT 방지 |
| 2026-05-07 | matching_review_queue 테이블 제거 — has_match 필터로 매칭 실패 경로 소멸 |
| 2026-05-08 | concert_artist.approved 컬럼 제거 — confidence='HIGH'가 단일 노출 기준으로 통합 |
| 2026-05-09 | concert_status_log 테이블 제거 — 파이프라인 미사용, 상태 변경이 KOPIS 자동 수집으로만 발생 |
| 2026-05-13 | 전 테이블 FK 제약 제거 — 논리 참조만 유지, 이후 추가된 concert_image·concert_artist_candidate만 실제 FK 보유 (V2 마이그레이션) |
| 2026-05-22 | inquiry.concert_id·artist_id → target_id(bigint NOT NULL) 단일 컬럼으로 통합 — type 컬럼으로 참조 대상 구분 (CONCERT·SETLIST=concert.id, ARTIST=artist.id) |
| 2026-05-27 | artist.debut_date, artist_alias.is_learned, release_group.created_at/updated_at 컬럼 제거 — 수집 불일치 및 미사용 컬럼 정리 |
| 2026-05-28 | artist.image_url(text) 컬럼 추가 — 아티스트 프로필 이미지 URL 저장 |
| 2026-05-28 | concert_image 테이블 추가 — 공연 스틸컷 이미지 URL 관리 (position 정렬) |
| 2026-05-30 | concert.fetch_attempted_at(timestamptz) 컬럼 추가 — 파이프라인 중복 API 호출 방지 |
| 2026-06-04 | release_group·track 테이블 재구성 — spotify_id(varchar 22) 추가, mbid NOT NULL 제약 제거(MusicBrainz 없이 Spotify 단독 수집 허용), release_group.total_tracks·track.disc_number·track.explicit 컬럼 추가 |
| 2026-06-05 | artist_url UNIQUE(artist_id, url) → UNIQUE(artist_id, type) 변경, (artist_id, type) 기준 중복 정리 (V14 마이그레이션) |
| 2026-06-07 | concert_artist.confidence·matched_by 컬럼 제거 — 파이프라인 매칭 신뢰도 기준 폐기, 전체 공연 어드민 검토 큐로 통합 |
| 2026-06-07 | concert_artist_candidate 테이블 추가 — 파이프라인 매칭 결과를 어드민 검토 전 임시 저장 (승인 시 concert_artist로 이동, 거절 시 concert=EXCLUDED) |
| 2026-06-07 | concert_artist_candidate.id는 IDENTITY가 아닌 BIGSERIAL로 생성 (V16 마이그레이션) |
| 2026-06-11 | concert.venue_address 컬럼 제거 — 파이프라인 미수집, API 미노출 |
| 2026-06-11 | artist, artist_alias, concert, concert_artist, concert_artist_candidate audit 컬럼(created_at/updated_at) 제거 — 파이프라인 수집 테이블은 감사 이력 미관리. user, user_follow_artist, user_concert_calendar, inquiry는 유지 |
| 2026-06-17 | artist.spotify_album_total(int), concert.ticket_open_at(timestamp) 컬럼 추가 (V19 마이그레이션) |
| 2026-06-25 | user.nickname NULL 허용 + 부분 UNIQUE 인덱스(user_nickname_unique), birth_year·agreed_terms·agreed_privacy·agreed_marketing·agreed_at 컬럼 추가, role PENDING 도입 (V20 마이그레이션) |
| 2026-06-30 | setlist.attribution_url(text) 컬럼 추가 — setlist.fm 저작자 표시 URL 저장 (V21 마이그레이션) |
| 2026-10-29 | user.profile_image_url 컬럼 제거 반영 — V20 마이그레이션에서 삭제됨, 개인정보처리방침 수집 항목에서도 제외 |
| 2026-06-30 | concert_artist_candidate.matched_by 컬럼 제거 — title 단일 전략만 사용, write-only 고정값으로 의미 없음 (V22 마이그레이션) |
| 2026-07-02 | inquiry.target_id nullable 변경, type에 DATA_REQUEST·FEEDBACK 추가 — 대상 없는 데이터 요청·피드백 문의 허용 (V23 마이그레이션) |
| 2026-07-07 | user.email(varchar 255) 컬럼 추가 — 기존 user 행 전체 삭제 (V24 마이그레이션) |
| 2026-07-22 | concert.kopis_id NULL 허용 — 어드민이 직접 등록하는 공연은 KOPIS 연동 없이 저장, UNIQUE 유지 (V25 마이그레이션) |
| 2026-08-12 | UNIQUE 추가 — user(provider, provider_id), user_follow_artist(user_id, artist_id), user_concert_calendar(user_id, concert_id) (V26 마이그레이션) |
| 2026-08-13 | 성능 인덱스 추가 — user_follow_artist(artist_id), release_group(artist_id), concert(start_date, end_date, status), setlist(concert_id), setlist_track(setlist_id), track(release_group_id), inquiry(user_id), inquiry(target_id) (V27 마이그레이션) |
| 2026-08-13 | concert(ticket_open_at, status) 인덱스 추가 (V28 마이그레이션) |
| 2026-09-06 | concert_artist(artist_id) 인덱스 추가 (V29 마이그레이션) |
| 2026-09-10 | post, post_entity_tag, post_recommend 테이블 추가 (V30 마이그레이션) |
| 2026-09-11 | comment, comment_like 테이블 추가, post.comment_count 컬럼 추가 (V31 마이그레이션) |
| 2026-09-16 | policy_document 테이블 추가 (V32 마이그레이션) |
| 2026-09-16 | user_policy_agreement 테이블 추가 — 기존 user.agreed_* 동의 이력을 legacy 정책 레코드로 이관 (V33 마이그레이션) |
| 2026-09-16 | user.agreed_terms·agreed_privacy·agreed_at 컬럼 제거 (V34 마이그레이션) |
| 2026-09-16 | policy_notification_target 테이블 추가 (V35 마이그레이션) |
| 2026-09-16 | Spring Batch 메타 테이블 6개 추가 (V36 마이그레이션) |
| 2026-09-18 | policy_document.requires_reconsent 컬럼 제거 (V37 마이그레이션) |
| 2026-09-19 | notice 테이블 추가 (V38 마이그레이션) |
| 2026-09-19 | report 테이블 추가 (V39 마이그레이션) |
| 2026-09-20 | rating 테이블 추가 (V40 마이그레이션) |
| 2026-10-21 | user.apple_refresh_token(varchar 512) 컬럼 추가 (V41 마이그레이션) |
| 2026-10-29 | user.suspended_until·suspension_reason·suspension_detail·withdrawn_at 컬럼 추가 — status SUSPENDED 도입 (V42 마이그레이션) |
| 2026-10-29 | 기존 탈퇴 회원 개인정보 파기·provider_id를 `withdrawn:{id}`로 치환 (V43 마이그레이션) |
| 2026-11-07 | post.cover_concert_id 컬럼 추가 — 대표 공연 (V44 마이그레이션) |
| 2026-11-08 | post.comment_count를 삭제되지 않은 댓글·답글 수로 재집계 (V45 마이그레이션) |
| 2026-11-11 | notice.content text → jsonb(TipTap JSON) 변환 (V46 마이그레이션) |
