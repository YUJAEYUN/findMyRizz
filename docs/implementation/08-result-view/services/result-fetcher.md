# Result Fetcher 서비스

## 📋 목표

인증된 사용자의 결과를 조회하는 서비스를 구현합니다.

---

## 🔧 구현

### 파일: `app/services/result_fetcher.py`

```python
"""
Result Fetcher 서비스
"""
from sqlalchemy.orm import Session
from typing import Dict
import logging
import json

from app.models.job import Job, JobStatus
from app.models.job_file import JobFile
from app.models.report import Report
from app.services.s3_service import S3Service
from app.core.exceptions import AppException

logger = logging.getLogger(__name__)


class ResultFetcher:
    """
    결과 조회 서비스
    """

    def __init__(self, db: Session):
        self.db = db
        self.s3_service = S3Service()

    def fetch_result(self, job_id: str) -> Dict:
        """
        Job 결과 조회

        Args:
            job_id: Job ID

        Returns:
            결과 데이터
            {
                "job_id": str,
                "status": str,
                "files": List[dict],
                "report": dict | None,
                "expires_at": datetime
            }

        Raises:
            AppException(E-AUTH-001): Job 없음
            AppException(E-AUTH-005): Job 만료됨
        """
        logger.info(f"[ResultFetcher] Fetching result for job: {job_id}")

        # Step 1: Job 조회
        job = self.db.query(Job).filter(Job.id == job_id).first()
        if not job:
            raise AppException(
                status_code=404,
                error_code="E-AUTH-001",
                message="Job not found"
            )

        # Step 2: 만료 확인
        if job.is_expired():
            raise AppException(
                status_code=410,
                error_code="E-AUTH-005",
                message="Job has expired"
            )

        # Step 3: 파일 조회 및 Presigned URL 생성
        files = self._get_files_with_urls(job)

        # Step 4: 리포트 조회
        report = self._get_report(job)

        # Step 5: 응답 생성
        result = {
            "job_id": job.id,
            "status": job.status.value,
            "files": files,
            "report": report,
            "expires_at": job.expires_at
        }

        logger.info(f"[ResultFetcher] Result fetched successfully")

        return result

    def _get_files_with_urls(self, job: Job) -> list:
        """
        파일 목록 조회 및 Presigned URL 생성
        """
        files = []

        for job_file in job.files:
            # Presigned URL 생성 (1시간 유효)
            presigned_url = self.s3_service.generate_presigned_url(
                job_file.s3_key,
                expiration=3600
            )

            files.append({
                "id": job_file.id,
                "file_type": job_file.file_type.value,
                "s3_key": job_file.s3_key,
                "presigned_url": presigned_url,
                "prediction_id": job_file.prediction_id,
                "seed": job_file.seed
            })

        # 정렬: original 먼저, 그 다음 generated (seed 순)
        files.sort(key=lambda x: (
            0 if x["file_type"] == "original" else 1,
            x["seed"] or 0
        ))

        return files

    def _get_report(self, job: Job) -> dict | None:
        """
        리포트 조회
        """
        if job.status != JobStatus.COMPLETED:
            return None

        report = self.db.query(Report).filter(
            Report.job_id == job.id
        ).first()

        if not report:
            return None

        # analysis_result 파싱
        analysis_result = json.loads(report.analysis_result)

        # knowledge_items 변환
        knowledge_items = [
            {
                "id": rk.knowledge_item.id,
                "title": rk.knowledge_item.title,
                "description": rk.knowledge_item.description,
                "procedure_type": rk.knowledge_item.procedure_type,
                "estimated_cost": rk.knowledge_item.estimated_cost
            }
            for rk in report.knowledge_items
        ]

        return {
            "id": report.id,
            "analysis_result": analysis_result,
            "knowledge_items": knowledge_items,
            "created_at": report.created_at.isoformat()
        }
```

---

## 🔍 사용 예시

### API에서 사용

