# 배포 파이프라인

> 레포별 CI/CD 흐름 및 배포 방식

## 개요

- 각 앱 레포(coming-be, coming-data)는 **이미지 빌드 및 배포**를 담당하는 CI/CD를 독립적으로 보유
- 서버의 `docker-compose.yml`은 **최초 1회 SSH 직접 작성** 후 유지
- 이미지 레지스트리: **GitHub Container Registry (ghcr.io)**

## CI/CD 흐름

```
coming-be (PR/push)                  coming-data (PR/push)
  └── CI                               └── CI
      ├── 단위 테스트                        ├── pytest
      └── 이미지 빌드 검증                   └── ruff check

coming-be (main 머지)                coming-data (main 머지)
  └── CD                               └── CD
      ├── 이미지 빌드                        ├── 이미지 빌드
      ├── ghcr.io/coming/be:latest push     ├── ghcr.io/coming/data:latest push
      └── SSH → Instance 1                  └── SSH → Instance 1
              └── docker compose pull be            └── docker compose pull data
                  docker compose up -d be                docker compose up -d data
```

## 레포별 역할 분담

| 레포 | 보유 파일 | 없는 파일 |
|------|----------|----------|
| coming-be | `Dockerfile`, CI/CD 워크플로우 | `docker-compose.yml` |
| coming-data | `Dockerfile`, CI/CD 워크플로우 | `docker-compose.yml` |

## 서버 파일 구조

### Instance 1

```
~/compose/
├── docker-compose.yml   # 서버 직접 작성 (레포에 없음)
└── .env                 # CD 실행 시 GitHub Secrets로 자동 갱신
```

`docker-compose.yml` 구성:

```yaml
services:
  be:
    image: ghcr.io/cominggg/coming-be:latest
    ports: ["8080:8080"]
    env_file: .env
    environment:
      JAVA_OPTS: "-Xmx300m -Xms200m"
    depends_on: [redis]

  data:
    image: ghcr.io/cominggg/coming-data:latest
    env_file: .env

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

### Instance 2

```
~/compose/
├── docker-compose.yml   # 수동 관리 (모니터링은 배포 자동화 없음)
└── prometheus.yml
```

## 배포 특성

| 서비스 | 배포 방식 | 다운타임 |
|--------|----------|----------|
| spring-be | CD 자동 배포 | 없음 (블루그린 전환) |
| python-data | CD 자동 배포 | 수 초 (스케줄러 재시작) |
| 모니터링 | 수동 SSH | 해당 없음 |

## GitHub Secrets 목록 (Instance 1 CD)

| Secret | 용도 |
|--------|------|
| `SERVER_HOST` | 서버 IP |
| `SERVER_USER` | SSH 사용자명 |
| `SSH_PRIVATE_KEY` | SSH 개인키 |
| `DB_HOST` / `DB_PORT` / `DB_NAME` / `DB_USER` / `DB_PASSWORD` | DB 접속 정보 |
| `KOPIS_API_KEY` | KOPIS API |
| `MUSICBRAINZ_USER_AGENT` | MusicBrainz API |
| `SETLISTFM_API_KEY` | setlist.fm API |
| `SPOTIFY_CLIENT_ID` / `SPOTIFY_CLIENT_SECRET` | Spotify API |
| `INTERNAL_SECRET` | 내부 서비스 인증 |
