# Docker 설정 (온프레미스)

## 📋 개요

**온프레미스 서버**에서 Docker 컨테이너로 애플리케이션을 실행합니다.

**호스팅 환경:**

- ✅ **온프레미스 서버** - FastAPI, PostgreSQL, Redis
- ✅ **AWS S3** - 이미지 저장 (외부 서비스)

---

## 🎯 목적

1. **온프레미스 배포** - 자체 서버에서 실행
2. **컨테이너화** - Docker로 환경 일관성 보장
3. **S3 연동** - 이미지는 S3에 저장
4. **확장성** - 필요 시 스케일 아웃 가능

---

## 🔧 Dockerfile

### 파일: `Dockerfile`

```dockerfile
FROM python:3.11-slim

# 작업 디렉토리
WORKDIR /app

# 시스템 패키지 설치
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Python 패키지 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 애플리케이션 코드 복사
COPY . .

# 헬스체크
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# 포트 노출
EXPOSE 8000

# 실행 명령
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

---

## 🔧 docker-compose.yml (온프레미스)

### 파일: `docker-compose.yml`

```yaml
version: "3.8"

services:
  # FastAPI 애플리케이션
  api:
    build: .
    container_name: fmr-api
    ports:
      - "8000:8000"
    environment:
      # Database (온프레미스)
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}

      # AWS S3 (이미지 저장용)
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - AWS_REGION=${AWS_REGION}
      - S3_BUCKET_NAME=${S3_BUCKET_NAME}

      # External APIs
      - PORTONE_API_KEY=${PORTONE_API_KEY}
      - PORTONE_API_SECRET=${PORTONE_API_SECRET}
      - REPLICATE_API_TOKEN=${REPLICATE_API_TOKEN}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - COOLSMS_API_KEY=${COOLSMS_API_KEY}
      - COOLSMS_API_SECRET=${COOLSMS_API_SECRET}

      # LangChain (선택)
      - LANGCHAIN_API_KEY=${LANGCHAIN_API_KEY}
      - LANGCHAIN_PROJECT=${LANGCHAIN_PROJECT}

      # Celery + Redis
      - CELERY_BROKER_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
      - CELERY_RESULT_BACKEND=redis://:${REDIS_PASSWORD}@redis:6379/0
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0

      # Timezone
      - TIMEZONE=Asia/Seoul

      # Application
      - ENVIRONMENT=production
      - LOG_LEVEL=info
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - ./logs:/app/logs
      - /etc/localtime:/etc/localtime:ro # 서버 시간 동기화
    restart: unless-stopped
    networks:
      - fmr-network

  # PostgreSQL (온프레미스)
  db:
    image: postgres:15-alpine
    container_name: fmr-db
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_INITDB_ARGS=--encoding=UTF-8 --locale=C
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - fmr-network

  # Redis (온프레미스 - Celery/캐싱용)
  redis:
    image: redis:7-alpine
    container_name: fmr-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - fmr-network

  # Celery Worker
  celery-worker:
    build: .
    container_name: fmr-celery-worker
    command: celery -A app.core.celery_app worker --loglevel=info
    environment:
      # Database (온프레미스)
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}

      # AWS S3 (이미지 저장용)
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - AWS_REGION=${AWS_REGION}
      - S3_BUCKET_NAME=${S3_BUCKET_NAME}

      # External APIs
      - PORTONE_API_KEY=${PORTONE_API_KEY}
      - PORTONE_API_SECRET=${PORTONE_API_SECRET}
      - REPLICATE_API_TOKEN=${REPLICATE_API_TOKEN}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - COOLSMS_API_KEY=${COOLSMS_API_KEY}
      - COOLSMS_API_SECRET=${COOLSMS_API_SECRET}

      # LangChain (선택)
      - LANGCHAIN_API_KEY=${LANGCHAIN_API_KEY}
      - LANGCHAIN_PROJECT=${LANGCHAIN_PROJECT}

      # Celery + Redis
      - CELERY_BROKER_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
      - CELERY_RESULT_BACKEND=redis://:${REDIS_PASSWORD}@redis:6379/0
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0

      # Timezone
      - TIMEZONE=Asia/Seoul

      # Application
      - ENVIRONMENT=production
      - LOG_LEVEL=info
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - ./logs:/app/logs
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    networks:
      - fmr-network

  # Celery Beat (스케줄러)
  celery-beat:
    build: .
    container_name: fmr-celery-beat
    command: celery -A app.core.celery_app beat --loglevel=info
    environment:
      # Database (온프레미스)
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}

      # Celery + Redis
      - CELERY_BROKER_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
      - CELERY_RESULT_BACKEND=redis://:${REDIS_PASSWORD}@redis:6379/0
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0

      # Timezone
      - TIMEZONE=Asia/Seoul

      # Application
      - ENVIRONMENT=production
      - LOG_LEVEL=info
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - ./logs:/app/logs
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    networks:
      - fmr-network

  # Nginx (리버스 프록시)
  nginx:
    image: nginx:alpine
    container_name: fmr-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./logs/nginx:/var/log/nginx
    depends_on:
      - api
    restart: unless-stopped
    networks:
      - fmr-network

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  fmr-network:
    driver: bridge