```python
from app.services.result_fetcher import ResultFetcher

@router.get("/results/{job_id}")
async def get_result(
    job_id: str,
    token: str = Depends(verify_token),
    db: Session = Depends(get_db)
):
    # 토큰에서 job_id 추출 및 검증
    token_job_id = extract_job_id_from_token(token)
    if token_job_id != job_id:
        raise HTTPException(status_code=403, detail="Forbidden")

    # 결과 조회
    fetcher = ResultFetcher(db)
    result = fetcher.fetch_result(job_id)

    return result
```

---

## 🔍 응답 예시

```json
{
  "job_id": "job-uuid",
  "status": "completed",
  "files": [
    {
      "id": "file-uuid-1",
      "file_type": "original",
      "s3_key": "jobs/job-uuid/original/file-uuid-1.jpg",
      "presigned_url": "https://s3.amazonaws.com/...",
      "prediction_id": null,
      "seed": null
    },
    {
      "id": "file-uuid-2",
      "file_type": "generated",
      "s3_key": "jobs/job-uuid/generated/file-uuid-2.jpg",
      "presigned_url": "https://s3.amazonaws.com/...",
      "prediction_id": "pred-123",
      "seed": 42
    },
    {
      "id": "file-uuid-3",
      "file_type": "generated",
      "s3_key": "jobs/job-uuid/generated/file-uuid-3.jpg",
      "presigned_url": "https://s3.amazonaws.com/...",
      "prediction_id": "pred-124",
      "seed": 123
    },
    {
      "id": "file-uuid-4",
      "file_type": "generated",
      "s3_key": "jobs/job-uuid/generated/file-uuid-4.jpg",
      "presigned_url": "https://s3.amazonaws.com/...",
      "prediction_id": "pred-125",
      "seed": 456
    }
  ],
  "report": {
    "id": "report-uuid",
    "analysis_result": {
      "skin_tone": "uneven",
      "facial_features": ["dark_circles"],
      "improvement_areas": ["skin", "eyes"],
      "recommendations": ["vitamin_c", "eye_cream"]
    },
    "knowledge_items": [
      {
        "id": "item-uuid-1",
        "title": "비타민 C 세럼",
        "description": "피부 톤을 밝게...",
        "procedure_type": "cosmetic",
        "estimated_cost": "3만원~10만원"
      }
    ],
    "created_at": "2024-01-15T14:30:22.123456"
  },
  "expires_at": "2024-01-16T14:30:22.123456"
}
```

---

## ✅ 테스트

```python
def test_fetch_result_success(db):
    # Job 생성
    from datetime import datetime
    from zoneinfo import ZoneInfo
    from app.config import settings

    def get_kst_now():
        return datetime.now(ZoneInfo(settings.TIMEZONE))

    job = Job(
        id=str(uuid.uuid4()),
        status=JobStatus.COMPLETED,
        expires_at=get_kst_now() + timedelta(hours=24)
    )
    db.add(job)

    # JobFile 생성
    job_file = JobFile(
        id=str(uuid.uuid4()),
        job_id=job.id,
        file_type=FileType.ORIGINAL,
        s3_key="test.jpg"
    )
    db.add(job_file)
    db.commit()

    # 결과 조회
    fetcher = ResultFetcher(db)
    result = fetcher.fetch_result(job.id)

    # 검증
    assert result["job_id"] == job.id
    assert len(result["files"]) == 1
    assert result["files"][0]["presigned_url"] is not None


def test_fetch_result_expired(db):
    from datetime import datetime
    from zoneinfo import ZoneInfo
    from app.config import settings

    def get_kst_now():
        return datetime.now(ZoneInfo(settings.TIMEZONE))

    # 만료된 Job
    job = Job(
        id=str(uuid.uuid4()),
        expires_at=get_kst_now() - timedelta(hours=1)
    )
    db.add(job)
    db.commit()

    # 결과 조회
    fetcher = ResultFetcher(db)

    with pytest.raises(AppException, match="E-AUTH-005"):
        fetcher.fetch_result(job.id)
```

---

## 📝 체크리스트

- [ ] app/services/result_fetcher.py 생성
- [ ] ResultFetcher 클래스 구현
- [ ] fetch_result() 메서드
- [ ] \_get_files_with_urls() 메서드
- [ ] \_get_report() 메서드
- [ ] 테스트 작성

---

## 🚀 다음 단계

Result Fetcher 완료 → **result API**
