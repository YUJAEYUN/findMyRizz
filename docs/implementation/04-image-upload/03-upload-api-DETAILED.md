# 이미지 업로드 API - 완전한 슈도코드 명세

## 📋 개요

결제 완료 후 사용자가 이미지를 업로드하는 API의 **라인 바이 라인 구현 명세**

---

## 🎯 API 스펙

### 엔드포인트

```
POST /api/v1/uploads/{job_id}
Content-Type: multipart/form-data
```

### 요청

```
file: (binary) - 이미지 파일
crop_data: (string) - JSON 문자열 {"x": 0, "y": 0, "width": 300, "height": 400}
```

### 성공 응답 (200)

```json
{
  "success": true,
  "data": {
    "file_id": "file-uuid-123",
    "s3_key": "jobs/550e8400-e29b-41d4-a716-446655440000/original/file-uuid-123.jpg",
    "message": "Image uploaded successfully. AI generation started."
  }
}
```

### 에러 응답

| HTTP 코드 | 에러 코드 | 메시지                   | 발생 조건                        |
| --------- | --------- | ------------------------ | -------------------------------- |
| 404       | E-IMG-001 | Job not found            | Job ID가 존재하지 않음           |
| 400       | E-IMG-002 | Invalid job status       | Job 상태가 pending_upload가 아님 |
| 400       | E-IMG-003 | File size exceeds 10MB   | 파일 크기 초과                   |
| 400       | E-IMG-004 | Invalid file format      | JPEG/PNG가 아님                  |
| 400       | E-IMG-005 | Invalid image dimensions | 이미지 크기 부적합               |
| 500       | E-IMG-999 | Failed to upload image   | S3 업로드 실패                   |

---

## 📁 파일 구조

```
app/
├── api/v1/uploads.py           # ← API 엔드포인트
├── schemas/upload.py           # ← Pydantic 스키마
├── services/
│   ├── image_service.py        # ← 이미지 처리 로직
│   └── s3_service.py           # ← S3 업로드 로직
├── utils/image.py              # ← 이미지 검증 유틸
└── tasks/ai_generation.py      # ← Background Task
```

---

## 🔧 구현 1: 이미지 검증 유틸

### 파일: `app/utils/image.py`

```python
"""
이미지 검증 유틸리티
"""
from PIL import Image
import io
from typing import Tuple
import logging

logger = logging.getLogger(__name__)


class ImageValidator:
    """
    이미지 검증 클래스

    검증 항목:
    1. 파일 크기 (10KB ~ 10MB)
    2. 파일 형식 (JPEG, PNG)
    3. 이미지 유효성 (손상 여부)
    4. 이미지 크기 (512x512 ~ 4096x4096)
    """

    # 상수 정의
    MAX_SIZE = 10 * 1024 * 1024  # 10MB
    MIN_SIZE = 10 * 1024  # 10KB
    ALLOWED_CONTENT_TYPES = ['image/jpeg', 'image/png']
    ALLOWED_EXTENSIONS = ['.jpg', '.jpeg', '.png']
    MIN_DIMENSION = 512
    MAX_DIMENSION = 4096

    @classmethod
    def validate(cls, file_data: bytes, content_type: str, filename: str) -> Tuple[bool, str, dict]:
        """
        이미지 전체 검증

        Args:
            file_data (bytes): 이미지 파일 바이너리 데이터
            content_type (str): Content-Type 헤더 값
            filename (str): 파일명

        Returns:
            Tuple[bool, str, dict]:
                - is_valid (bool): 검증 성공 여부
                - error_message (str): 에러 메시지 (성공 시 빈 문자열)
                - metadata (dict): 이미지 메타데이터 (width, height, format)
        """
        # ===== Step 1: 파일 크기 검증 =====
        file_size = len(file_data)
        logger.info(f"[ImageValidator] Validating image: size={file_size} bytes, content_type={content_type}")

        if file_size > cls.MAX_SIZE:
            return False, f"File size exceeds {cls.MAX_SIZE // (1024*1024)}MB", {}

        if file_size < cls.MIN_SIZE:
            return False, f"File size is too small (minimum {cls.MIN_SIZE // 1024}KB)", {}

        # ===== Step 2: Content-Type 검증 =====
        if content_type not in cls.ALLOWED_CONTENT_TYPES:
            return False, "Only JPEG and PNG files are allowed", {}

        # ===== Step 3: 파일 확장자 검증 =====
        file_ext = filename.lower().split('.')[-1] if '.' in filename else ''
        if f".{file_ext}" not in cls.ALLOWED_EXTENSIONS:
            return False, "Invalid file extension", {}

        # ===== Step 4: 이미지 유효성 검증 (PIL로 열기 시도) =====
        try:
            image = Image.open(io.BytesIO(file_data))
            width, height = image.size
            image_format = image.format
            logger.info(f"[ImageValidator] Image opened successfully: {width}x{height}, format={image_format}")
        except Exception as e:
            logger.error(f"[ImageValidator] Failed to open image: {e}")
            return False, "Invalid or corrupted image file", {}

        # ===== Step 5: 이미지 크기 검증 =====
        if width < cls.MIN_DIMENSION or height < cls.MIN_DIMENSION:
            return False, f"Image dimensions must be at least {cls.MIN_DIMENSION}x{cls.MIN_DIMENSION}", {}

        if width > cls.MAX_DIMENSION or height > cls.MAX_DIMENSION:
            return False, f"Image dimensions must not exceed {cls.MAX_DIMENSION}x{cls.MAX_DIMENSION}", {}

        # ===== Step 6: 메타데이터 반환 =====
        metadata = {
            "width": width,
            "height": height,
            "format": image_format,
            "size_bytes": file_size
        }

        logger.info(f"[ImageValidator] Image validation passed: {metadata}")
        return True, "", metadata
```

