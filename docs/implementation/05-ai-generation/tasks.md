# AI 생성 Celery Task

## 📋 목표

Celery Task로 AI 이미지 3장을 생성합니다.

---

## 🔧 구현

### 파일: `app/tasks/ai_generation.py`

```python
"""
AI 이미지 생성 Celery Task
"""
from sqlalchemy.orm import Session
import logging
from datetime import datetime
from zoneinfo import ZoneInfo

from app.core.celery_app import celery_app
from app.core.database import SessionLocal
from app.models.job import Job, JobStatus
from app.services.replicate_service import ReplicateService
from app.services.s3_service import S3Service
from app.config import settings

logger = logging.getLogger(__name__)


def get_kst_now():
    """KST 현재 시간 반환"""
    return datetime.now(ZoneInfo(settings.TIMEZONE))


@celery_app.task(name="generate_ai_images")
def generate_ai_images_task(job_id: str, original_s3_key: str):
    """
    AI 이미지 3장 생성 Celery Task

    처리:
    1. 원본 이미지 Presigned URL 생성
    2. Replicate API 호출 (3회)
    3. prediction_id 저장
    4. Webhook 대기

    Args:
        job_id: Job ID
        original_s3_key: 원본 이미지 S3 키
    """
    db = SessionLocal()

    try:
        logger.info(f"[AIGenTask] Starting: job_id={job_id}")

        # Step 1: Job 조회
        job = db.query(Job).filter(Job.id == job_id).first()
        if not job:
            logger.error(f"[AIGenTask] Job not found: {job_id}")
            return

        # Step 2: Presigned URL 생성
        s3_service = S3Service()
        image_url = s3_service.generate_presigned_url(
            s3_key=original_s3_key,
            expiration=3600  # 1시간
        )

        logger.info(f"[AIGenTask] Presigned URL generated")

        # Step 3: Webhook URL 생성
        webhook_url = f"{settings.API_BASE_URL}/api/v1/webhooks/replicate"

        # Step 4: Replicate API 호출 (3장)
        replicate_service = ReplicateService()
        results = replicate_service.generate_multiple(
            image_url=image_url,
            count=3,
            webhook_url=webhook_url
        )

        logger.info(f"[AIGenTask] Generated 3 predictions: {results}")

        # Step 5: prediction_id 저장 (선택사항)
        # JobFile에 prediction_id를 미리 저장할 수도 있음

        logger.info(f"[AIGenTask] Completed: job_id={job_id}")

    except Exception as e:
        logger.error(f"[AIGenTask] Failed: {e}", exc_info=True)

        # Job 상태를 FAILED로 변경
        try:
            job = db.query(Job).filter(Job.id == job_id).first()
            if job:
                job.status = JobStatus.FAILED
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
from app.tasks.ai_generation import generate_ai_images_task

@router.post("/uploads/{job_id}")
async def upload_image(
    job_id: str,
    ...
):
    # 업로드 처리...

    # Celery Task 실행
    generate_ai_images_task.delay(
        job_id=job_id,
        original_s3_key=s3_key
    )

    return {"success": True}
```

---

## 🔍 처리 흐름

```
1. upload_image API 호출
   ↓
2. Celery Task 큐에 추가
   ↓
3. generate_ai_images_task 실행
   ↓
4. Replicate API 호출 (3회)
   - prediction_id_1
   - prediction_id_2
   - prediction_id_3
   ↓
5. Replicate가 생성 완료 시 Webhook 호출
   ↓
6. Webhook Handler가 이미지 다운로드 및 저장
```

---

## ✅ 테스트 시나리오

### 1. Task 실행 성공

```python
def test_generate_ai_images_success(mocker):
    mock_replicate = mocker.Mock()
    mock_replicate.generate_multiple.return_value = [
        {"prediction_id": "pred_1", "seed": 123},
        {"prediction_id": "pred_2", "seed": 456},
        {"prediction_id": "pred_3", "seed": 789}
    ]

    mocker.patch(
        'app.services.replicate_service.ReplicateService',
        return_value=mock_replicate
    )

    generate_ai_images(job_id="test-job", original_s3_key="test.jpg")

    assert mock_replicate.generate_multiple.called
```

---

## 📝 체크리스트

- [ ] app/tasks/ai_generation.py 생성
- [ ] @celery_app.task 데코레이터 적용
- [ ] generate_ai_images_task() 함수 구현
- [ ] Presigned URL 생성
- [ ] Replicate API 호출
- [ ] KST 시간대 사용
- [ ] 에러 처리
- [ ] 테스트 작성

---

## 🚀 다음 단계

AI Generation Task 완료 → **Webhook Handler**
