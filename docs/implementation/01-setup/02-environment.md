# 환경변수 설정

## 📋 목표

모든 API 키와 설정값을 `.env` 파일로 관리합니다.

---

## 🔧 Step 1: .env 파일 생성

### 파일: `.env`

```bash
# ===== 애플리케이션 설정 =====
APP_NAME=FindMyRizz
APP_ENV=development
DEBUG=True
SECRET_KEY=your-secret-key-here-change-in-production

# ===== 서버 설정 =====
HOST=0.0.0.0
PORT=8000

# ===== 데이터베이스 =====
DATABASE_URL=postgresql://user:password@localhost:5432/fmr_db
DB_ECHO=False

# ===== Celery + Redis =====
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0
REDIS_URL=redis://localhost:6379/0

# ===== Timezone =====
TIMEZONE=Asia/Seoul

# ===== AWS S3 =====
AWS_ACCESS_KEY_ID=your-aws-access-key
AWS_SECRET_ACCESS_KEY=your-aws-secret-key
AWS_REGION=ap-northeast-2
S3_BUCKET_NAME=fmr-uploads

# ===== PortOne V2 (결제) =====
PORTONE_STORE_ID=store-4ff4af41-85e3-4559-8eb8-0d08a2c6ceec
PORTONE_CHANNEL_KEY=channel-key-9987cb87-6458-4888-b94e-68d9a2da896d
PORTONE_API_SECRET=your-v2-api-secret
PORTONE_PG_PROVIDER=welcome

# ===== Replicate (AI 이미지 생성) =====
REPLICATE_API_TOKEN=your-replicate-api-token
REPLICATE_WEBHOOK_URL=https://your-domain.com/api/v1/webhooks/replicate

# ===== OpenAI (리포트 생성) =====
OPENAI_API_KEY=your-openai-api-key
OPENAI_MODEL=gpt-4o
OPENAI_MAX_TOKENS=2000

# ===== CoolSMS (SMS 발송) =====
COOLSMS_API_KEY=your-coolsms-api-key
COOLSMS_API_SECRET=your-coolsms-api-secret
COOLSMS_SENDER_NUMBER=01012345678

# ===== 비즈니스 로직 설정 =====
PAYMENT_AMOUNT=9900
JOB_EXPIRATION_HOURS=24
PHONE_VERIFICATION_MAX_ATTEMPTS=3
PHONE_VERIFICATION_BLOCK_DURATION_MINUTES=60
PRESIGNED_URL_EXPIRATION_SECONDS=3600

# ===== 보안 설정 =====
RATE_LIMIT_PER_MINUTE=60
ALLOWED_ORIGINS=http://localhost:3000,https://your-domain.com

# ===== 로깅 =====
LOG_LEVEL=INFO
LOG_FILE=logs/app.log
```

---

## 🔧 Step 2: .env.example 생성

### 파일: `.env.example`

```bash
# .env 파일 템플릿 (실제 값은 .env에 작성)

APP_NAME=FindMyRizz
APP_ENV=development
DEBUG=True
SECRET_KEY=change-me

DATABASE_URL=postgresql://user:password@localhost:5432/fmr_db

CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0
TIMEZONE=Asia/Seoul

AWS_ACCESS_KEY_ID=your-aws-access-key
AWS_SECRET_ACCESS_KEY=your-aws-secret-key
AWS_REGION=ap-northeast-2
S3_BUCKET_NAME=fmr-uploads

PORTONE_STORE_ID=store-4ff4af41-85e3-4559-8eb8-0d08a2c6ceec
PORTONE_CHANNEL_KEY=channel-key-9987cb87-6458-4888-b94e-68d9a2da896d
PORTONE_API_SECRET=your-v2-api-secret
PORTONE_PG_PROVIDER=welcome

REPLICATE_API_TOKEN=your-replicate-api-token
OPENAI_API_KEY=your-openai-api-key
COOLSMS_API_KEY=your-coolsms-api-key

PAYMENT_AMOUNT=9900
JOB_EXPIRATION_HOURS=24
```

---

## 🔧 Step 3: .gitignore 생성

### 파일: `.gitignore`

