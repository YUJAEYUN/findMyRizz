# Alembic 마이그레이션

## 📋 목표

Alembic을 사용하여 데이터베이스 마이그레이션을 설정합니다.

---

## 🔧 구현

### 1. Alembic 초기화

```bash
# Alembic 초기화
alembic init alembic
```

---

## 🔧 설정 파일

### 파일: `alembic.ini`

```ini
[alembic]
script_location = alembic
prepend_sys_path = .
sqlalchemy.url = postgresql://postgres:password@localhost:5432/fmr

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

---

## 🔧 env.py 설정

### 파일: `alembic/env.py`

```python
from logging.config import fileConfig
from sqlalchemy import engine_from_config
from sqlalchemy import pool
from alembic import context

# 설정 파일
config = context.config

# 로깅 설정
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# 모델 메타데이터
from app.core.database import Base
from app.models import job, payment, job_file, sms_log
from app.models import phone_verification, payment_failure
from app.models import knowledge, report, report_knowledge
from app.models import satisfaction_survey

target_metadata = Base.metadata

# 환경변수에서 DATABASE_URL 가져오기
from app.config import settings
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)


def run_migrations_offline() -> None:
    """오프라인 마이그레이션"""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    """온라인 마이그레이션"""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

---

## 🔧 마이그레이션 명령어

### 초기 마이그레이션 생성

```bash
# 마이그레이션 파일 생성
alembic revision --autogenerate -m "Initial migration"

# 마이그레이션 실행
alembic upgrade head
```

### 일반 명령어

```bash
# 현재 버전 확인
alembic current

# 마이그레이션 히스토리
alembic history

# 특정 버전으로 업그레이드
alembic upgrade <revision>

# 다운그레이드
alembic downgrade -1

# 최신 버전으로 업그레이드
alembic upgrade head

# 모든 마이그레이션 롤백
alembic downgrade base
```

---

## 🔧 마이그레이션 파일 예시

### 파일: `alembic/versions/001_initial.py`

```python
"""Initial migration

Revision ID: 001
Revises: 
Create Date: 2024-01-15 14:30:22.123456

"""
from alembic import op
import sqlalchemy as sa

# revision identifiers
revision = '001'
down_revision = None
branch_labels = None
depends_on = None


def upgrade() -> None:
    # jobs 테이블
    op.create_table(
        'jobs',
        sa.Column('id', sa.String(36), primary_key=True),
        sa.Column('user_phone_number', sa.String(11), nullable=False),
        sa.Column('status', sa.String(20), nullable=False),
        sa.Column('expires_at', sa.DateTime(), nullable=False),
        sa.Column('created_at', sa.DateTime(), nullable=False),
        sa.Column('updated_at', sa.DateTime(), nullable=False)
    )
    
    # 인덱스
    op.create_index('idx_jobs_phone', 'jobs', ['user_phone_number'])
    op.create_index('idx_jobs_status', 'jobs', ['status'])
    op.create_index('idx_jobs_expires_at', 'jobs', ['expires_at'])
    
    # payments 테이블
    op.create_table(
        'payments',
        sa.Column('id', sa.String(36), primary_key=True),
        sa.Column('job_id', sa.String(36), nullable=False),
        sa.Column('merchant_uid', sa.String(100), nullable=False),
        sa.Column('imp_uid', sa.String(100), nullable=True),
        sa.Column('amount', sa.Integer(), nullable=False),
        sa.Column('status', sa.String(20), nullable=False),
        sa.Column('paid_at', sa.DateTime(), nullable=True),
        sa.Column('created_at', sa.DateTime(), nullable=False),
        sa.Column('updated_at', sa.DateTime(), nullable=False),
        sa.ForeignKeyConstraint(['job_id'], ['jobs.id'], ondelete='CASCADE')
    )
    
    # ... 나머지 테이블들


def downgrade() -> None:
    op.drop_table('payments')
    op.drop_table('jobs')
    # ... 나머지 테이블들
```

---

## 🔧 Docker에서 실행

```bash
# docker-compose.yml에 추가
services:
  api:
    command: >
      sh -c "alembic upgrade head && 
             uvicorn main:app --host 0.0.0.0 --port 8000"
```

---

## ✅ 테스트

```bash
# 마이그레이션 생성 테스트
alembic revision --autogenerate -m "Test migration"

# 업그레이드 테스트
alembic upgrade head

# 다운그레이드 테스트
alembic downgrade -1

# 다시 업그레이드
alembic upgrade head
```

---

## 📝 체크리스트

- [ ] Alembic 초기화
- [ ] alembic.ini 설정
- [ ] env.py 설정
- [ ] 초기 마이그레이션 생성
- [ ] 마이그레이션 실행
- [ ] 테스트

---

## 🚀 다음 단계

Alembic 설정 완료 → **seed-data.md**

