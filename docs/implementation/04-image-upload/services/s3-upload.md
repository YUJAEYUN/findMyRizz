# S3 업로드 서비스

## 📋 목표

AWS S3에 파일을 업로드하고 Presigned URL을 생성합니다.

---

## 🔧 구현

### 파일: `app/services/s3_service.py`

```python
"""
AWS S3 파일 관리 서비스
"""
import boto3
from botocore.exceptions import ClientError
import logging
from typing import Optional

from app.config import settings
from app.core.exceptions import AppException

logger = logging.getLogger(__name__)


class S3Service:
    """S3 파일 관리"""

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
            file_data: 파일 데이터
            s3_key: S3 객체 키
                예: "jobs/job-id/original/file-id.jpg"
            content_type: Content-Type
            metadata: 추가 메타데이터

        Returns:
            str: S3 키

        Raises:
            AppException(E-FILE-001): 업로드 실패
        """
        try:
            logger.info(f"[S3Service] Uploading: {s3_key}, size={len(file_data)}")

            # 업로드 파라미터
            upload_params = {
                'Bucket': self.bucket_name,
                'Key': s3_key,
                'Body': file_data,
                'ContentType': content_type
            }

            # 메타데이터 추가
            if metadata:
                str_metadata = {k: str(v) for k, v in metadata.items()}
                upload_params['Metadata'] = str_metadata

            # S3 업로드
            self.s3_client.put_object(**upload_params)

            logger.info(f"[S3Service] Upload success: {s3_key}")
            return s3_key

        except ClientError as e:
            logger.error(f"[S3Service] Upload failed: {e}", exc_info=True)
            raise AppException(
                status_code=500,
                error_code="E-FILE-001",
                message="Failed to upload file to S3"
            )

    def generate_presigned_url(
        self,
        s3_key: str,
        expiration: int = 3600
    ) -> str:
        """
        Presigned URL 생성

        Args:
            s3_key: S3 객체 키
            expiration: 유효 시간 (초)

        Returns:
            str: Presigned URL
        """
        try:
            url = self.s3_client.generate_presigned_url(
                'get_object',
                Params={
                    'Bucket': self.bucket_name,
                    'Key': s3_key
                },
                ExpiresIn=expiration
            )
            logger.info(f"[S3Service] Presigned URL generated: {s3_key}")
            return url

        except ClientError as e:
            logger.error(f"[S3Service] Presigned URL failed: {e}")
            raise AppException(
                status_code=500,
                error_code="E-FILE-002",
                message="Failed to generate presigned URL"
            )

    def delete_file(self, s3_key: str) -> bool:
        """
        S3 파일 삭제

        Args:
            s3_key: S3 객체 키

        Returns:
            bool: 성공 여부
        """
        try:
            self.s3_client.delete_object(
                Bucket=self.bucket_name,
                Key=s3_key
            )
            logger.info(f"[S3Service] Deleted: {s3_key}")
            return True

        except ClientError as e:
            logger.error(f"[S3Service] Delete failed: {e}")
            return False

    def generate_s3_key(
        self,
        job_id: str,
        file_id: str,
        file_type: str,
        extension: str
    ) -> str:
        """
        S3 키 생성

        형식: jobs/{job_id}/{file_type}/{file_id}.{extension}

        Args:
            job_id: Job ID
            file_id: File ID
            file_type: "original", "generated", "thumbnail"
            extension: "jpg", "png"

        Returns:
            str: S3 키
        """
        s3_key = f"jobs/{job_id}/{file_type}/{file_id}.{extension}"
        return s3_key
```

---

## 🔍 S3 키 구조

```
jobs/
├── {job_id}/
│   ├── original/
│   │   └── {file_id}.jpg          # 원본 이미지
│   ├── generated/
│   │   ├── {file_id_1}.jpg        # AI 생성 1
│   │   ├── {file_id_2}.jpg        # AI 생성 2
│   │   └── {file_id_3}.jpg        # AI 생성 3
│   └── thumbnail/
│       └── {file_id}.jpg          # 썸네일 이미지
```

---

## 🔍 사용 예시

### 파일 업로드

```python
from app.services.s3_service import S3Service

s3_service = S3Service()

# S3 키 생성
s3_key = s3_service.generate_s3_key(
    job_id="job-uuid",
    file_id="file-uuid",
    file_type="original",
    extension="jpg"
)
# → "jobs/job-uuid/original/file-uuid.jpg"

# 업로드
s3_service.upload_file(
    file_data=image_bytes,
    s3_key=s3_key,
    content_type="image/jpeg",
    metadata={"width": "1024", "height": "768"}
)
```

### Presigned URL 생성

```python
# 1시간 유효한 URL
url = s3_service.generate_presigned_url(
    s3_key="jobs/job-uuid/original/file-uuid.jpg",
    expiration=3600
)

print(url)
# https://s3.ap-northeast-2.amazonaws.com/bucket/jobs/...?X-Amz-...
```

---

## ✅ 테스트 시나리오

### 1. 업로드 성공

```python
def test_upload_success():
    s3_service = S3Service()
    s3_key = s3_service.upload_file(
        file_data=b"test data",
        s3_key="test/file.jpg",
        content_type="image/jpeg"
    )
    assert s3_key == "test/file.jpg"
```

### 2. Presigned URL 생성

```python
def test_presigned_url():
    s3_service = S3Service()
    url = s3_service.generate_presigned_url("test/file.jpg")
    assert "X-Amz-" in url
```

---

## 📝 체크리스트

- [ ] app/services/s3_service.py 생성
- [ ] S3Service 클래스 구현
- [ ] upload_file() 구현
- [ ] generate_presigned_url() 구현
- [ ] delete_file() 구현
- [ ] generate_s3_key() 구현
- [ ] 테스트 작성

---

## 🚀 다음 단계

S3 업로드 완료 → **image-handler.md**