```

---

## 🔧 환경 변수 (.env)

### 파일: `.env.example`

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
```

---

## 🔧 Nginx 설정

### 파일: `nginx/nginx.conf`

```nginx
events {
    worker_connections 1024;
}

http {
    upstream api {
        server api:8000;
    }

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

    server {
        listen 80;
        server_name your-domain.com;

        # Redirect to HTTPS
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name your-domain.com;

        # SSL 인증서
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;

        # SSL 설정
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        # 로그
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        # API 프록시
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;

            proxy_pass http://api;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Timeout 설정
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        # Health check
        location /health {
            proxy_pass http://api/health;
            access_log off;
        }
    }
}
```

---

## 🔧 .dockerignore

### 파일: `.dockerignore`

```
__pycache__
*.pyc
*.pyo
*.pyd
.Python
env/
venv/
.venv/
.git
.gitignore
.env
.env.local
*.log
logs/
.pytest_cache
.coverage
htmlcov/
dist/
build/
*.egg-info
```

---

## 🚀 온프레미스 서버 배포

### 1. 서버 준비

```bash
# Docker 설치 (Ubuntu/Debian)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Docker Compose 설치
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 확인
docker --version
docker-compose --version
```

### 2. 프로젝트 배포

```bash
# 프로젝트 클론
git clone git@github.com:your-org/fmr-api.git
cd fmr-api

# 환경 변수 설정
cp .env.example .env
nano .env  # 실제 값으로 수정

# 빌드 및 실행
docker-compose up -d

# 로그 확인
docker-compose logs -f api
```

### 3. DB 초기화

```bash
# 마이그레이션
docker-compose exec api alembic upgrade head

# Knowledge Base 시드 데이터
docker-compose exec api python scripts/seed_knowledge.py

# 확인
docker-compose exec db psql -U fmr_user -d fmr_production -c "SELECT COUNT(*) FROM kb_items;"
```

### 4. SSL 인증서 설정

```bash
# Let's Encrypt (Certbot)
sudo apt-get install certbot

# 인증서 발급
sudo certbot certonly --standalone -d your-domain.com

# Nginx SSL 디렉토리에 복사
sudo cp /etc/letsencrypt/live/your-domain.com/fullchain.pem nginx/ssl/cert.pem
sudo cp /etc/letsencrypt/live/your-domain.com/privkey.pem nginx/ssl/key.pem

# Nginx 재시작
docker-compose restart nginx
```

---

## 🔧 운영 명령어

### 일상 운영

```bash
# 상태 확인
docker-compose ps

# 로그 확인
docker-compose logs -f api
docker-compose logs -f celery-worker
docker-compose logs -f celery-beat
docker-compose logs -f db
docker-compose logs -f nginx

# 재시작
docker-compose restart api
docker-compose restart celery-worker
docker-compose restart celery-beat

# 업데이트 배포
git pull origin main
docker-compose build api
docker-compose up -d api
```

### 백업

```bash
# DB 백업
docker-compose exec db pg_dump -U fmr_user fmr_production > backup_$(date +%Y%m%d).sql

# 복원
docker-compose exec -T db psql -U fmr_user fmr_production < backup_20240101.sql
```

### 모니터링

```bash
# 리소스 사용량
docker stats

# 디스크 사용량
docker system df

# 컨테이너 상태
docker-compose ps
```

---

## ⚠️ 주의사항

### 1. S3 연동 확인

```bash
# S3 접근 테스트
docker-compose exec api python -c "
import boto3
s3 = boto3.client('s3')
print(s3.list_buckets())
"
```

### 2. 방화벽 설정

```bash
# 필요한 포트만 오픈
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow 22/tcp    # SSH
sudo ufw enable
```

### 3. 로그 로테이션

```bash
# /etc/logrotate.d/fmr-api
/path/to/fmr-api/logs/*.log {
    daily
    rotate 30
    compress
    delaycompress
    notifempty
    create 0640 root root
    sharedscripts
    postrotate
        docker-compose restart api
    endscript
}
```

---

## ✅ 체크리스트

- [ ] Docker 설치
- [ ] 프로젝트 클론
- [ ] .env 설정 (S3 키 포함)
- [ ] docker-compose.yml 확인
- [ ] Nginx 설정
- [ ] SSL 인증서 설정
- [ ] 빌드 및 실행
- [ ] DB 마이그레이션
- [ ] Knowledge Base 시드 데이터
- [ ] S3 연동 테스트
- [ ] 방화벽 설정
- [ ] 로그 로테이션 설정
- [ ] 백업 스크립트 설정

---

## 📚 참고

- **다음 문서**: `ci-cd.md` - 온프레미스 CI/CD
- **모니터링**: `monitoring.md` - Prometheus/Grafana
