# 환경 변수 스펙

## 1. 필수 환경 변수

### 1.1 데이터베이스

```bash
# PostgreSQL
DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/fmr_db
DATABASE_POOL_SIZE=20
DATABASE_MAX_OVERFLOW=10
```

### 1.2 Redis

```bash
# Redis (Celery Broker & Cache)
REDIS_URL=redis://localhost:6379/0
CELERY_BROKER_URL=redis://localhost:6379/1
CELERY_RESULT_BACKEND=redis://localhost:6379/2
```

### 1.3 AWS S3

```bash
# AWS Credentials
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_REGION=ap-northeast-2

# S3 Bucket (Private Bucket + Presigned URL 방식)
AWS_S3_BUCKET=findmyrizz-files
S3_BUCKET_POLICY=private  # private (권장) 또는 public
S3_PRESIGNED_URL_EXPIRATION=3600  # 1시간 (초 단위, private일 때만 사용)
```

### 1.4 PortOne V2

```bash
# PortOne V2
PORTONE_STORE_ID=store-4ff4af41-85e3-4559-8eb8-0d08a2c6ceec
PORTONE_CHANNEL_KEY=channel-key-9987cb87-6458-4888-b94e-68d9a2da896d
PORTONE_API_SECRET=your_v2_api_secret
PORTONE_PG_PROVIDER=welcome
```

### 1.5 Replicate (AI 이미지 생성)

```bash
# Replicate API
REPLICATE_API_TOKEN=your_replicate_token
REPLICATE_MODEL_VERSION=ai_image_generation_model_version_id
REPLICATE_WEBHOOK_URL=https://api.findmyrizz.com/api/v1/webhooks/replicate
```

### 1.6 OpenAI

```bash
# OpenAI GPT-4o
OPENAI_API_KEY=sk-proj-...
OPENAI_MODEL=gpt-4o
OPENAI_MAX_TOKENS=2000
OPENAI_TEMPERATURE=0.7
```

### 1.7 LangGraph/LangSmith

```bash
# LangSmith Tracing (LangGraph 모니터링)
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=your_langsmith_api_key
LANGCHAIN_PROJECT=find-my-rizz
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com

# LangGraph Settings
LANGCHAIN_VERBOSE=false
LANGGRAPH_RECURSION_LIMIT=25  # LangGraph 최대 재귀 깊이
```

### 1.8 CoolSMS

```bash
# CoolSMS
COOLSMS_API_KEY=your_coolsms_api_key
COOLSMS_API_SECRET=your_coolsms_api_secret
COOLSMS_SENDER_NUMBER=01012345678
```

### 1.9 보안

```bash
# JWT
JWT_SECRET_KEY=your_super_secret_jwt_key_min_32_characters
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_HOURS=1

# API Rate Limiting
RATE_LIMIT_PER_MINUTE=100
PHONE_VERIFICATION_MAX_ATTEMPTS=3
PHONE_VERIFICATION_BLOCK_HOURS=1
```

### 1.10 애플리케이션

```bash
# FastAPI
APP_NAME=Find My Rizz API
APP_VERSION=1.0.0
APP_ENV=development  # development, staging, production
DEBUG=true

# CORS
CORS_ORIGINS=http://localhost:3000,https://findmyrizz.com
CORS_ALLOW_CREDENTIALS=true

# Timezone
TZ=Asia/Seoul
```

### 1.11 모니터링

```bash
# Sentry
SENTRY_DSN=https://your_sentry_dsn@sentry.io/project_id
SENTRY_ENVIRONMENT=production
SENTRY_TRACES_SAMPLE_RATE=0.1

# Logging
LOG_LEVEL=INFO  # DEBUG, INFO, WARNING, ERROR, CRITICAL
LOG_FORMAT=json  # json, text
```

---

## 2. 환경별 설정

### 2.1 Development (.env.development)

```bash
APP_ENV=development
DEBUG=true
LOG_LEVEL=DEBUG

DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/fmr_dev
REDIS_URL=redis://localhost:6379/0

CORS_ORIGINS=http://localhost:3000,http://localhost:5173
```

### 2.2 Staging (.env.staging)

```bash
APP_ENV=staging
DEBUG=false
LOG_LEVEL=INFO

DATABASE_URL=postgresql+asyncpg://user:password@staging-db:5432/fmr_staging
REDIS_URL=redis://staging-redis:6379/0

CORS_ORIGINS=https://staging.findmyrizz.com
```

