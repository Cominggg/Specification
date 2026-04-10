# Coming Specification

Jpop 아티스트 내한 공연 정보 통합 웹 플랫폼 **Coming**의 기획 명세서 레포지토리입니다.

## 관련 레포지토리

| 레포 | 설명 |
|------|------|
| coming-frontend | React 19 + Vite (JavaScript/JSX) |
| coming-backend | Java 17 + Spring Boot 3.x |
| coming-data | Python 데이터 수집 파이프라인 |

## 명세 파일 구성

| 파일 | 내용 |
|------|------|
| spec/overview.md | 프로젝트 개요, 기술 스택, 우선순위 기준 |
| spec/features.md | 기능 명세 (AUTH / ART / CON / CAL / MY / SRC / INQ) |
| spec/api.md | API 엔드포인트 명세 |
| spec/auth-policy.md | 인증 및 토큰 정책 |
| spec/ux-policy.md | 에러 처리·UX 정책 |
| spec/admin.md | 관리자 기능 명세 |
| spec/pipeline.md | 데이터 수집 파이프라인 명세 |
| spec/constraints.md | 제약 사항 및 리스크 |
| CHANGELOG.md | 명세 변경 이력 |

## 명세 작성 규칙

- 모든 기능은 `기능코드-번호` 형식으로 관리 (예: AUTH-01, CON-07)
- 우선순위: P0 (MVP), P1 (v1.1), P2 (v2.0 이후)
- 명세 변경 시 반드시 CHANGELOG.md에 날짜(YYYY-MM-DD)와 변경 내용 기입
- 날짜 포맷: `YYYY-MM-DD`, 같은 날 여러 번이면 `(1)`, `(2)` 구분

## 워크플로우

- 명세 업데이트: `/sync-docs` 스킬 사용
- 커밋: `/commit` 스킬 사용
- PR: `/pr` 스킬 사용
