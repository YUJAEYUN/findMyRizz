# 전화번호 등록 API

## 📋 목표

결제 성공 후 사용자 전화번호를 등록하는 API를 구현합니다.

---

## 🎯 API 스펙

### 엔드포인트

```
PATCH /api/v1/jobs/{job_id}/phone
Content-Type: application/json
```

### 요청 바디

```json
{
  "user_phone_number": "010-1234-5678"
}
```

### 성공 응답 (200)

```json
{
  "success": true,
  "data": {
    "job_id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "pending_upload",
    "user_phone_number": "01012345678"
  }
}
```

### 에러 응답

| HTTP 코드 | 에러 코드        | 메시지                           | 발생 조건                       |
| --------- | ---------------- | -------------------------------- | ------------------------------- |
| 404       | E-JOB-001        | Job not found                    | Job이 존재하지 않음             |
| 400       | E-JOB-002        | Invalid job status               | Job 상태가 pending_phone이 아님 |
| 422       | validation_error | Phone number must start with 010 | 010으로 시작하지 않음           |
| 422       | validation_error | Phone number must be 11 digits   | 11자리가 아님                   |

---

## 🔧 구현

### 파일: `app/schemas/job.py`

```python
"""
Job 관련 Pydantic 스키마
"""
from pydantic import BaseModel, Field, validator
import re


class JobPhoneUpdateRequest(BaseModel):
    """
    전화번호 등록 요청 스키마
    """
    user_phone_number: str = Field(
        ...,
        description="사용자 전화번호",
        example="010-1234-5678"
    )

    @validator('user_phone_number')
    def validate_phone_number(cls, v: str) -> str:
        """
        전화번호 검증 및 정제

        검증 규칙:
        1. 하이픈, 공백 제거
        2. 숫자만 포함하는지 확인
        3. 010으로 시작하는지 확인
        4. 정확히 11자리인지 확인
        """
        # 하이픈, 공백 제거
        phone = v.replace('-', '').replace(' ', '')

        # 숫자만 포함하는지 확인
        if not phone.isdigit():
            raise ValueError("Phone number must contain only digits")

        # 010으로 시작하는지 확인
        if not phone.startswith('010'):
            raise ValueError("Phone number must start with 010")

        # 정확히 11자리인지 확인
        if len(phone) != 11:
            raise ValueError("Phone number must be 11 digits")

        return phone


class JobPhoneUpdateResponse(BaseModel):
    """전화번호 등록 응답 데이터"""
    job_id: str
    status: str
    user_phone_number: str
```

---

## 🔧 서비스 로직

### 파일: `app/services/job_service.py`

```python
"""
Job 비즈니스 로직
"""
from sqlalchemy.orm import Session
from datetime import datetime
from zoneinfo import ZoneInfo
import logging

from app.models.job import Job, JobStatus
from app.config import settings
from app.core.exceptions import AppException

logger = logging.getLogger(__name__)


def get_kst_now():
    """KST 현재 시간 반환"""
    return datetime.now(ZoneInfo(settings.TIMEZONE))


class JobService:
    """Job 서비스 클래스"""

    def __init__(self, db: Session):
        self.db = db

    def update_phone_number(self, job_id: str, phone_number: str) -> dict:
        """
        전화번호 등록

        처리 흐름:
        1. Job 조회
        2. 상태 확인 (pending_phone)
        3. 전화번호 업데이트
        4. 상태 변경 (pending_upload)

        Args:
            job_id: Job ID
            phone_number: 전화번호 (11자리)

        Returns:
            {
                "job_id": str,
                "status": str,
                "user_phone_number": str
            }
        """
        try:
            # Step 1: Job 조회
            job = self.db.query(Job).filter(Job.id == job_id).first()

            if not job:
                logger.error(f"[JobService] Job not found: {job_id}")
                raise AppException(
                    status_code=404,
                    error_code="E-JOB-001",
                    message="Job not found"
                )

            # Step 2: 상태 확인
            if job.status != JobStatus.PENDING_PHONE:
                logger.error(
                    f"[JobService] Invalid status: {job.status}, expected: pending_phone"
                )
                raise AppException(
                    status_code=400,
                    error_code="E-JOB-002",
                    message="Invalid job status. Payment must be completed first."
                )

            # Step 3: 전화번호 업데이트
            job.user_phone_number = phone_number
            job.status = JobStatus.PENDING_UPLOAD
            job.updated_at = get_kst_now()

            self.db.commit()
            logger.info(f"[JobService] Phone updated: job_id={job_id}")

            return {
                "job_id": job.id,
                "status": job.status.value,
                "user_phone_number": job.user_phone_number
            }

        except AppException:
            self.db.rollback()
            raise
        except Exception as e:
            self.db.rollback()
            logger.error(f"[JobService] Failed to update phone: {e}", exc_info=True)
            raise AppException(
                status_code=500,
                error_code="E-JOB-003",
                message="Failed to update phone number"
            )
```

