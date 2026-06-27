# 프로젝트 개요

> Jpop 아티스트 내한 공연 정보 통합 웹 플랫폼

| 항목 | 내용 |
|------|------|
| 최신 명세 | 2026-06-28 |
| 상태 | In Review |
| 프론트엔드 | React 19 + Vite, JavaScript (JSX) |
| 백엔드 | Java 21 (Spring Boot 4.x) |

---

**Coming**은 Jpop 아티스트 및 내한 공연 정보를 통합 관리·공유하는 웹 플랫폼입니다. KOPIS 공공 API와 MusicBrainz를 연동해 공연·아티스트 데이터를 수집하고, 사용자에게 공연 정보·아티스트 탐색·캘린더 기능을 제공합니다.

## 기술 스택

| 구분 | 상세 |
|------|------|
| 프론트엔드 | React 19 + Vite, JavaScript (JSX) — 라우터: React Router v6, 상태관리: Zustand (전역) + React Query (서버), HTTP: Axios |
| 백엔드 | Java 21 + Spring Boot 4.x |
| 데이터 수집 | Python (별도 레포지토리) — APScheduler 기반 스케줄링 |
| 인증 | OAuth 2.0 (Google / Kakao) + JWT (Access / Refresh Token) |
| 외부 API | KOPIS OpenAPI, MusicBrainz API, setlist.fm API |
| 이미지 CDN | 미정 (S3 + CloudFront 또는 외부 CDN 검토 중) |

## 우선순위 기준

| 등급 | 의미 | 목표 릴리스 |
|------|------|-------------|
| P0 | MVP 핵심 기능 — 없으면 서비스 불가 | v1.0 |
| P1 | 주요 기능 — 빠른 시일 내 제공 필요 | v1.1 |
| P2 | 부가 기능 — 추후 개발 | v2.0 이후 |

## 관련 레포지토리

| 레포 | 설명 |
|------|------|
| coming-frontend | React 프론트엔드 |
| coming-backend | Java Spring Boot 백엔드 |
| coming-collector | Python 데이터 수집 파이프라인 |