### 2.3 Production (.env.production)

```bash
APP_ENV=production
DEBUG=false
LOG_LEVEL=WARNING

DATABASE_URL=postgresql+asyncpg://user:password@prod-db:5432/fmr_prod
REDIS_URL=redis://prod-redis:6379/0

CORS_ORIGINS=https://findmyrizz.com
SENTRY_TRACES_SAMPLE_RATE=0.01
```

---

## 3. 설정 파일 구조

### 3.1 Pydantic Settings

```python
# app/core/config.py

from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    # App
    APP_NAME: str = "Find My Rizz API"
    APP_VERSION: str = "1.0.0"
    APP_ENV: str = "development"
    DEBUG: bool = True

    # Database
    DATABASE_URL: str
    DATABASE_POOL_SIZE: int = 20
    DATABASE_MAX_OVERFLOW: int = 10

    # Redis
    REDIS_URL: str
    CELERY_BROKER_URL: str
    CELERY_RESULT_BACKEND: str

    # AWS
    AWS_ACCESS_KEY_ID: str
    AWS_SECRET_ACCESS_KEY: str
    AWS_REGION: str = "ap-northeast-2"
    AWS_S3_BUCKET: str
    S3_BUCKET_POLICY: str = "private"  # private 또는 public
    S3_PRESIGNED_URL_EXPIRATION: int = 3600

    # PortOne
    PORTONE_STORE_ID: str
    PORTONE_CHANNEL_KEY: str
    PORTONE_API_SECRET: str
    PORTONE_PG_PROVIDER: str = "welcome"

    # Replicate
    REPLICATE_API_TOKEN: str
    REPLICATE_MODEL_VERSION: str
    REPLICATE_WEBHOOK_URL: str

    # OpenAI
    OPENAI_API_KEY: str
    OPENAI_MODEL: str = "gpt-4o"
    OPENAI_MAX_TOKENS: int = 2000
    OPENAI_TEMPERATURE: float = 0.7

    # LangGraph/LangSmith
    LANGCHAIN_TRACING_V2: bool = True
    LANGCHAIN_API_KEY: str
    LANGCHAIN_PROJECT: str = "find-my-rizz"
    LANGCHAIN_ENDPOINT: str = "https://api.smith.langchain.com"
    LANGCHAIN_VERBOSE: bool = False
    LANGGRAPH_RECURSION_LIMIT: int = 25

    # CoolSMS
    COOLSMS_API_KEY: str
    COOLSMS_API_SECRET: str
    COOLSMS_SENDER_NUMBER: str

    # Security
    JWT_SECRET_KEY: str
    JWT_ALGORITHM: str = "HS256"
    JWT_ACCESS_TOKEN_EXPIRE_HOURS: int = 1

    # Rate Limiting
    RATE_LIMIT_PER_MINUTE: int = 100
    PHONE_VERIFICATION_MAX_ATTEMPTS: int = 3
    PHONE_VERIFICATION_BLOCK_HOURS: int = 1

    # CORS
    CORS_ORIGINS: List[str] = ["http://localhost:3000"]
    CORS_ALLOW_CREDENTIALS: bool = True

    # Sentry
    SENTRY_DSN: str | None = None
    SENTRY_ENVIRONMENT: str = "development"
    SENTRY_TRACES_SAMPLE_RATE: float = 0.1

    # Logging
    LOG_LEVEL: str = "INFO"
    LOG_FORMAT: str = "json"

    class Config:
        env_file = ".env"
        case_sensitive = True

settings = Settings()
```

---

## 4. 보안 주의사항

### 4.1 절대 커밋하지 말 것

- `.env` 파일
- `.env.production` 파일
- API 키, Secret 키

### 4.2 .gitignore 설정

```gitignore
# Environment
.env
.env.*
!.env.example

# Secrets
*.pem
*.key
secrets/
```

### 4.3 .env.example 제공

```bash
# .env.example (실제 값 없이 구조만)

DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/dbname
REDIS_URL=redis://localhost:6379/0
AWS_ACCESS_KEY_ID=your_access_key
PORTONE_API_SECRET=your_secret
...
```

---

## 5. 참고 문서

- [Pydantic Settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
- [12 Factor App - Config](https://12factor.net/config)
