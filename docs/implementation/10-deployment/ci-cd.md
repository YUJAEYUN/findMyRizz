# CI/CD 파이프라인 (온프레미스)

## 📋 개요

**온프레미스 서버**로 자동 배포하는 CI/CD 파이프라인

**배포 흐름:**

1. GitHub Actions - CI (테스트, 린트)
2. SSH를 통해 온프레미스 서버로 배포
3. Docker Compose로 무중단 배포

---

## 🎯 목적

1. **자동화된 테스트** - PR/Push 시 자동 테스트
2. **온프레미스 배포** - SSH로 서버 접속 후 배포
3. **무중단 배포** - Rolling update
4. **롤백 지원** - 이전 버전으로 복구 가능

---

## 🔧 CI Workflow (테스트)

### 파일: `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
          POSTGRES_DB: fmr_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.11"

      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov pytest-asyncio

      - name: Run tests
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/fmr_test
          AWS_ACCESS_KEY_ID: test
          AWS_SECRET_ACCESS_KEY: test
          AWS_REGION: ap-northeast-2
          S3_BUCKET_NAME: test-bucket
          PORTONE_API_KEY: test
          REPLICATE_API_TOKEN: test
          OPENAI_API_KEY: test
          COOLSMS_API_KEY: test
        run: |
          pytest tests/ -v --cov=app --cov-report=xml --cov-report=term

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          fail_ci_if_error: false

  lint:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.11"

      - name: Install linters
        run: |
          pip install flake8 black isort mypy

      - name: Run flake8
        run: flake8 app/ --max-line-length=100 --exclude=__pycache__

      - name: Run black
        run: black --check app/

      - name: Run isort
        run: isort --check-only app/
```

---

## 🚀 CD Workflow (온프레미스 배포)

### 파일: `.github/workflows/deploy.yml`

```yaml
name: Deploy to On-Premise

on:
  push:
    branches: [main]
  workflow_dispatch: # 수동 실행 가능

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.8.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Add server to known hosts
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -H ${{ secrets.SERVER_HOST }} >> ~/.ssh/known_hosts

      - name: Deploy to server
        env:
          SERVER_HOST: ${{ secrets.SERVER_HOST }}
          SERVER_USER: ${{ secrets.SERVER_USER }}
          DEPLOY_PATH: ${{ secrets.DEPLOY_PATH }}
        run: |
          ssh $SERVER_USER@$SERVER_HOST << 'EOF'
            set -e

            # 배포 디렉토리로 이동
            cd ${{ secrets.DEPLOY_PATH }}

            # Git pull
            git fetch origin
            git reset --hard origin/main

            # 환경 변수 확인
            if [ ! -f .env ]; then
              echo "Error: .env file not found"
              exit 1
            fi

            # Docker 이미지 빌드
            docker-compose build api

            # 무중단 배포 (Rolling update)
            docker-compose up -d --no-deps --build api

            # 헬스 체크
            sleep 10
            curl -f http://localhost:8000/health || exit 1

            # 이전 이미지 정리
            docker image prune -f

            echo "Deployment completed successfully"
          EOF

      - name: Notify deployment
        if: success()
        run: |
          echo "Deployment to on-premise server completed"

      - name: Rollback on failure
        if: failure()
        env:
          SERVER_HOST: ${{ secrets.SERVER_HOST }}
          SERVER_USER: ${{ secrets.SERVER_USER }}
          DEPLOY_PATH: ${{ secrets.DEPLOY_PATH }}
        run: |
          ssh $SERVER_USER@$SERVER_HOST << 'EOF'
            cd ${{ secrets.DEPLOY_PATH }}

            # 이전 커밋으로 롤백
            git reset --hard HEAD~1
            docker-compose up -d --no-deps --build api

            echo "Rollback completed"
          EOF
```

---

## 🔧 GitHub Secrets 설정

### Repository Settings → Secrets and variables → Actions

```bash
# SSH 접속 정보
SSH_PRIVATE_KEY=<온프레미스 서버 SSH 개인키>
SERVER_HOST=<서버 IP 또는 도메인>
SERVER_USER=<SSH 사용자명>
DEPLOY_PATH=/home/user/fmr-api

