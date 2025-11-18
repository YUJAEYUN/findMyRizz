# Image Upload API

## 📋 목표

이미지 업로드 API를 구현합니다.

---

## 🔧 구현

### 파일: `app/api/v1/upload.py`

```python
"""
이미지 업로드 API
"""
from fastapi import APIRouter, Depends, HTTPException, UploadFile, File, Form
from sqlalchemy.orm import Session
import logging
import uuid

from app.core.database import get_db
from app.models.job import Job, JobStatus
from app.models.job_file import JobFile, FileType
from app.utils.image import ImageValidator
from app.services.s3_service import S3Service
from app.tasks.ai_generation import generate_ai_images_task
from app.core.exceptions import AppException

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/uploads", tags=["uploads"])


@router.post("/{job_id}")
async def upload_image(
    job_id: str,
    file: UploadFile = File(...),
    crop_data: str = Form(...),
    db: Session = Depends(get_db)
):
    """
    이미지 업로드

    처리:
    1. Job 조회 및 상태 확인
    2. 이미지 검증
    3. S3 업로드
    4. JobFile 생성
    5. Job 상태 업데이트 (processing)
    6. Celery Task: AI 생성 시작

    Args:
        job_id: Job ID
        file: 이미지 파일
        crop_data: 크롭 데이터 (JSON string)

    Returns:
        {
            "success": true,
            "job_id": str,
            "file_id": str,
            "message": "Image uploaded successfully"
        }
    """
    try:
        logger.info(f"[UploadAPI] Upload: job_id={job_id}")

        # Step 1: Job 조회
        job = db.query(Job).filter(Job.id == job_id).first()
        if not job:
            raise HTTPException(status_code=404, detail="Job not found")

        # Step 2: Job 상태 확인
        if job.status != JobStatus.PENDING_UPLOAD:
            raise HTTPException(
                status_code=400,
                detail=f"Invalid job status: {job.status.value}"
            )

        if job.is_expired():
            raise HTTPException(status_code=400, detail="Job expired")

        # Step 3: 파일 읽기
        file_data = await file.read()

        # Step 4: 이미지 검증
        is_valid, error_msg, metadata = ImageValidator.validate(
            file_data=file_data,
            content_type=file.content_type,
            filename=file.filename
        )

        if not is_valid:
            raise HTTPException(status_code=400, detail=error_msg)

        logger.info(f"[UploadAPI] Validation passed: {metadata}")

        # Step 5: S3 업로드
        s3_service = S3Service()
        file_id = str(uuid.uuid4())

        s3_key = s3_service.generate_s3_key(
            job_id=job_id,
            file_id=file_id,
            file_type="original",
            extension="jpg"
        )

        s3_service.upload_file(
            file_data=file_data,
            s3_key=s3_key,
            content_type="image/jpeg",
            metadata=metadata
        )

        logger.info(f"[UploadAPI] S3 upload success: {s3_key}")

        # Step 6: JobFile 생성
        job_file = JobFile(
            id=file_id,
            job_id=job_id,
            file_type=FileType.ORIGINAL,
            s3_key=s3_key,
            display_order=0,
            crop_data=crop_data
        )
        db.add(job_file)

        # Step 7: Job 상태 업데이트
        job.status = JobStatus.PROCESSING
        db.commit()

        logger.info(f"[UploadAPI] Job status updated: processing")

        # Step 8: Celery Task 실행
        generate_ai_images_task.delay(
            job_id=job_id,
            original_s3_key=s3_key
        )

        logger.info(f"[UploadAPI] AI generation task queued")

        return {
            "success": True,
            "job_id": job_id,
            "file_id": file_id,
            "message": "Image uploaded successfully"
        }

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"[UploadAPI] Failed: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="Internal server error")
```

---

## 🔍 API 사용 예시

### 이미지 업로드

```bash
curl -X POST http://localhost:8000/api/v1/uploads/{job_id} \
  -F "file=@photo.jpg" \
  -F 'crop_data={"x": 0, "y": 0, "width": 300, "height": 400}'
```

**응답:**

```json
{
  "success": true,
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "file_id": "660e8400-e29b-41d4-a716-446655440001",
  "message": "Image uploaded successfully"
}
```

---

## ✅ 테스트 시나리오

### 1. 업로드 성공

```python
def test_upload_success(client):
    # 이미지 파일 준비
    with open("test.jpg", "rb") as f:
        response = client.post(
            f"/api/v1/uploads/{job_id}",
            files={"file": ("test.jpg", f, "image/jpeg")},
            data={"crop_data": '{"x": 0, "y": 0}'}
        )

    assert response.status_code == 200
    data = response.json()
    assert data["success"] == True
```

### 2. Job 없음

```python
def test_upload_job_not_found(client):
    response = client.post(
        "/api/v1/uploads/invalid-job-id",
        files={"file": ("test.jpg", b"data", "image/jpeg")},
        data={"crop_data": "{}"}
    )

    assert response.status_code == 404
```

### 3. 잘못된 파일 형식

```python
def test_upload_invalid_format(client):
    response = client.post(
        f"/api/v1/uploads/{job_id}",
        files={"file": ("test.gif", b"data", "image/gif")},
        data={"crop_data": "{}"}
    )

    assert response.status_code == 400
```

---

## 📝 체크리스트

- [ ] app/api/v1/upload.py 생성
- [ ] upload_image 엔드포인트 구현
- [ ] Job 상태 확인 로직
- [ ] 이미지 검증 연동
- [ ] S3 업로드 연동
- [ ] Background Task 추가
- [ ] 테스트 작성

---

## 🚀 다음 단계

Image Upload API 완료 → **Phase 5: AI Generation Tasks**
