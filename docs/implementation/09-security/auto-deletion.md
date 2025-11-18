# 24시간 자동 삭제

## 📋 목표

만료된 Job을 자동으로 삭제하는 Celery Beat Task를 구현합니다.

---

## 🔧 구현

### 파일: `app/tasks/cleanup.py`

```python
"""
자동 삭제 Celery Task
"""
from sqlalchemy.orm import Session
from datetime import datetime
from zoneinfo import ZoneInfo
import logging

from app.core.celery_app import celery_app
from app.core.database import SessionLocal
from app.models.job import Job
from app.config import settings

logger = logging.getLogger(__name__)


def get_kst_now():
    """KST 현재 시간 반환"""
    return datetime.now(ZoneInfo(settings.TIMEZONE))


@celery_app.task(name="delete_expired_jobs")
def delete_expired_jobs_task():
    """
    만료된 Job 삭제 Celery Task

    처리:
    1. expires_at < 현재 시간인 Job 조회
    2. CASCADE 삭제 (모든 관련 데이터 자동 삭제)
    3. S3 파일 삭제 (선택사항)

    실행 주기: 매시간 (Celery Beat)
    """
    db = SessionLocal()

    try:
        logger.info("[CleanupTask] Starting cleanup")

        # Step 1: 만료된 Job 조회
        now = get_kst_now()
        expired_jobs = db.query(Job).filter(
            Job.expires_at < now
        ).all()

        logger.info(f"[CleanupTask] Found {len(expired_jobs)} expired jobs")

        # Step 2: 각 Job 삭제
        for job in expired_jobs:
            logger.info(f"[CleanupTask] Deleting job: {job.id}")

            # S3 파일 삭제 (선택사항)
            # _delete_s3_files(job)

            # DB 삭제 (CASCADE로 모든 관련 데이터 삭제)
            db.delete(job)

        db.commit()
        logger.info(f"[CleanupTask] Deleted {len(expired_jobs)} jobs")

    except Exception as e:
        logger.error(f"[CleanupTask] Failed: {e}", exc_info=True)
        db.rollback()
    finally:
        db.close()


def _delete_s3_files(job: Job):
    """
    Job의 모든 S3 파일 삭제
    """
    from app.services.s3_service import S3Service

    s3_service = S3Service()

    for file in job.files:
        try:
            s3_service.delete_file(file.s3_key)
            logger.info(f"[CleanupTask] Deleted S3: {file.s3_key}")
        except Exception as e:
            logger.error(f"[CleanupTask] S3 delete failed: {e}")
```

---

## 🔧 Celery Beat 설정

### 파일: `app/core/celery_app.py`

```python
from celery import Celery
from celery.schedules import crontab
from app.config import settings

celery_app = Celery(
    "fmr",
    broker=settings.CELERY_BROKER_URL,
    backend=settings.CELERY_RESULT_BACKEND
)

# Celery Beat 스케줄 설정
celery_app.conf.beat_schedule = {
    'delete-expired-jobs': {
        'task': 'delete_expired_jobs',
        'schedule': crontab(minute=0, hour='*'),  # 매시간 정각 실행
    },
}

celery_app.conf.timezone = settings.TIMEZONE  # KST
```

### Celery Beat 실행

```bash
# Celery Worker 실행
celery -A app.core.celery_app worker --loglevel=info

# Celery Beat 실행 (별도 프로세스)
celery -A app.core.celery_app beat --loglevel=info
```

---

## 🔍 CASCADE 삭제

### DB 스키마 설정

```python
# Job 모델
files = relationship(
    "JobFile",
    back_populates="job",
    cascade="all, delete-orphan"  # Job 삭제 시 자동 삭제
)

payment = relationship(
    "Payment",
    back_populates="job",
    cascade="all, delete-orphan"
)

# ... 모든 관계에 cascade 설정
```

### 삭제되는 데이터

```
Job 삭제 시:
├── Payment (1개)
├── JobFile (4개: 원본 1 + AI 생성 3)
├── SmsLog (N개)
├── PhoneVerificationAttempt (N개)
├── PaymentFailure (N개)
└── Report (1개)
    └── ReportKnowledge (N개)
```

---

## ✅ 테스트

```python
def test_delete_expired_jobs():
    from datetime import datetime
    from zoneinfo import ZoneInfo
    from app.config import settings

    def get_kst_now():
        return datetime.now(ZoneInfo(settings.TIMEZONE))

    # 만료된 Job 생성
    job = Job(
        id=str(uuid.uuid4()),
        expires_at=get_kst_now() - timedelta(hours=1)
    )
    db.add(job)
    db.commit()

    # 삭제 실행
    delete_expired_jobs_task()

    # 확인
    job = db.query(Job).filter(Job.id == job.id).first()
    assert job is None
```

---

## 📝 체크리스트

- [ ] app/tasks/cleanup.py 생성
- [ ] @celery_app.task 데코레이터 적용
- [ ] delete_expired_jobs_task() 구현
- [ ] Celery Beat 스케줄 설정
- [ ] KST 시간대 사용
- [ ] CASCADE 관계 확인
- [ ] S3 파일 삭제 (선택)
- [ ] 테스트 작성

---

## 🚀 다음 단계

자동 삭제 완료 → **error-handling.md**
