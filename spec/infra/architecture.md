# 아키텍처

> 인스턴스별 컨테이너 구성 및 네트워크 구조

## 전체 구조

```
                        인터넷
                          │
              ┌───────────▼───────────┐
              │     Instance 1        │
              │      (앱 서버)         │
              │                       │
              │  Nginx (호스트 직접 설치)│
              │    :80 / :443         │
              │       │               │
              │       ▼               │
              │  ┌─────────────┐      │
              │  │  be-blue    │      │ ← 활성 슬롯
              │  │   :8080     │      │
              │  └─────────────┘      │
              │  ┌─────────────┐      │
              │  │  be-green   │      │ ← 대기 슬롯
              │  │   :8081     │      │
              │  └─────────────┘      │
              │  ┌─────────────┐      │
              │  │ python-data │      │
              │  │ (스케줄러)   │      │
              │  └─────────────┘      │
              │  ┌─────────────┐      │
              │  │    redis    │      │
              │  │   :6379     │      │
              │  └─────────────┘      │
              └───────────────────────┘

              ┌───────────────────────┐
              │     Instance 2        │
              │     (모니터링)         │
              │                       │
              │  ┌─────────────┐      │
              │  │ prometheus  │      │
              │  │   :9090     │      │
              │  └─────────────┘      │
              │  ┌─────────────┐      │
              │  │   grafana   │      │
              │  │   :3000     │      │
              │  └─────────────┘      │
              └───────────────────────┘

              ┌───────────────────────┐
              │     DB Instance       │
              │  PostgreSQL (별도 관리) │
              └───────────────────────┘
```

## Instance 1 — 앱 서버

| 항목 | 내용 |
|------|------|
| 스펙 | 2GB RAM, 2 vCPU, 60GB SSD, 3TB Transfer |
| Nginx | 호스트 직접 설치 (apt), SSL 인증서 Certbot 관리 |
| 컨테이너 관리 | `~/compose/docker-compose.yml` (서버 직접 작성) |

### 컨테이너 목록

| 컨테이너 | 이미지 | 포트 | 비고 |
|----------|--------|------|------|
| be-blue | `ghcr.io/cominggg/coming-be:latest` | 8080 | JVM 힙 `-Xmx300m -Xms200m -XX:MaxMetaspaceSize=128m` |
| be-green | `ghcr.io/cominggg/coming-be:latest` | 8081 | JVM 힙 동일, 블루-그린 대기 슬롯 |
| python-data | `ghcr.io/cominggg/coming-data:latest` | - | APScheduler 상시 실행 |
| redis | `redis:7-alpine` | 6379 | |

> **RAM 예산**: 전환 순간 두 JVM 동시 기동 기준 약 1.5GB 사용 (여유 ~500MB).
> `-XX:MaxMetaspaceSize=128m`으로 메타스페이스 상한을 명시해 OOM 위험 감소.

### Nginx 리버스 프록시

```
/api/** → coming_backend (upstream.conf가 가리키는 슬롯)
```

`/etc/nginx/conf.d/upstream.conf`로 활성 슬롯 포트를 관리하며, 배포 시 교체 후 `nginx -s reload`로 무중단 전환.

### 활성 슬롯 추적

`~/compose/active_slot` 파일에 현재 활성 슬롯(`blue` 또는 `green`)을 기록.

## Instance 2 — 모니터링

| 항목 | 내용 |
|------|------|
| 스펙 | 512MB RAM, 2 vCPU, 20GB SSD, 1TB Transfer |
| 컨테이너 관리 | `~/compose/docker-compose.yml` (서버 직접 작성) |

### 컨테이너 목록

| 컨테이너 | 이미지 | 포트 | 비고 |
|----------|--------|------|------|
| prometheus | `prom/prometheus` | 9090 | 보존 기간 7일 |
| grafana | `grafana/grafana` | 3000 | |

## DB Instance

| 항목 | 내용 |
|------|------|
| 엔진 | PostgreSQL |
| 관리 | 별도 인스턴스 (앱 서버와 분리) |