---

## 🔧 구현 2: S3 서비스

### 파일: `app/services/s3_service.py`

```python
"""
AWS S3 파일 업로드 서비스
"""
import boto3
from botocore.exceptions import ClientError
import logging
from typing import Optional

from app.config import settings
from app.core.exceptions import AppException

logger = logging.getLogger(__name__)


class S3Service:
    """
    S3 파일 관리 서비스

    책임:
    - 파일 업로드
    - Presigned URL 생성
    - 파일 삭제
    """

    def __init__(self):
        """S3 클라이언트 초기화"""
        self.s3_client = boto3.client(
            's3',
            aws_access_key_id=settings.AWS_ACCESS_KEY_ID,
            aws_secret_access_key=settings.AWS_SECRET_ACCESS_KEY,
            region_name=settings.AWS_REGION
        )
        self.bucket_name = settings.S3_BUCKET_NAME

    def upload_file(
        self,
        file_data: bytes,
        s3_key: str,
        content_type: str,
        metadata: Optional[dict] = None
    ) -> str:
        """
        S3에 파일 업로드

        Args:
            file_data (bytes): 업로드할 파일 데이터
            s3_key (str): S3 객체 키 (경로)
                예: "jobs/job-id/original/file-id.jpg"
            content_type (str): Content-Type
                예: "image/jpeg"
            metadata (dict, optional): 추가 메타데이터
                예: {"width": "1024", "height": "768"}

        Returns:
            str: 업로드된 파일의 S3 키

        Raises:
            AppException: S3 업로드 실패 시 (E-FILE-001)
        """
        try:
            logger.info(f"[S3Service] Uploading file to S3: key={s3_key}, size={len(file_data)} bytes")

            # S3 업로드 파라미터
            upload_params = {
                'Bucket': self.bucket_name,
                'Key': s3_key,
                'Body': file_data,
                'ContentType': content_type
            }

            # 메타데이터 추가 (선택)
            if metadata:
                # 메타데이터는 문자열만 가능
                str_metadata = {k: str(v) for k, v in metadata.items()}
                upload_params['Metadata'] = str_metadata

            # S3 업로드 실행
            self.s3_client.put_object(**upload_params)

            logger.info(f"[S3Service] File uploaded successfully: {s3_key}")
            return s3_key

        except ClientError as e:
            logger.error(f"[S3Service] S3 upload failed: {e}", exc_info=True)
            raise AppException(
                status_code=500,
                error_code="E-FILE-001",
                message="Failed to upload file to S3"
            )

    def generate_s3_key(self, job_id: str, file_id: str, file_type: str, extension: str) -> str:
        """
        S3 키 생성

        형식: jobs/{job_id}/{file_type}/{file_id}.{extension}
        예시: jobs/550e8400-e29b-41d4-a716-446655440000/original/file-123.jpg

        Args:
            job_id (str): Job ID
            file_id (str): File ID (UUID)
            file_type (str): 파일 타입 ("original" 또는 "generated")
            extension (str): 파일 확장자 ("jpg", "png")

        Returns:
            str: S3 키
        """
        s3_key = f"jobs/{job_id}/{file_type}/{file_id}.{extension}"
        return s3_key
```

---

## 🔧 구현 3: 이미지 서비스

### 파일: `app/services/image_service.py`

