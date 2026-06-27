# 인프라 개요

> Coming 서비스 인프라 구성 요약

| 항목 | 내용 |
|------|------|
| 최신 명세 | 2026-06-27 |
| 상태 | Active |

## 인스턴스 구성

| 인스턴스 | 역할 | 주요 컨테이너 |
|----------|------|--------------|
| Instance 1 | 앱 서버 | Spring BE, Python Data, Redis |
| Instance 2 | 모니터링 | Prometheus, Grafana |
| DB Instance | 데이터베이스 | PostgreSQL (별도 관리) |

## 상세 문서

- [아키텍처](infra/architecture.md) — 인스턴스별 컨테이너 구성 및 네트워크
- [배포 파이프라인](infra/deployment.md) — CI/CD 흐름 및 배포 방식
