# 리포트 생성 Celery Task

## 📋 목표

Celery Task로 LangChain Workflow를 실행하여 리포트를 생성합니다.

---

## 🔧 구현

### 파일: `app/tasks/report_generation.py`

```python
"""
리포트 생성 Celery Task
"""
import logging
from datetime import datetime
from zoneinfo import ZoneInfo

from app.core.celery_app import celery_app
from app.core.database import SessionLocal
from app.models.job import Job, JobStatus
from app.models.job_file import JobFile, FileType
from app.services.langchain_workflow import run_report_workflow
from app.services.s3_service import S3Service
from app.config import settings

logger = logging.getLogger(__name__)


def get_kst_now():
    """KST 현재 시간 반환"""
    return datetime.now(ZoneInfo(settings.TIMEZONE))


@celery_app.task(name="generate_report")
def generate_report_task(job_id: str):
    """
    리포트 생성 Celery Task

    처리:
    1. Job 조회 및 상태 확인
    2. 생성된 이미지 URL 수집
    3. LangChain Workflow 실행
    4. Job 상태 업데이트 (completed)
    5. SMS 발송 트리거

    Args:
        job_id: Job ID
    """
    db = SessionLocal()

    try:
        logger.info(f"[ReportGenTask] Starting: job_id={job_id}")

        # Step 1: Job 조회
        job = db.query(Job).filter(Job.id == job_id).first()
        if not job:
            logger.error(f"[ReportGenTask] Job not found: {job_id}")
            return

        # Step 2: 생성된 이미지 파일 조회
        generated_files = db.query(JobFile).filter(
            JobFile.job_id == job_id,
            JobFile.file_type == FileType.GENERATED
        ).all()

        if len(generated_files) != 3:
            logger.error(f"[ReportGenTask] Expected 3 images, got {len(generated_files)}")
            job.status = JobStatus.FAILED
            job.updated_at = get_kst_now()
            db.commit()
            return

        logger.info(f"[ReportGenTask] Found {len(generated_files)} generated images")

        # Step 3: S3 Presigned URL 생성
        s3_service = S3Service()
        image_urls = []

        for file in generated_files:
            presigned_url = s3_service.generate_presigned_url(
                s3_key=file.s3_key,
                expiration=3600  # 1시간
            )
            image_urls.append(presigned_url)

        logger.info(f"[ReportGenTask] Generated {len(image_urls)} presigned URLs")

        # Step 4: LangChain Workflow 실행
        import asyncio
        result = asyncio.run(run_report_workflow(
            job_id=job_id,
            image_urls=image_urls
        ))

        logger.info(f"[ReportGenTask] Workflow completed: report_id={result['report_id']}")

        # Step 5: Job 상태 업데이트 (completed)
        job.status = JobStatus.COMPLETED
        job.updated_at = get_kst_now()
        db.commit()

        logger.info(f"[ReportGenTask] Job completed: {job_id}")

        # Step 6: SMS 발송 트리거
        from app.tasks.sms_notification import send_sms_notification_task
        send_sms_notification_task.delay(job_id=job_id)

        logger.info(f"[ReportGenTask] SMS notification task queued")

    except Exception as e:
        logger.error(f"[ReportGenTask] Failed: {e}", exc_info=True)

        # Job 상태를 FAILED로 변경
        try:
            job = db.query(Job).filter(Job.id == job_id).first()
            if job:
                job.status = JobStatus.FAILED
                job.updated_at = get_kst_now()
                db.commit()
        except:
            pass

    finally:
        db.close()
```

---

## 🔍 사용 예시

### Celery Task 실행

```python
from app.tasks.report_generation import generate_report_task

# Webhook에서 3장 완료 시 호출
if generated_count + 1 == 3:
    generate_report_task.delay(job_id=job_id)
```

---

## 🔍 처리 흐름

```
1. Replicate Webhook (3장 완료)
   ↓
2. Celery Task 큐에 추가
   ↓
3. generate_report_task 실행
   ↓
4. 생성된 이미지 URL 수집
   ↓
5. LangChain Workflow 실행
   - Node 1: GPT-4o Vision 분석
   - Node 2: Knowledge 매칭
   - Node 3: Report DB 저장
   ↓
6. Job 상태 업데이트 (completed)
   ↓
7. SMS 발송 Celery Task 트리거
```

---

## ✅ 테스트 시나리오

### 1. Task 실행 성공

```python
def test_generate_report_task_success(mocker):
    # Mock Workflow
    mock_workflow_result = {
        'report_id': 'report-123',
        'analysis_result': {...},
        'matched_knowledge': [...]
    }

    mocker.patch(
        'app.services.langchain_workflow.run_report_workflow',
        return_value=mock_workflow_result
    )

    # Task 실행
    generate_report_task(job_id="test-job")

    # 검증
    job = db.query(Job).filter(Job.id == "test-job").first()
    assert job.status == JobStatus.COMPLETED
```

---

## 📝 체크리스트

- [ ] app/tasks/report_generation.py 생성
- [ ] @celery_app.task 데코레이터 적용
- [ ] generate_report_task() 함수 구현
- [ ] 생성된 이미지 URL 수집
- [ ] LangChain Workflow 실행
- [ ] KST 시간대 사용
- [ ] Job 상태 업데이트 (completed)
- [ ] SMS 발송 트리거
- [ ] 에러 처리
- [ ] 테스트 작성

---

## 🚀 다음 단계

Report Generation Task 완료 → **SMS Notification**
