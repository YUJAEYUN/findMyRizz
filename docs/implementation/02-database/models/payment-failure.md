# PaymentFailure 모델

## 📋 목표

결제 실패 로그를 저장하는 모델을 정의합니다.

---

## 🔧 구현

### 파일: `app/models/payment_failure.py`

```python
"""
PaymentFailure 모델
"""
from sqlalchemy import Column, String, Text, ForeignKey
from sqlalchemy.orm import relationship

from app.core.database import Base
from app.models.base import TimestampMixin


class PaymentFailure(Base, TimestampMixin):
    """
    PaymentFailure 테이블
    
    Job과 1:N 관계
    결제 실패 로그 기록
    """
    __tablename__ = "payment_failures"
    
    # ===== 기본 필드 =====
    id = Column(String(36), primary_key=True)
    
    job_id = Column(
        String(36),
        ForeignKey("jobs.id", ondelete="CASCADE"),
        nullable=False,
        index=True
    )
    
    imp_uid = Column(
        String(100),
        nullable=True,
        comment="PortOne 결제 고유번호"
    )
    
    merchant_uid = Column(
        String(100),
        nullable=True,
        comment="가맹점 주문번호"
    )
    
    fail_reason = Column(
        Text,
        nullable=True,
        comment="실패 사유"
    )
    
    error_code = Column(
        String(50),
        nullable=True,
        comment="에러 코드"
    )
    
    # ===== 관계 =====
    job = relationship("Job", back_populates="payment_failures")
    
    def __repr__(self):
        return f"<PaymentFailure(id={self.id}, reason={self.fail_reason})>"
```

---

## 📊 테이블 스키마

```sql
CREATE TABLE payment_failures (
    id VARCHAR(36) PRIMARY KEY,
    job_id VARCHAR(36) NOT NULL,
    imp_uid VARCHAR(100),
    merchant_uid VARCHAR(100),
    fail_reason TEXT,
    error_code VARCHAR(50),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (job_id) REFERENCES jobs(id) ON DELETE CASCADE
);

CREATE INDEX idx_payment_failures_job_id ON payment_failures(job_id);
```

---

## 🔍 사용 예시

### 실패 로그 기록

```python
from app.models.payment_failure import PaymentFailure

failure = PaymentFailure(
    id=str(uuid.uuid4()),
    job_id=job_id,
    imp_uid="imp_123456789",
    merchant_uid="FMR_20240115_143022_ABC123",
    fail_reason="카드 한도 초과",
    error_code="CARD_LIMIT_EXCEEDED"
)
db.add(failure)
db.commit()
```

### 실패 로그 조회

```python
failures = db.query(PaymentFailure).filter(
    PaymentFailure.job_id == job_id
).all()

for failure in failures:
    print(f"Reason: {failure.fail_reason}")
```

---

## 📝 체크리스트

- [ ] app/models/payment_failure.py 생성
- [ ] PaymentFailure 클래스 정의
- [ ] Job 관계 설정
- [ ] 테스트 작성

---

## 🚀 다음 단계

PaymentFailure 모델 완료 → **knowledge-category.md**

