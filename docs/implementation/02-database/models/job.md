# Job 모델

## 📋 목표

중심 테이블인 Job 모델을 정의합니다.

---

## 🔧 구현

### 파일: `app/models/job.py`

```python
"""
Job 모델 - 중심 테이블
"""
from sqlalchemy import Column, String, Enum as SQLEnum, DateTime
from sqlalchemy.orm import relationship
from datetime import datetime
import enum

from app.core.database import Base
from app.models.base import TimestampMixin


class JobStatus(str, enum.Enum):
    """
    Job 상태

    흐름:
    pending_payment → pending_phone → pending_upload → processing → completed → expired
    """
    PENDING_PAYMENT = "pending_payment"  # 결제 대기
    PENDING_PHONE = "pending_phone"      # 전화번호 입력 대기
    PENDING_UPLOAD = "pending_upload"    # 이미지 업로드 대기
    PROCESSING = "processing"            # AI 생성 및 분석 중
    COMPLETED = "completed"              # 완료
    FAILED = "failed"                    # 실패
    EXPIRED = "expired"                  # 만료 (24시간 경과)


class Job(Base, TimestampMixin):
    """
    Job 테이블

    사용자의 전체 작업을 관리하는 중심 테이블
    """
    __tablename__ = "jobs"

    # ===== 기본 필드 =====
    id = Column(
        String(36),
        primary_key=True,
        comment="Job ID (UUID)"
    )

    user_phone_number = Column(
        String(11),
        nullable=False,
        index=True,
        comment="사용자 전화번호 (01012345678)"
    )

    status = Column(
        SQLEnum(JobStatus),
        nullable=False,
        default=JobStatus.PENDING_PAYMENT,
        index=True,
        comment="Job 상태"
    )

    expires_at = Column(
        DateTime,
        nullable=False,
        index=True,
        comment="만료 시간 (생성 후 24시간)"
    )

    # ===== 관계 =====
    # 1:1 관계
    payment = relationship(
        "Payment",
        back_populates="job",
        uselist=False,
        cascade="all, delete-orphan"
    )

    report = relationship(
        "Report",
        back_populates="job",
        uselist=False,
        cascade="all, delete-orphan"
    )

    # 1:N 관계
    files = relationship(
        "JobFile",
        back_populates="job",
        cascade="all, delete-orphan",
        order_by="JobFile.display_order"
    )

    sms_logs = relationship(
        "SmsLog",
        back_populates="job",
        cascade="all, delete-orphan"
    )

    phone_verification_attempts = relationship(
        "PhoneVerificationAttempt",
        back_populates="job",
        cascade="all, delete-orphan"
    )

    payment_failures = relationship(
        "PaymentFailure",
        back_populates="job",
        cascade="all, delete-orphan"
    )

    def __repr__(self):
        return f"<Job(id={self.id}, status={self.status.value}, phone={self.user_phone_number})>"

    def is_expired(self) -> bool:
        """
        만료 여부 확인

        Returns:
            bool: 만료되었으면 True
        """
        return datetime.utcnow() > self.expires_at

    def can_upload(self) -> bool:
        """
        업로드 가능 여부

        Returns:
            bool: 업로드 가능하면 True
        """
        return self.status == JobStatus.PENDING_UPLOAD and not self.is_expired()

    def can_view_result(self) -> bool:
        """
        결과 조회 가능 여부

        Returns:
            bool: 조회 가능하면 True
        """
        return self.status == JobStatus.COMPLETED and not self.is_expired()
```

---

## 📊 테이블 스키마

```sql
CREATE TABLE jobs (
    id VARCHAR(36) PRIMARY KEY,
    user_phone_number VARCHAR(11) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending_payment',
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_jobs_phone ON jobs(user_phone_number);
CREATE INDEX idx_jobs_status ON jobs(status);
CREATE INDEX idx_jobs_expires_at ON jobs(expires_at);
```

---

## 🔍 사용 예시

### Job 생성

```python
from app.models.job import Job, JobStatus
from datetime import datetime, timedelta
import uuid

# Job 생성
job = Job(
    id=str(uuid.uuid4()),
    user_phone_number="01012345678",
    status=JobStatus.PENDING_PAYMENT,
    expires_at=datetime.utcnow() + timedelta(hours=24)
)

db.add(job)
db.commit()
```

### Job 조회

```python
# ID로 조회
job = db.query(Job).filter(Job.id == job_id).first()

# 전화번호로 조회
jobs = db.query(Job).filter(
    Job.user_phone_number == "01012345678"
).all()

# 상태로 조회
pending_jobs = db.query(Job).filter(
    Job.status == JobStatus.PENDING_UPLOAD
).all()
```

### Job 상태 변경

```python
job.status = JobStatus.GENERATING
job.updated_at = datetime.utcnow()
db.commit()
```

---

## 📝 체크리스트

- [ ] app/models/job.py 생성
- [ ] JobStatus Enum 정의
- [ ] Job 클래스 정의
- [ ] 관계 설정 (payment, files, etc.)
- [ ] 헬퍼 메서드 구현
- [ ] 테스트 작성

---

## 🚀 다음 단계

Job 모델 완료 → **payment.md** (Payment 모델)