```python
"""
이미지 처리 비즈니스 로직
"""
from sqlalchemy.orm import Session
from datetime import datetime
import uuid
import json
import logging

from app.models.job import Job, JobStatus
from app.models.job_file import JobFile, FileType
from app.services.s3_service import S3Service
from app.utils.image import ImageValidator
from app.core.exceptions import AppException

logger = logging.getLogger(__name__)


class ImageService:
    """
    이미지 처리 서비스

    책임:
    - 이미지 검증
    - S3 업로드
    - JobFile 레코드 생성
    - Job 상태 업데이트
    """

    def __init__(self, db: Session):
        """
        Args:
            db: SQLAlchemy 세션
        """
        self.db = db
        self.s3_service = S3Service()

    def upload_image(
        self,
        job_id: str,
        file_data: bytes,
        filename: str,
        content_type: str,
        crop_data_str: str
    ) -> dict:
        """
        이미지 업로드 메인 로직

        처리 흐름:
        1. Job 조회 및 상태 확인
        2. 이미지 검증
        3. crop_data 파싱
        4. S3 업로드
        5. JobFile 레코드 생성
        6. Job 상태 업데이트 (processing)
        7. 응답 반환

        Args:
            job_id (str): Job ID
            file_data (bytes): 이미지 파일 데이터
            filename (str): 파일명
            content_type (str): Content-Type
            crop_data_str (str): 크롭 데이터 JSON 문자열

        Returns:
            dict: {
                "file_id": str,
                "s3_key": str,
                "message": str
            }

        Raises:
            AppException: 각종 검증 실패 또는 업로드 실패 시
        """
        # ===== Step 1: Job 조회 및 상태 확인 =====
        job = self.db.query(Job).filter(Job.id == job_id).first()

        if not job:
            logger.error(f"[ImageService] Job not found: {job_id}")
            raise AppException(
                status_code=404,
                error_code="E-IMG-001",
                message="Job not found"
            )

        if job.status != JobStatus.PENDING_UPLOAD:
            logger.error(f"[ImageService] Invalid job status: {job.status}, expected: pending_upload")
            raise AppException(
                status_code=400,
                error_code="E-IMG-002",
                message=f"Invalid job status: {job.status.value}"
            )

        logger.info(f"[ImageService] Job found: {job_id}, status={job.status.value}")

        # ===== Step 2: 이미지 검증 =====
        is_valid, error_msg, metadata = ImageValidator.validate(
            file_data=file_data,
            content_type=content_type,
            filename=filename
        )

        if not is_valid:
            logger.error(f"[ImageService] Image validation failed: {error_msg}")
            raise AppException(
                status_code=400,
                error_code="E-IMG-005",
                message=error_msg
            )

        # ===== Step 3: crop_data 파싱 =====
        try:
            crop_data = json.loads(crop_data_str)
            # 필수 필드 확인
            required_fields = ['x', 'y', 'width', 'height']
            for field in required_fields:
                if field not in crop_data:
                    raise ValueError(f"Missing field: {field}")
            logger.info(f"[ImageService] Crop data parsed: {crop_data}")
        except Exception as e:
            logger.error(f"[ImageService] Invalid crop_data: {e}")
            raise AppException(
                status_code=400,
                error_code="E-IMG-006",
                message="Invalid crop_data format"
            )

        # ===== Step 4: S3 업로드 =====
        file_id = str(uuid.uuid4())
        file_extension = filename.split('.')[-1].lower()

        s3_key = self.s3_service.generate_s3_key(
            job_id=job_id,
            file_id=file_id,
            file_type="original",
            extension=file_extension
        )

        self.s3_service.upload_file(
            file_data=file_data,
            s3_key=s3_key,
            content_type=content_type,
            metadata=metadata
        )

        logger.info(f"[ImageService] File uploaded to S3: {s3_key}")

        # ===== Step 5: JobFile 레코드 생성 =====
        job_file = JobFile(
            id=file_id,
            job_id=job_id,
            file_type=FileType.ORIGINAL,
            s3_key=s3_key,
            file_size=metadata['size_bytes'],
            mime_type=content_type,
            width=metadata['width'],
            height=metadata['height'],
            crop_data=crop_data,  # JSON 타입
            display_order=0,
            created_at=datetime.utcnow()
        )
        self.db.add(job_file)
        logger.info(f"[ImageService] JobFile created: {file_id}")

        # ===== Step 6: Job 상태 업데이트 =====
        job.status = JobStatus.PROCESSING
        job.updated_at = get_kst_now()

        self.db.commit()
        logger.info(f"[ImageService] Job status updated to: processing")

        # ===== Step 7: 응답 반환 =====
        return {
            "file_id": file_id,
            "s3_key": s3_key,
            "message": "Image uploaded successfully. AI generation started."
        }
```

---

## 🔧 구현 4: API 라우터

