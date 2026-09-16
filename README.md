<div align="center">
  <img src=".github/logo.png" alt="Coming" width="180" />

  <br />
  <br />

  **Jpop 아티스트 내한 공연 정보 통합 웹 플랫폼**

  기획·정책·API·DB 명세를 관리하는 기획 명세서 레포지토리

  <br />

  ![Status](https://img.shields.io/badge/status-In_Review-yellow?style=flat-square)
  ![Latest Spec](https://img.shields.io/badge/latest_spec-2026--09--15-blue?style=flat-square)
  [![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)](LICENSE)

  <br />

  **[→ comingg.com](https://comingg.com)**

</div>

---

## 소개

**커밍**(Coming)은 Jpop 아티스트의 국내 내한 공연 정보를 한곳에서 확인할 수 있는 웹 플랫폼입니다. KOPIS(공연예술통합전산망)·MusicBrainz·setlist.fm·Spotify 등 여러 외부 소스에 흩어진 아티스트·공연·발매 정보를 자동으로 수집·매칭해, 공연 탐색부터 아티스트 팔로우, 캘린더 관리, 커뮤니티 게시판까지 하나의 서비스로 제공합니다.

이 레포지토리는 코드가 아닌 **기획·정책·API·DB 명세**만을 담당하며, 아래 세 레포지토리가 이 명세를 기준으로 구현됩니다.

## 관련 레포지토리

| 레포 | 설명 | 스택 |
|------|------|------|
| [Frontend](https://github.com/Cominggg/Frontend) | 웹 클라이언트 | React 19 + Vite, JavaScript (JSX) |
| [Backend](https://github.com/Cominggg/Backend) | REST API 서버 | Java 21 + Spring Boot 4.x |
| [Data](https://github.com/Cominggg/Data) | 데이터 수집 파이프라인 | Python (APScheduler + FastAPI) |

## 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 최신 명세 | 2026-09-15 |
| 상태 | In Review |
| 인증 | OAuth 2.0 (Google / Kakao) + JWT (Access / Refresh Token) |
| 외부 API | KOPIS OpenAPI, MusicBrainz API, setlist.fm API, Spotify Web API |

---

## 문서 구성

| 파일 | 내용 |
|------|------|
| [spec/overview.md](spec/overview.md) | 프로젝트 개요, 기술 스택, 우선순위 기준 |
| [spec/features.md](spec/features.md) | 기능 명세 (AUTH / ART / CON / CAL / MY / REL / POST / INQ) |
| [spec/api/](spec/api/) | API 엔드포인트 명세 (도메인별 분리: auth·artists·concerts·calendar·my·releases·posts·admin·pipeline) |
| [spec/policy/auth-policy.md](spec/policy/auth-policy.md) | 인증 및 토큰 정책 |
| [spec/policy/ux-policy.md](spec/policy/ux-policy.md) | 에러 처리·UX 정책 |
| [spec/policy/constraints.md](spec/policy/constraints.md) | 제약 사항 및 리스크 |
| [spec/data/erd.md](spec/data/erd.md) | 데이터베이스 구조 (ERD) |
| [spec/data/](spec/data/) | 데이터 수집 파이프라인 명세 (개요·스케줄러·매칭 로직·예외 처리·수집기별 상세) |
| [spec/admin/features.md](spec/admin/features.md) | 관리자 기능 명세 (ADM-xx) |
| [spec/infra.md](spec/infra.md) | 인프라 구성 개요 |
| [spec/infra/](spec/infra/) | 아키텍처·배포 파이프라인 상세 |
| [spec/legal/](spec/legal/) | 법적 문서 (이용약관·개인정보처리방침·마케팅 수신 동의) |
| [CHANGELOG.md](CHANGELOG.md) | 명세 변경 이력 |

---

## 기능 도메인

기능은 `기능코드-번호` 형식으로 관리하며, 상세 내용은 [spec/features.md](spec/features.md)에서 확인할 수 있습니다.

| 코드 | 도메인 | 주요 내용 |
|------|--------|-----------|
| `AUTH` | 회원 관리 | 소셜 로그인(Google/Kakao), 프로필 조회·수정, 회원 탈퇴, 회원가입 온보딩 |
| `ART` | 아티스트 | 목록·상세 조회, 내한 공연 내역, 팔로우, 내한 예정 배지 |
| `CON` | 공연 정보 | 목록·상세 조회, 인기 공연, 셋리스트, 상태 자동 갱신, 검색, 티켓 오픈 예정 |
| `CAL` | 캘린더 / 일정 | 월별 캘린더 뷰, 내 캘린더 공연 추가·제거 |
| `MY` | 마이페이지 | 다녀온 공연, 관심 아티스트, 예정 공연 |
| `REL` | 음악 발매 | 앨범·싱글·EP 조회, 상세, 전체 목록, 검색 |
| `POST` | 게시판 | 게시글 CRUD, 공연·아티스트·음악(발매/트랙) 인라인 멘션, 추천, 댓글, 엔티티별 백링크, 통합 검색, 인기 게시글·트렌딩 태그 |
| `INQ` | 데이터 문의 | 공연·아티스트·셋리스트 정보 문의, 내 문의 내역 |
| `ADM` | 관리자 | [spec/admin/features.md](spec/admin/features.md) 별도 관리 — 아티스트/공연 수동 관리, 검토 큐, 파이프라인 트리거 |

> `POST-01~09`는 이슈 #116(`feat/#116-post-board`)에서 구현된 커뮤니티 게시판이며, `ART-05`(아티스트 전용 게시판·P2·댓글 없음)와는 별개 기능입니다.

### 우선순위 기준

| 등급 | 의미 | 목표 릴리스 |
|------|------|-------------|
| P0 | MVP 핵심 기능 — 없으면 서비스 불가 | v1.0 |
| P1 | 주요 기능 — 빠른 시일 내 제공 필요 | v1.1 |
| P2 | 부가 기능 — 추후 개발 | v2.0 이후 |

---

## 데이터 파이프라인

Python 별도 레포지토리([Data](https://github.com/Cominggg/Data))에서 KOPIS·MusicBrainz·setlist.fm·Spotify 데이터를 수집·매칭해 Backend와 동일한 DB에 적재합니다. DB 스키마 변경은 Backend의 Flyway 마이그레이션으로 단일 관리합니다. 자세한 내용은 [spec/data/pipeline.md](spec/data/pipeline.md)(개요)와 하위의 [scheduler.md](spec/data/scheduler.md)·[matchers.md](spec/data/matchers.md)·[error-handling.md](spec/data/error-handling.md)·[collectors/](spec/data/collectors/), [spec/data/erd.md](spec/data/erd.md)를 참고하세요.

## 법적 문서

| 문서 | 내용 |
|------|------|
| [이용약관](spec/legal/terms-of-service.md) | 서비스 이용 관련 권리·의무·책임 사항 |
| [개인정보처리방침](spec/legal/privacy-policy.md) | 수집 항목, 이용 목적, 보유 기간 |
| [마케팅 수신 동의](spec/legal/marketing-consent.md) | 선택 동의 항목 및 철회 방법 |

---

## 명세 작성 규칙

- 모든 기능은 `기능코드-번호` 형식으로 관리 (예: `AUTH-01`, `CON-07`)
- 우선순위: P0 (MVP) / P1 (v1.1) / P2 (v2.0 이후)
- 명세 변경 시 반드시 [CHANGELOG.md](CHANGELOG.md)에 날짜(`YYYY-MM-DD`)와 변경 내용 기입 — 같은 날 여러 번이면 `(1)`, `(2)`로 구분

## License

[MIT](LICENSE) © 2026 Cominggg