---

## 🔧 API 엔드포인트

### 파일: `app/api/v1/jobs.py`

```python
"""
Job API 엔드포인트
"""
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
import logging

from app.core.database import get_db
from app.schemas.job import JobPhoneUpdateRequest, JobPhoneUpdateResponse
from app.services.job_service import JobService
from app.core.exceptions import AppException

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/jobs", tags=["jobs"])


@router.patch("/{job_id}/phone", response_model=JobPhoneUpdateResponse)
async def update_job_phone(
    job_id: str,
    request: JobPhoneUpdateRequest,
    db: Session = Depends(get_db)
):
    """
    전화번호 등록

    결제 성공 후 호출되는 API

    Args:
        job_id: Job ID (UUID)
        request: {
            "user_phone_number": "010-1234-5678"
        }

    Returns:
        {
            "job_id": str,
            "status": str,
            "user_phone_number": str
        }
    """
    try:
        logger.info(f"[JobAPI] Update phone: job_id={job_id}")

        service = JobService(db)
        result = service.update_phone_number(job_id, request.user_phone_number)

        return JobPhoneUpdateResponse(**result)

    except AppException as e:
        logger.error(f"[JobAPI] Failed: {e.message}")
        raise HTTPException(status_code=e.status_code, detail=e.message)
    except Exception as e:
        logger.error(f"[JobAPI] Unexpected error: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="Internal server error")
```

---

## 🔧 라우터 등록

### 파일: `app/api/v1/__init__.py`

```python
from fastapi import APIRouter
from app.api.v1 import payment, jobs

api_router = APIRouter()

api_router.include_router(payment.router)
api_router.include_router(jobs.router)  # 추가
```

---

## ✅ 테스트

### 성공 케이스

```python
def test_update_phone_success(client, db):
    # 1. Job 생성 (pending_phone 상태)
    job = Job(
        id=str(uuid.uuid4()),
        status=JobStatus.PENDING_PHONE,
        expires_at=datetime.now() + timedelta(hours=24)
    )
    db.add(job)
    db.commit()

    # 2. 전화번호 등록
    response = client.patch(
        f"/api/v1/jobs/{job.id}/phone",
        json={"user_phone_number": "010-1234-5678"}
    )

    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "pending_upload"
    assert data["user_phone_number"] == "01012345678"
```

### 실패 케이스

```python
def test_update_phone_invalid_status(client, db):
    # Job이 pending_payment 상태
    job = Job(
        id=str(uuid.uuid4()),
        status=JobStatus.PENDING_PAYMENT,
        expires_at=datetime.now() + timedelta(hours=24)
    )
    db.add(job)
    db.commit()

    response = client.patch(
        f"/api/v1/jobs/{job.id}/phone",
        json={"user_phone_number": "010-1234-5678"}
    )

    assert response.status_code == 400
```

---

## 📝 체크리스트

- [ ] app/schemas/job.py 생성
- [ ] app/services/job_service.py 생성
- [ ] app/api/v1/jobs.py 생성
- [ ] 라우터 등록
- [ ] 테스트 작성 및 통과

---

## 🚀 다음 단계

전화번호 등록 완료 → **Phase 4: Image Upload**