```
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
venv/
env/
ENV/

# 환경변수
.env
.env.local

# IDE
.vscode/
.idea/
*.swp
*.swo

# 로그
logs/
*.log

# 데이터베이스
*.db
*.sqlite3

# OS
.DS_Store
Thumbs.db

# 테스트
.pytest_cache/
.coverage
htmlcov/

# 빌드
dist/
build/
*.egg-info/
```

---

## 🔧 Step 4: config.py 생성

### 파일: `app/config.py`

```python
"""
애플리케이션 설정
환경변수를 로드하고 검증합니다.
"""
from pydantic_settings import BaseSettings
from typing import List


class Settings(BaseSettings):
    """
    애플리케이션 설정 클래스

    .env 파일에서 환경변수를 자동으로 로드합니다.
    """

    # ===== 애플리케이션 =====
    APP_NAME: str = "FindMyRizz"
    APP_ENV: str = "development"
    DEBUG: bool = True
    SECRET_KEY: str

    # ===== 서버 =====
    HOST: str = "0.0.0.0"
    PORT: int = 8000

    # ===== 데이터베이스 =====
    DATABASE_URL: str
    DB_ECHO: bool = False

    # ===== Celery + Redis =====
    CELERY_BROKER_URL: str = "redis://localhost:6379/0"
    CELERY_RESULT_BACKEND: str = "redis://localhost:6379/0"
    REDIS_URL: str = "redis://localhost:6379/0"

    # ===== Timezone =====
    TIMEZONE: str = "Asia/Seoul"

    # ===== AWS S3 =====
    AWS_ACCESS_KEY_ID: str
    AWS_SECRET_ACCESS_KEY: str
    AWS_REGION: str = "ap-northeast-2"
    S3_BUCKET_NAME: str

    # ===== PortOne V2 =====
    PORTONE_STORE_ID: str
    PORTONE_CHANNEL_KEY: str
    PORTONE_API_SECRET: str
    PORTONE_PG_PROVIDER: str = "welcome"

    # ===== Replicate =====
    REPLICATE_API_TOKEN: str
    REPLICATE_WEBHOOK_URL: str

    # ===== OpenAI =====
    OPENAI_API_KEY: str
    OPENAI_MODEL: str = "gpt-4o"
    OPENAI_MAX_TOKENS: int = 2000

    # ===== CoolSMS =====
    COOLSMS_API_KEY: str
    COOLSMS_API_SECRET: str
    COOLSMS_SENDER_NUMBER: str

    # ===== 비즈니스 로직 =====
    PAYMENT_AMOUNT: int = 9900
    JOB_EXPIRATION_HOURS: int = 24
    PHONE_VERIFICATION_MAX_ATTEMPTS: int = 3
    PHONE_VERIFICATION_BLOCK_DURATION_MINUTES: int = 60
    PRESIGNED_URL_EXPIRATION_SECONDS: int = 3600

    # ===== 보안 =====
    RATE_LIMIT_PER_MINUTE: int = 60
    ALLOWED_ORIGINS: str = "http://localhost:3000"

    # ===== 로깅 =====
    LOG_LEVEL: str = "INFO"
    LOG_FILE: str = "logs/app.log"

    class Config:
        env_file = ".env"
        case_sensitive = True


# 싱글톤 인스턴스
settings = Settings()
```

---

## ✅ 테스트

### 1. 환경변수 로드 테스트

```python
# test_config.py
from app.config import settings

print(f"APP_NAME: {settings.APP_NAME}")
print(f"DATABASE_URL: {settings.DATABASE_URL}")
print(f"PAYMENT_AMOUNT: {settings.PAYMENT_AMOUNT}")
```

```bash
python test_config.py
```

**예상 출력:**

```
APP_NAME: FindMyRizz
DATABASE_URL: postgresql://user:password@localhost:5432/fmr_db
PAYMENT_AMOUNT: 9900
```

---

## 📝 체크리스트

- [ ] .env 파일 생성
- [ ] .env.example 파일 생성
- [ ] .gitignore 파일 생성
- [ ] app/config.py 생성
- [ ] pydantic-settings 설치 (`pip install pydantic-settings`)
- [ ] 환경변수 로드 테스트 성공

---

## 🚀 다음 단계

환경변수 설정 완료 → **03-dependencies.md** (의존성 패키지)
