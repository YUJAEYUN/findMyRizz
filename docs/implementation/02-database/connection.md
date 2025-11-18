# DB 연결 설정

## 📋 목표

SQLAlchemy를 사용하여 PostgreSQL에 연결합니다.

---

## 🔧 구현

### 파일: `app/core/database.py`

```python
"""
데이터베이스 연결 및 세션 관리
"""
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from typing import Generator
import logging

from app.config import settings

logger = logging.getLogger(__name__)

# ===== SQLAlchemy 엔진 생성 =====
# DATABASE_URL 형식: postgresql://user:password@host:port/dbname
engine = create_engine(
    settings.DATABASE_URL,
    echo=settings.DB_ECHO,  # SQL 쿼리 로깅 (개발: True, 프로덕션: False)
    pool_size=10,           # 커넥션 풀 크기
    max_overflow=20,        # 최대 추가 커넥션
    pool_pre_ping=True,     # 커넥션 유효성 체크
    pool_recycle=3600       # 1시간마다 커넥션 재생성
)

# ===== 세션 팩토리 =====
SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine
)

# ===== Base 클래스 =====
Base = declarative_base()


def get_db() -> Generator[Session, None, None]:
    """
    FastAPI 의존성 주입용 DB 세션 생성

    사용 예:
        @app.get("/items")
        def get_items(db: Session = Depends(get_db)):
            return db.query(Item).all()

    Yields:
        Session: SQLAlchemy 세션
    """
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


def init_db():
    """
    데이터베이스 초기화

    - 모든 테이블 생성 (개발 환경에서만 사용)
    - 프로덕션에서는 Alembic 마이그레이션 사용
    """
    # 모든 모델 import (테이블 생성을 위해)
    from app.models import job, payment, job_file, sms_log
    from app.models import phone_verification, payment_failure
    from app.models import knowledge, report

    # 테이블 생성
    Base.metadata.create_all(bind=engine)
    logger.info("Database tables created successfully")


def check_db_connection() -> bool:
    """
    DB 연결 확인

    Returns:
        bool: 연결 성공 여부
    """
    try:
        db = SessionLocal()
        db.execute("SELECT 1")
        db.close()
        logger.info("Database connection successful")
        return True
    except Exception as e:
        logger.error(f"Database connection failed: {e}")
        return False
```

---

## 🔧 Base 모델 Mixin

### 파일: `app/models/base.py`

```python
"""
모든 모델이 상속받는 Base Mixin
"""
from sqlalchemy import Column, DateTime
from datetime import datetime
from zoneinfo import ZoneInfo

from app.config import settings


def get_kst_now():
    """
    KST 현재 시간 반환

    Returns:
        datetime: KST 시간대의 현재 시간
    """
    return datetime.now(ZoneInfo(settings.TIMEZONE))


class TimestampMixin:
    """
    created_at, updated_at 자동 관리 (KST 기준)
    """
    created_at = Column(
        DateTime(timezone=True),
        nullable=False,
        default=get_kst_now
    )
    updated_at = Column(
        DateTime(timezone=True),
        nullable=False,
        default=get_kst_now,
        onupdate=get_kst_now
    )
```

---

## 🔧 main.py에 추가

### 파일: `main.py`

```python
from app.core.database import check_db_connection, init_db

@app.on_event("startup")
async def startup_event():
    """
    애플리케이션 시작 시 실행
    """
    # DB 연결 확인
    if not check_db_connection():
        raise Exception("Database connection failed")

    # 개발 환경에서만 테이블 자동 생성
    if settings.APP_ENV == "development":
        init_db()

    logger.info("Application started successfully")


@app.on_event("shutdown")
async def shutdown_event():
    """
    애플리케이션 종료 시 실행
    """
    logger.info("Application shutting down")
```

---

## ✅ 테스트

### 1. DB 연결 테스트

```python
# test_db_connection.py
from app.core.database import check_db_connection

if __name__ == "__main__":
    if check_db_connection():
        print("✅ DB 연결 성공")
    else:
        print("❌ DB 연결 실패")
```

```bash
python test_db_connection.py
```

### 2. 세션 테스트

```python
from app.core.database import get_db

db = next(get_db())
result = db.execute("SELECT version()")
print(result.fetchone())
db.close()
```

---

## 📝 체크리스트

- [ ] PostgreSQL 설치 및 실행
- [ ] DATABASE_URL 환경변수 설정
- [ ] app/core/database.py 생성
- [ ] app/models/base.py 생성
- [ ] DB 연결 테스트 성공
- [ ] main.py에 startup 이벤트 추가

---

## 🚀 다음 단계

DB 연결 완료 → **models/\*.md** (모델 정의)
