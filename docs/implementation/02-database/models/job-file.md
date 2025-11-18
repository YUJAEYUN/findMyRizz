# JobFile 모델

## 📋 목표

업로드된 파일 정보를 저장하는 JobFile 모델을 정의합니다.

---

## 🔧 구현

### 파일: `app/models/job_file.py`

```python
"""
JobFile 모델
"""
from sqlalchemy import Column, String, Integer, Enum as SQLEnum, Text, ForeignKey
from sqlalchemy.orm import relationship
import enum

from app.core.database import Base
from app.models.base import TimestampMixin


class FileType(str, enum.Enum):
    """파일 타입"""
    ORIGINAL = "original"      # 원본 이미지
    GENERATED = "generated"    # AI 생성 이미지
    THUMBNAIL = "thumbnail"    # 썸네일 이미지


class JobFile(Base, TimestampMixin):
    """
    JobFile 테이블

    Job과 1:N 관계
    """
    __tablename__ = "job_files"

    # ===== 기본 필드 =====
    id = Column(String(36), primary_key=True)

    job_id = Column(
        String(36),
        ForeignKey("jobs.id", ondelete="CASCADE"),
        nullable=False,
        index=True
    )

    file_type = Column(
        SQLEnum(FileType),
        nullable=False,
        index=True
    )

    s3_key = Column(
        String(500),
        nullable=False,
        unique=True,
        comment="S3 객체 키"
    )

    display_order = Column(
        Integer,
        nullable=False,
        default=0,
        comment="표시 순서 (0: 원본, 1-3: AI 생성)"
    )

    crop_data = Column(
        Text,
        nullable=True,
        comment="크롭 데이터 (JSON)"
    )

    prediction_id = Column(
        String(100),
        nullable=True,
        comment="Replicate prediction ID"
    )

    seed = Column(
        Integer,
        nullable=True,
        comment="AI 생성 시 사용한 seed"
    )

    # ===== 관계 =====
    job = relationship("Job", back_populates="files")

    def __repr__(self):
        return f"<JobFile(id={self.id}, type={self.file_type.value}, order={self.display_order})>"
```

---

## 📊 테이블 스키마

```sql
CREATE TABLE job_files (
    id VARCHAR(36) PRIMARY KEY,
    job_id VARCHAR(36) NOT NULL,
    file_type VARCHAR(20) NOT NULL,
    s3_key VARCHAR(500) NOT NULL UNIQUE,
    display_order INTEGER NOT NULL DEFAULT 0,
    crop_data TEXT,
    prediction_id VARCHAR(100),
    seed INTEGER,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (job_id) REFERENCES jobs(id) ON DELETE CASCADE
);

CREATE INDEX idx_job_files_job_id ON job_files(job_id);
CREATE INDEX idx_job_files_type ON job_files(file_type);
```

---

## 🔍 사용 예시

### 원본 이미지 저장

```python
from app.models.job_file import JobFile, FileType
import json

job_file = JobFile(
    id=str(uuid.uuid4()),
    job_id=job_id,
    file_type=FileType.ORIGINAL,
    s3_key="jobs/job-id/original/file-id.jpg",
    display_order=0,
    crop_data=json.dumps({"x": 0, "y": 0, "width": 300, "height": 400})
)
db.add(job_file)
db.commit()
```

### AI 생성 이미지 저장

```python
job_file = JobFile(
    id=str(uuid.uuid4()),
    job_id=job_id,
    file_type=FileType.GENERATED,
    s3_key="jobs/job-id/generated/file-id.jpg",
    display_order=1,
    prediction_id="pred_abc123",
    seed=123456
)
db.add(job_file)
db.commit()
```

---

## 📝 체크리스트

- [ ] app/models/job_file.py 생성
- [ ] FileType Enum 정의
- [ ] JobFile 클래스 정의
- [ ] Job 관계 설정
- [ ] 테스트 작성

---

## 🚀 다음 단계

JobFile 모델 완료 → **sms-log.md**
