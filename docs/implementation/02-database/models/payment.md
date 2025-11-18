# Payment 모델

## 📋 목표

결제 정보를 저장하는 Payment 모델을 정의합니다.

---

## 🔧 구현

### 파일: `app/models/payment.py`

```python
"""
Payment 모델
"""
from sqlalchemy import Column, String, Integer, Enum as SQLEnum, DateTime, ForeignKey
from sqlalchemy.orm import relationship
import enum

from app.core.database import Base
from app.models.base import TimestampMixin


class PaymentStatus(str, enum.Enum):
    """결제 상태"""
    PENDING = "pending"          # 결제 대기
    PAID = "paid"                # 결제 완료
    FAILED = "failed"            # 결제 실패
    CANCELLED = "cancelled"      # 결제 취소
    REFUNDED = "refunded"        # 환불 완료


class Payment(Base, TimestampMixin):
    """
    Payment 테이블
    
    Job과 1:1 관계
    """
    __tablename__ = "payments"
    
    # ===== 기본 필드 =====
    id = Column(String(36), primary_key=True)
    
    job_id = Column(
        String(36),
        ForeignKey("jobs.id", ondelete="CASCADE"),
        nullable=False,
        unique=True,
        index=True
    )
    
    merchant_uid = Column(
        String(100),
        nullable=False,
        unique=True,
        index=True,
        comment="가맹점 주문번호"
    )
    
    imp_uid = Column(
        String(100),
        nullable=True,
        unique=True,
        index=True,
        comment="PortOne 결제 고유번호"
    )
    
    amount = Column(
        Integer,
        nullable=False,
        comment="결제 금액"
    )
    
    status = Column(
        SQLEnum(PaymentStatus),
        nullable=False,
        default=PaymentStatus.PENDING,
        index=True
    )
    
    paid_at = Column(
        DateTime,
        nullable=True,
        comment="결제 완료 시간"
    )
    
    # ===== 관계 =====
    job = relationship("Job", back_populates="payment")
    
    def __repr__(self):
        return f"<Payment(id={self.id}, merchant_uid={self.merchant_uid}, status={self.status.value})>"
```

---

## 📊 테이블 스키마

```sql
CREATE TABLE payments (
    id VARCHAR(36) PRIMARY KEY,
    job_id VARCHAR(36) NOT NULL UNIQUE,
    merchant_uid VARCHAR(100) NOT NULL UNIQUE,
    imp_uid VARCHAR(100) UNIQUE,
    amount INTEGER NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    paid_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (job_id) REFERENCES jobs(id) ON DELETE CASCADE
);

CREATE INDEX idx_payments_job_id ON payments(job_id);
CREATE INDEX idx_payments_merchant_uid ON payments(merchant_uid);
CREATE INDEX idx_payments_imp_uid ON payments(imp_uid);
CREATE INDEX idx_payments_status ON payments(status);
```

---

## 🔍 사용 예시

```python
from app.models.payment import Payment, PaymentStatus
from datetime import datetime

# Payment 생성
payment = Payment(
    id=str(uuid.uuid4()),
    job_id=job_id,
    merchant_uid="FMR_20240115_143022_ABC123",
    amount=9900,
    status=PaymentStatus.PENDING
)
db.add(payment)
db.commit()

# 결제 완료 처리
payment.imp_uid = "imp_123456789"
payment.status = PaymentStatus.PAID
payment.paid_at = datetime.utcnow()
db.commit()
```

---

## 📝 체크리스트

- [ ] app/models/payment.py 생성
- [ ] PaymentStatus Enum 정의
- [ ] Payment 클래스 정의
- [ ] Job 관계 설정
- [ ] 테스트 작성

---

## 🚀 다음 단계

Payment 모델 완료 → **job-file.md**

