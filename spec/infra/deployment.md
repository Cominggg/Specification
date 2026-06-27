# 배포 파이프라인

> 레포별 CI/CD 흐름 및 배포 방식

## 개요

- 각 앱 레포(coming-be, coming-data)는 **이미지 빌드 및 배포**를 담당하는 CI/CD를 독립적으로 보유
- 서버의 `docker-compose.yml`은 **최초 1회 SSH 직접 작성** 후 유지
- 이미지 레지스트리: **GitHub Container Registry (ghcr.io)**

## CI/CD 흐름

```
coming-be (PR → main)                coming-data (PR → main)
  └── CI                               └── CI
      ├── 단위/통합 테스트                   ├── pytest
      │   (PostgreSQL·Redis 서비스 컨테이너) └── ruff check
      └── Docker 이미지 빌드 검증

coming-be (main 머지)                coming-data (main 머지)
  └── CD                               └── CD
      ├── 이미지 빌드                        ├── 이미지 빌드
      ├── ghcr.io/cominggg/coming-be:latest  ├── ghcr.io/cominggg/coming-data:latest
      │   push                              │   push
      └── SSH → Instance 1                  └── SSH → Instance 1
              ├── SCP: scripts/deploy.sh            └── docker compose pull data
              ├── .env 갱신 (GitHub Secrets)             docker compose up -d data
              └── bash ~/scripts/deploy.sh
                      ├── 비활성 슬롯 이미지 pull
                      ├── 비활성 슬롯 기동
                      ├── /actuator/health 대기 (최대 30s)
                      ├── upstream.conf 교체 + nginx -s reload
                      └── 구 슬롯 중지
```

## 레포별 역할 분담

| 레포 | 보유 파일 | 없는 파일 |
|------|----------|----------|
| coming-be | `Dockerfile`, `.dockerignore`, `.github/workflows/ci.yml`, `.github/workflows/cd.yml`, `scripts/deploy.sh` | `docker-compose.yml` |
| coming-data | `Dockerfile`, CI/CD 워크플로우 | `docker-compose.yml` |

## 서버 파일 구조

### Instance 1

```
~/compose/
├── docker-compose.yml   # 서버 직접 작성 (레포에 없음)
├── .env                 # CD 실행 시 GitHub Secrets로 자동 갱신
└── active_slot          # 현재 활성 슬롯 (blue 또는 green)

~/scripts/
└── deploy.sh            # CD가 SCP로 동기화 후 실행

/etc/nginx/conf.d/
└── upstream.conf        # 활성 슬롯 포트 지정, 배포마다 교체
```

`docker-compose.yml` 구성:

```yaml
services:
  be-blue:
    image: ghcr.io/cominggg/coming-be:latest
    ports: ["8080:8080"]
    env_file: .env
    environment:
      JAVA_OPTS: "-Xmx300m -Xms200m -XX:MaxMetaspaceSize=128m"
    depends_on: [redis]

  be-green:
    image: ghcr.io/cominggg/coming-be:latest
    ports: ["8081:8080"]
    env_file: .env
    environment:
      JAVA_OPTS: "-Xmx300m -Xms200m -XX:MaxMetaspaceSize=128m"
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
| spring-be | 블루-그린 (비활성 슬롯 기동 → health check → Nginx 전환 → 구 슬롯 중지) | 없음 |
| python-data | CD 자동 배포 | 수 초 (스케줄러 재시작) |
| 모니터링 | 수동 SSH | 해당 없음 |

## GitHub Secrets 목록 (Instance 1 CD)

| Secret | 용도 |
|--------|------|
| `SERVER_HOST` | 서버 IP |
| `SERVER_USER` | SSH 사용자명 |
| `SERVER_HOME` | 서버 홈 디렉토리 경로 (SCP target 경로 지정) |
| `SSH_PRIVATE_KEY` | SSH 개인키 |
| `DB_URL` | DB 접속 URL (`jdbc:postgresql://host:port/db`) |
| `DB_USERNAME` / `DB_PASSWORD` | DB 인증 정보 |
| `JWT_SECRET` | JWT 서명 키 |
| `REDIS_PASSWORD` | Redis 인증 비밀번호 |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Google OAuth |
| `KAKAO_CLIENT_ID` / `KAKAO_CLIENT_SECRET` | Kakao OAuth |
| `CORS_ALLOWED_ORIGINS` | 허용 오리진 |
| `OAUTH2_REDIRECT_BASE_URI` | OAuth2 콜백 기본 URI |
| `DATA_PIPELINE_SECRET` | 내부 서비스 인증 |
