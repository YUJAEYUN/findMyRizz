# Replicate Webhook Handler

## 📋 목표

Replicate Webhook을 수신하고 생성된 이미지를 저장합니다.

---

## 🔧 구현

### 파일: `app/api/v1/webhook.py`

```python
"""
Webhook API
"""
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from pydantic import BaseModel
import logging
import requests
import uuid
from datetime import datetime
from zoneinfo import ZoneInfo

from app.core.database import get_db
from app.models.job import Job, JobStatus
from app.models.job_file import JobFile, FileType
from app.services.s3_service import S3Service
from app.tasks.report_generation import generate_report_task
from app.config import settings

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/webhooks", tags=["webhooks"])


def get_kst_now():
    """KST 현재 시간 반환"""
    return datetime.now(ZoneInfo(settings.TIMEZONE))


class ReplicateWebhook(BaseModel):
    """Replicate Webhook 데이터"""
    id: str
    status: str
    output: str = None
    input: dict


@router.post("/replicate")
async def replicate_webhook(
    webhook: ReplicateWebhook,
    db: Session = Depends(get_db)
):
    """
    Replicate Webhook 수신

    처리:
    1. prediction_id로 Job 조회
    2. 생성된 이미지 다운로드
    3. S3 업로드
    4. JobFile 생성
    5. 3장 완료 시 리포트 생성 시작

    Args:
        webhook: {
            "id": "pred_abc123",
            "status": "succeeded",
            "output": "https://replicate.delivery/...jpg",
            "input": {...}
        }
    """
    try:
        logger.info(f"[ReplicateWebhook] Received: {webhook.id}")

        # Step 1: 상태 확인
        if webhook.status != "succeeded":
            logger.error(f"[ReplicateWebhook] Failed: {webhook.status}")
            return {"success": False, "message": "Generation failed"}

        if not webhook.output:
            logger.error(f"[ReplicateWebhook] No output")
            return {"success": False, "message": "No output"}

        # Step 2: 원본 이미지 URL에서 job_id 추출
        # input.image = "https://s3.../jobs/{job_id}/original/..."
        input_image_url = webhook.input.get("image", "")
        job_id = extract_job_id_from_url(input_image_url)

        if not job_id:
            logger.error(f"[ReplicateWebhook] Cannot extract job_id")
            return {"success": False, "message": "Invalid input"}

        # Step 3: Job 조회
        job = db.query(Job).filter(Job.id == job_id).first()
        if not job:
            logger.error(f"[ReplicateWebhook] Job not found: {job_id}")
            return {"success": False, "message": "Job not found"}

        # Step 4: 생성된 이미지 다운로드
        image_response = requests.get(webhook.output, timeout=30)
        image_data = image_response.content

        logger.info(f"[ReplicateWebhook] Downloaded: {len(image_data)} bytes")

        # Step 5: S3 업로드
        s3_service = S3Service()
        file_id = str(uuid.uuid4())

        s3_key = s3_service.generate_s3_key(
            job_id=job_id,
            file_id=file_id,
            file_type="generated",
            extension="jpg"
        )

        s3_service.upload_file(
            file_data=image_data,
            s3_key=s3_key,
            content_type="image/jpeg"
        )

        logger.info(f"[ReplicateWebhook] S3 upload: {s3_key}")

        # Step 6: JobFile 생성
        # 현재 생성된 이미지 개수 확인
        generated_count = db.query(JobFile).filter(
            JobFile.job_id == job_id,
            JobFile.file_type == FileType.GENERATED
        ).count()

        job_file = JobFile(
            id=file_id,
            job_id=job_id,
            file_type=FileType.GENERATED,
            s3_key=s3_key,
            display_order=generated_count + 1,
            prediction_id=webhook.id,
            seed=webhook.input.get("seed")
        )
        db.add(job_file)
        db.commit()

        logger.info(f"[ReplicateWebhook] JobFile created: {generated_count + 1}/3")

        # Step 7: 3장 완료 시 리포트 생성
        if generated_count + 1 == 3:
            logger.info(f"[ReplicateWebhook] All 3 images generated")

            # Job 상태 업데이트
            job.status = JobStatus.PROCESSING
            job.updated_at = get_kst_now()
            db.commit()

            # Celery Task: 리포트 생성
            generate_report_task.delay(job_id=job_id)

        return {"success": True, "message": "Image saved"}

    except Exception as e:
        logger.error(f"[ReplicateWebhook] Error: {e}", exc_info=True)
        return {"success": False, "message": str(e)}


def extract_job_id_from_url(url: str) -> str:
    """
    S3 URL에서 job_id 추출

    URL 형식: https://s3.../jobs/{job_id}/original/...

    Args:
        url: S3 URL

    Returns:
        str: job_id or None
    """
    try:
        parts = url.split("/")
        jobs_index = parts.index("jobs")
        job_id = parts[jobs_index + 1]
        return job_id
    except:
        return None
```

---

## 🔍 Webhook 데이터 예시

```json
{
  "id": "pred_abc123xyz",
  "status": "succeeded",
  "output": "https://replicate.delivery/pbxt/...jpg",
  "input": {
    "image": "https://s3.../jobs/job-uuid/original/file-uuid.jpg",
    "seed": 123456
  }
}
```

---

## ✅ 테스트 시나리오

### 1. Webhook 성공

```python
def test_replicate_webhook_success(client):
    response = client.post(
        "/api/v1/webhooks/replicate",
        json={
            "id": "pred_123",
            "status": "succeeded",
            "output": "https://replicate.delivery/test.jpg",
            "input": {
                "image": "https://s3.../jobs/job-id/original/file.jpg",
                "seed": 123
            }
        }
    )

    assert response.status_code == 200
    data = response.json()
    assert data["success"] == True
```

---

## 📝 체크리스트

- [ ] app/api/v1/webhook.py 생성
- [ ] replicate_webhook 엔드포인트 구현
- [ ] 이미지 다운로드 로직
- [ ] S3 업로드 연동
- [ ] JobFile 생성
- [ ] KST 시간대 사용
- [ ] 3장 완료 감지
- [ ] Celery Task로 리포트 생성 트리거
- [ ] 테스트 작성

---

## 🚀 다음 단계

Webhook Handler 완료 → **Phase 6: Report Generation**
