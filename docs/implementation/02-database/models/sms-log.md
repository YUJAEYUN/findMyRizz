# SmsLog 모델

## 📋 목표

SMS 발송 로그를 저장하는 SmsLog 모델을 정의합니다.

---

## 🔧 구현

### 파일: `app/models/sms_log.py`

```python
"""
SmsLog 모델
"""
from sqlalchemy import Column, String, Boolean, Text, ForeignKey
from sqlalchemy.orm import relationship

from app.core.database import Base
from app.models.base import TimestampMixin


class SmsLog(Base, TimestampMixin):
    """
    SmsLog 테이블
    
    Job과 1:N 관계
    """
    __tablename__ = "sms_logs"
    
    # ===== 기본 필드 =====
    id = Column(String(36), primary_key=True)
    
    job_id = Column(
        String(36),
        ForeignKey("jobs.id", ondelete="CASCADE"),
        nullable=False,
        index=True
    )
    
    to_number = Column(
        String(11),
        nullable=False,
        comment="수신 전화번호"
    )
    
    message = Column(
        Text,
        nullable=False,
        comment="발송 메시지"
    )
    
    success = Column(
        Boolean,
        nullable=False,
        default=False,
        comment="발송 성공 여부"
    )
    
    message_id = Column(
        String(100),
        nullable=True,
        comment="CoolSMS 메시지 ID"
    )
    
    error_message = Column(
        Text,
        nullable=True,
        comment="에러 메시지"
    )
    
    # ===== 관계 =====
    job = relationship("Job", back_populates="sms_logs")
    
    def __repr__(self):
        return f"<SmsLog(id={self.id}, to={self.to_number}, success={self.success})>"
```

---

## 📊 테이블 스키마

```sql
CREATE TABLE sms_logs (
    id VARCHAR(36) PRIMARY KEY,
    job_id VARCHAR(36) NOT NULL,
    to_number VARCHAR(11) NOT NULL,
    message TEXT NOT NULL,
    success BOOLEAN NOT NULL DEFAULT FALSE,
    message_id VARCHAR(100),
    error_message TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (job_id) REFERENCES jobs(id) ON DELETE CASCADE
);

CREATE INDEX idx_sms_logs_job_id ON sms_logs(job_id);
```

---

## 🔍 사용 예시

```python
from app.models.sms_log import SmsLog

# 성공 로그
sms_log = SmsLog(
    id=str(uuid.uuid4()),
    job_id=job_id,
    to_number="01012345678",
    message="결과가 준비되었습니다...",
    success=True,
    message_id="G4V20230115143022ABC123"
)
db.add(sms_log)
db.commit()

# 실패 로그
sms_log = SmsLog(
    id=str(uuid.uuid4()),
    job_id=job_id,
    to_number="01012345678",
    message="결과가 준비되었습니다...",
    success=False,
    error_message="Invalid phone number"
)
db.add(sms_log)
db.commit()
```

---

## 📝 체크리스트

- [ ] app/models/sms_log.py 생성
- [ ] SmsLog 클래스 정의
- [ ] Job 관계 설정
- [ ] 테스트 작성

---

## 🚀 다음 단계

SmsLog 모델 완료 → **phone-verification.md**

