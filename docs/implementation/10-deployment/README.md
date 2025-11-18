# Phase 10: Deployment (온프레미스)

## 📋 개요

**온프레미스 서버**에 Docker 기반으로 배포

**호스팅 환경:**

- ✅ **온프레미스 서버** - FastAPI, PostgreSQL, Redis, Nginx
- ✅ **AWS S3** - 이미지 저장 (외부 서비스)
- ✅ **Prometheus + Grafana** - 모니터링 (온프레미스)

---

## 🎯 목표

1. **온프레미스 배포** - 자체 서버에서 운영
2. **컨테이너화** - Docker로 환경 일관성
3. **CI/CD** - GitHub Actions로 자동 배포
4. **모니터링** - Prometheus + Grafana

---

## 📁 파일 구조

```
docs/implementation/10-deployment/
├── README.md (이 파일)
├── docker.md              ✅ 온프레미스 Docker 설정
├── ci-cd.md               ✅ GitHub Actions CI/CD
└── monitoring.md          ✅ Prometheus + Grafana
```

---

## 🚀 배포 순서

### Step 1: Docker 설정

**파일:** `docker.md`

**핵심 내용:**

- Dockerfile (Python 3.11, Uvicorn)
- docker-compose.yml (API, DB, Redis, Nginx)
- 환경 변수 (.env)
- Nginx 리버스 프록시
- SSL 인증서 설정

### Step 2: CI/CD 파이프라인

**파일:** `ci-cd.md`

**핵심 내용:**

- GitHub Actions CI (테스트, 린트)
- GitHub Actions CD (SSH 배포)
- 무중단 배포 (Rolling update)
- 자동 롤백

### Step 3: 모니터링

**파일:** `monitoring.md`

**핵심 내용:**

- Prometheus (메트릭 수집)
- Grafana (시각화)
- Alertmanager (알림)
- Sentry (에러 추적)

---

## 🔧 온프레미스 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                   온프레미스 서버                          │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │  Nginx   │───▶│ FastAPI  │───▶│PostgreSQL│          │
│  │  (80/443)│    │  (8000)  │    │  (5432)  │          │
│  └──────────┘    └──────────┘    └──────────┘          │
│                        │                                  │
│                        ▼                                  │
│                  ┌──────────┐                            │
│                  │  Redis   │◀───────────┐              │
│                  │  (6379)  │            │              │
│                  └──────────┘            │              │
│                        ▲                  │              │
│                        │                  │              │
│              ┌─────────┴─────────┐       │              │
│              │                   │       │              │
│        ┌──────────┐        ┌──────────┐ │              │
│        │  Celery  │        │  Celery  │ │              │
│        │  Worker  │        │   Beat   │─┘              │
│        └──────────┘        └──────────┘                │
│                                                           │
│  ┌──────────────────────────────────────────┐           │
│  │         Monitoring Stack                  │           │
│  ├──────────────────────────────────────────┤           │
│  │  Prometheus (9090)                        │           │
│  │  Grafana (3000)                           │           │
│  │  Alertmanager (9093)                      │           │
│  └──────────────────────────────────────────┘           │
│                                                           │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │    AWS S3        │  (이미지 저장)
              └──────────────────┘
```

---

## 🔧 환경 변수

### 필수 환경 변수

```bash
# Database (온프레미스)
DB_USER=fmr_user
DB_PASSWORD=your_secure_password
DB_NAME=fmr_production

# AWS S3 (이미지 저장)
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_REGION=ap-northeast-2
S3_BUCKET_NAME=fmr-images

# Redis
REDIS_PASSWORD=your_redis_password

# Celery + Redis
CELERY_BROKER_URL=redis://:your_redis_password@redis:6379/0
CELERY_RESULT_BACKEND=redis://:your_redis_password@redis:6379/0
REDIS_URL=redis://:your_redis_password@redis:6379/0

# Timezone
TIMEZONE=Asia/Seoul

# External APIs
PORTONE_API_KEY=your_portone_key
PORTONE_API_SECRET=your_portone_secret
REPLICATE_API_TOKEN=your_replicate_token
OPENAI_API_KEY=your_openai_key
COOLSMS_API_KEY=your_coolsms_key
COOLSMS_API_SECRET=your_coolsms_secret

# LangChain (선택)
LANGCHAIN_API_KEY=your_langchain_key
LANGCHAIN_PROJECT=fmr-production

# Monitoring
GRAFANA_PASSWORD=your_grafana_password
SENTRY_DSN=your_sentry_dsn
SLACK_WEBHOOK_URL=your_slack_webhook
```

---

## 🚀 빠른 시작

### 1. 서버 준비

```bash
# Docker 설치
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Docker Compose 설치
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### 2. 프로젝트 배포

```bash
# 클론
git clone git@github.com:your-org/fmr-api.git
cd fmr-api

# 환경 변수 설정
cp .env.example .env
nano .env  # 실제 값으로 수정

# 빌드 및 실행
docker-compose up -d

# 모니터링 스택 실행
docker-compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d
```

### 3. DB 초기화

```bash
# 마이그레이션
docker-compose exec api alembic upgrade head

# Knowledge Base 시드 데이터
docker-compose exec api python scripts/seed_knowledge.py
```

### 4. 확인

```bash
# API 헬스 체크
curl http://localhost:8000/health

# Prometheus
curl http://localhost:9090

# Grafana
open http://localhost:3000
```

---

## ✅ 체크리스트

### 서버 준비

- [ ] Docker 설치
- [ ] Docker Compose 설치
- [ ] 방화벽 설정 (80, 443, 22)
- [ ] SSL 인증서 발급

### 배포

- [ ] 프로젝트 클론
- [ ] .env 파일 설정
- [ ] docker-compose.yml 확인
- [ ] Nginx 설정
- [ ] 빌드 및 실행
- [ ] DB 마이그레이션
- [ ] Knowledge Base 시드 데이터

### CI/CD

- [ ] GitHub Secrets 설정
- [ ] SSH 키 등록
- [ ] CI Workflow 테스트
- [ ] CD Workflow 테스트

### 모니터링

- [ ] Prometheus 설정
- [ ] Grafana 대시보드 구성
- [ ] Alertmanager 알림 설정
- [ ] Sentry 연동

---

## 📚 참고

- **Docker**: `docker.md`
- **CI/CD**: `ci-cd.md`
- **Monitoring**: `monitoring.md`
