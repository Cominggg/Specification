# Coming — 기획 명세서

> Jpop 아티스트 내한 공연 정보 통합 웹 플랫폼

| 항목 | 내용 |
|------|------|
| 최신 명세 | 2026-04-06 |
| 상태 | In Review |
| 프론트엔드 | React 19 + Vite, JavaScript (JSX) |
| 백엔드 | Java 17 (Spring Boot 3.x) |

---

## 문서 구성

| 파일 | 내용 |
|------|------|
| [spec/overview.md](spec/overview.md) | 프로젝트 개요, 기술 스택, 우선순위 기준 |
| [spec/features.md](spec/features.md) | 기능 명세 (AUTH / ART / CON / CAL / MY / SRC / INQ) |
| [spec/api.md](spec/api.md) | API 엔드포인트 명세 (Spring 백엔드) |
| [spec/auth-policy.md](spec/auth-policy.md) | 인증 및 토큰 정책 |
| [spec/ux-policy.md](spec/ux-policy.md) | 에러 처리·UX 정책, 공통 UI/UX 정책 |
| [spec/admin.md](spec/admin.md) | 관리자 기능 명세 |
| [spec/pipeline.md](spec/pipeline.md) | 데이터 수집 파이프라인 명세 |
| [spec/constraints.md](spec/constraints.md) | 제약 사항 및 리스크 |
| [CHANGELOG.md](CHANGELOG.md) | 명세서 변경 이력 |

---

## 관련 레포지토리

| 레포 | 설명 |
|------|------|
| coming-frontend | React 프론트엔드 |
| coming-backend | Spring Boot 백엔드 |
| coming-collector | Python 데이터 수집 파이프라인 |