# 환경 변수 (선택 - 서버에 .env 파일이 있으면 불필요)
DB_PASSWORD=<DB 비밀번호>
AWS_ACCESS_KEY_ID=<S3 액세스 키>
AWS_SECRET_ACCESS_KEY=<S3 시크릿 키>
```

---

## 🔧 서버 준비

### 1. SSH 키 생성 및 등록

```bash
# GitHub Actions용 SSH 키 생성
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github_actions

# 공개키를 서버에 등록
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys

# 개인키를 GitHub Secrets에 등록
cat ~/.ssh/github_actions  # 이 내용을 SSH_PRIVATE_KEY에 등록
```

### 2. 배포 스크립트 (서버)

**파일: `scripts/deploy.sh`**

```bash
#!/bin/bash
set -e

echo "Starting deployment..."

# Git pull
git fetch origin
git reset --hard origin/main

# 환경 변수 확인
if [ ! -f .env ]; then
    echo "Error: .env file not found"
    exit 1
fi

# 백업 생성
BACKUP_DIR="backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR
docker-compose exec -T db pg_dump -U fmr_user fmr_production > $BACKUP_DIR/db_backup.sql

# Docker 이미지 빌드
docker-compose build api

# 무중단 배포
docker-compose up -d --no-deps --build api

# 헬스 체크
echo "Waiting for service to be ready..."
sleep 10

MAX_RETRIES=30
RETRY_COUNT=0

while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
    if curl -f http://localhost:8000/health > /dev/null 2>&1; then
        echo "Service is healthy"
        break
    fi

    RETRY_COUNT=$((RETRY_COUNT + 1))
    echo "Retry $RETRY_COUNT/$MAX_RETRIES..."
    sleep 2
done

if [ $RETRY_COUNT -eq $MAX_RETRIES ]; then
    echo "Health check failed. Rolling back..."
    git reset --hard HEAD~1
    docker-compose up -d --no-deps --build api
    exit 1
fi

# 이전 이미지 정리
docker image prune -f

echo "Deployment completed successfully"
```

---

## ✅ 배포 테스트

```bash
# 로컬에서 배포 스크립트 테스트
chmod +x scripts/deploy.sh
./scripts/deploy.sh

# GitHub Actions 수동 실행
# Repository → Actions → Deploy to On-Premise → Run workflow
```

---

## 🔧 pytest 설정

### 파일: `pytest.ini`

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    -v
    --strict-markers
    --tb=short
    --cov=app
    --cov-report=term-missing
    --cov-report=html
```

---

## 🔧 테스트 구조

```
tests/
├── conftest.py           # Fixtures
├── test_payment.py       # 결제 테스트
├── test_image_upload.py  # 이미지 업로드 테스트
├── test_ai_generation.py # AI 생성 테스트
└── test_report.py        # 리포트 테스트
```

### 파일: `tests/conftest.py`

```python
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from main import app
from app.core.database import Base, get_db

# 테스트 DB
SQLALCHEMY_DATABASE_URL = "postgresql://postgres:password@localhost:5432/fmr_test"

engine = create_engine(SQLALCHEMY_DATABASE_URL)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)


@pytest.fixture(scope="function")
def db():
    """테스트 DB 세션"""
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()
        Base.metadata.drop_all(bind=engine)


@pytest.fixture(scope="function")
def client(db):
    """테스트 클라이언트"""
    def override_get_db():
        try:
            yield db
        finally:
            pass

    app.dependency_overrides[get_db] = override_get_db
    yield TestClient(app)
    app.dependency_overrides.clear()
```

---

## ✅ 로컬 테스트

```bash
# 전체 테스트
pytest

# 특정 파일
pytest tests/test_payment.py

# 커버리지 포함
pytest --cov=app --cov-report=html

# 커버리지 리포트 확인
open htmlcov/index.html
```

---

## 📝 체크리스트

### CI 설정

- [ ] .github/workflows/ci.yml 생성
- [ ] pytest.ini 설정
- [ ] tests/conftest.py 생성
- [ ] 테스트 작성

### CD 설정 (온프레미스)

- [ ] .github/workflows/deploy.yml 생성
- [ ] scripts/deploy.sh 생성
- [ ] SSH 키 생성 및 등록
- [ ] GitHub Secrets 설정
- [ ] 서버에 .env 파일 설정
- [ ] 배포 테스트

---

## 📚 참고

- **이전 문서**: `docker.md` - Docker 설정
- **다음 문서**: `monitoring.md` - Prometheus/Grafana
