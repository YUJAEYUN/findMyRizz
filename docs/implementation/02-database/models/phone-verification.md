# PhoneVerificationAttempt 모델

## 📋 목표

전화번호 인증 시도를 기록하는 모델을 정의합니다.

---

## 🔧 구현

### 파일: `app/models/phone_verification.py`

```python
"""
PhoneVerificationAttempt 모델
"""
from sqlalchemy import Column, String, Boolean, ForeignKey
from sqlalchemy.orm import relationship

from app.core.database import Base
from app.models.base import TimestampMixin


class PhoneVerificationAttempt(Base, TimestampMixin):
    """
    PhoneVerificationAttempt 테이블
    
    Job과 1:N 관계
    전화번호 인증 시도 기록 (성공/실패)
    """
    __tablename__ = "phone_verification_attempts"
    
    # ===== 기본 필드 =====
    id = Column(String(36), primary_key=True)
    
    job_id = Column(
        String(36),
        ForeignKey("jobs.id", ondelete="CASCADE"),
        nullable=False,
        index=True
    )
    
    ip_address = Column(
        String(45),
        nullable=False,
        index=True,
        comment="요청 IP 주소 (IPv4: 15자, IPv6: 45자)"
    )
    
    success = Column(
        Boolean,
        nullable=False,
        default=False,
        comment="인증 성공 여부"
    )
    
    # ===== 관계 =====
    job = relationship("Job", back_populates="phone_verification_attempts")
    
    def __repr__(self):
        return f"<PhoneVerificationAttempt(id={self.id}, ip={self.ip_address}, success={self.success})>"
```

---

## 📊 테이블 스키마

```sql
CREATE TABLE phone_verification_attempts (
    id VARCHAR(36) PRIMARY KEY,
    job_id VARCHAR(36) NOT NULL,
    ip_address VARCHAR(45) NOT NULL,
    success BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (job_id) REFERENCES jobs(id) ON DELETE CASCADE
);

CREATE INDEX idx_phone_verification_job_id ON phone_verification_attempts(job_id);
CREATE INDEX idx_phone_verification_ip ON phone_verification_attempts(ip_address);
```

---

## 🔍 사용 예시

### 실패 시도 기록

```python
from app.models.phone_verification import PhoneVerificationAttempt

attempt = PhoneVerificationAttempt(
    id=str(uuid.uuid4()),
    job_id=job_id,
    ip_address="192.168.1.1",
    success=False
)
db.add(attempt)
db.commit()
```

### IP별 실패 횟수 조회

```python
from datetime import datetime, timedelta

one_hour_ago = datetime.utcnow() - timedelta(hours=1)

failed_count = db.query(PhoneVerificationAttempt).filter(
    PhoneVerificationAttempt.job_id == job_id,
    PhoneVerificationAttempt.ip_address == "192.168.1.1",
    PhoneVerificationAttempt.success == False,
    PhoneVerificationAttempt.created_at >= one_hour_ago
).count()

print(f"Failed attempts: {failed_count}")
```

---

## 📝 체크리스트

- [ ] app/models/phone_verification.py 생성
- [ ] PhoneVerificationAttempt 클래스 정의
- [ ] Job 관계 설정
- [ ] 테스트 작성

---

## 🚀 다음 단계

PhoneVerificationAttempt 모델 완료 → **payment-failure.md**