### 파일: `app/api/v1/uploads.py`

```python
"""
이미지 업로드 API 라우터
"""
from fastapi import APIRouter, UploadFile, File, Form, Depends, HTTPException
from sqlalchemy.orm import Session
import logging

from app.core.database import get_db
from app.services.image_service import ImageService
from app.tasks.ai_generation import generate_ai_images_task
from app.core.exceptions import AppException

router = APIRouter(prefix="/uploads", tags=["uploads"])
logger = logging.getLogger(__name__)


@router.post(
    "/{job_id}",
    summary="이미지 업로드",
    description="""
    결제 완료 후 사용자 이미지를 업로드합니다.

    처리 내용:
    1. 이미지 검증 (크기, 형식, 내용)
    2. S3 업로드
    3. JobFile 레코드 생성
    4. Job 상태 업데이트 (processing)
    5. AI 생성 Celery Task 시작
    """
)
async def upload_image(
    job_id: str,
    file: UploadFile = File(..., description="이미지 파일 (JPEG/PNG, 최대 10MB)"),
    crop_data: str = Form(..., description='크롭 데이터 JSON: {"x": 0, "y": 0, "width": 300, "height": 400}'),
    db: Session = Depends(get_db)
):
    """
    이미지 업로드 API 엔드포인트

    Args:
        job_id (str): Job ID (URL 파라미터)
        file (UploadFile): 업로드된 파일
        crop_data (str): 크롭 데이터 JSON 문자열
        db (Session): DB 세션

    Returns:
        dict: 업로드 결과
    """
    logger.info(f"[API] Image upload requested: job_id={job_id}, filename={file.filename}")

    try:
        # ===== Step 1: 파일 읽기 =====
        file_data = await file.read()
        logger.info(f"[API] File read: size={len(file_data)} bytes")

        # ===== Step 2: 이미지 서비스 호출 =====
        service = ImageService(db)
        result = service.upload_image(
            job_id=job_id,
            file_data=file_data,
            filename=file.filename,
            content_type=file.content_type,
            crop_data_str=crop_data
        )

        # ===== Step 3: Celery Task 실행 (AI 생성) =====
        from app.tasks.ai_generation import generate_ai_images_task
        generate_ai_images_task.delay(job_id=job_id)
        logger.info(f"[API] AI generation task queued to Celery")

        # ===== Step 4: 응답 반환 =====
        return {
            "success": True,
            "data": result
        }

    except AppException as e:
        logger.error(f"[API] AppException: {e.error_code} - {e.message}")
        raise

    except Exception as e:
        logger.error(f"[API] Unexpected error: {e}", exc_info=True)
        raise HTTPException(
            status_code=500,
            detail={
                "success": False,
                "error": {
                    "code": "E-SYS-999",
                    "message": "Internal server error"
                }
            }
        )
```

---

## ✅ 완전한 테스트 시나리오

### 테스트 1: 정상 케이스

```bash
curl -X POST http://localhost:8000/api/v1/uploads/550e8400-e29b-41d4-a716-446655440000 \
  -F "file=@test_image.jpg" \
  -F 'crop_data={"x": 100, "y": 50, "width": 300, "height": 400}'
```

**예상 응답 (200):**

```json
{
  "success": true,
  "data": {
    "file_id": "file-uuid-123",
    "s3_key": "jobs/550e8400-e29b-41d4-a716-446655440000/original/file-uuid-123.jpg",
    "message": "Image uploaded successfully. AI generation started."
  }
}
```

### 테스트 2: 파일 크기 초과

```bash
# 11MB 파일 업로드
curl -X POST http://localhost:8000/api/v1/uploads/550e8400-e29b-41d4-a716-446655440000 \
  -F "file=@large_image.jpg" \
  -F 'crop_data={"x": 0, "y": 0, "width": 300, "height": 400}'
```

**예상 응답 (400):**

```json
{
  "success": false,
  "error": {
    "code": "E-IMG-005",
    "message": "File size exceeds 10MB"
  }
}
```

---

## 🔍 구현 체크리스트

- [ ] `app/utils/image.py` - ImageValidator 클래스
- [ ] `app/services/s3_service.py` - S3Service 클래스
- [ ] `app/services/image_service.py` - ImageService 클래스
- [ ] `app/api/v1/uploads.py` - upload_image 엔드포인트
- [ ] `app/tasks/ai_generation.py` - generate_ai_images_task 함수
- [ ] JobFile 모델 정의
- [ ] FileType Enum 정의
- [ ] 단위 테스트 작성
- [ ] 통합 테스트 작성
- [ ] S3 버킷 생성 및 권한 설정
- [ ] 수동 테스트 (Postman)
