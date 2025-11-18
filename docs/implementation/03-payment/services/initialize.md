# 결제 초기화 서비스

## 📋 목표

사용자 전화번호를 받아 결제를 초기화하고 Job을 생성합니다.

---

## 🔧 구현

### 파일: `app/services/payment_service.py`

```python
"""
결제 비즈니스 로직
"""
from sqlalchemy.orm import Session
from datetime import datetime, timedelta
import uuid
import random
import string
import logging

from app.models.job import Job, JobStatus
from app.models.payment import Payment, PaymentStatus
from app.config import settings
from app.core.exceptions import AppException

logger = logging.getLogger(__name__)


class PaymentService:
    """결제 서비스"""
    
    def __init__(self, db: Session):
        self.db = db
    
    def initialize_payment(self, phone_number: str) -> dict:
        """
        결제 초기화
        
        처리 흐름:
        1. merchant_uid 생성
        2. Job 생성 (status=pending_payment)
        3. Payment 생성 (status=pending)
        4. DB 커밋
        5. 응답 반환
        
        Args:
            phone_number: 검증된 전화번호 (11자리)
            
        Returns:
            {
                "job_id": str,
                "merchant_uid": str,
                "amount": int,
                "payment_name": str,
                "buyer_tel": str
            }
            
        Raises:
            AppException(E-PAY-002): DB 저장 실패
        """
        try:
            # Step 1: merchant_uid 생성
            merchant_uid = self._generate_merchant_uid()
            logger.info(f"[PaymentService] Generated merchant_uid: {merchant_uid}")
            
            # Step 2: Job 생성
            job_id = str(uuid.uuid4())
            current_time = datetime.utcnow()
            expires_at = current_time + timedelta(hours=settings.JOB_EXPIRATION_HOURS)
            
            job = Job(
                id=job_id,
                user_phone_number=phone_number,
                status=JobStatus.PENDING_PAYMENT,
                expires_at=expires_at,
                created_at=current_time,
                updated_at=current_time
            )
            self.db.add(job)
            logger.info(f"[PaymentService] Created Job: {job_id}")
            
            # Step 3: Payment 생성
            payment_id = str(uuid.uuid4())
            payment = Payment(
                id=payment_id,
                job_id=job_id,
                merchant_uid=merchant_uid,
                amount=settings.PAYMENT_AMOUNT,
                status=PaymentStatus.PENDING,
                created_at=current_time,
                updated_at=current_time
            )
            self.db.add(payment)
            logger.info(f"[PaymentService] Created Payment: {payment_id}")
            
            # Step 4: DB 커밋
            self.db.commit()
            logger.info(f"[PaymentService] Payment initialized: job_id={job_id}")
            
            # Step 5: 응답 반환
            return {
                "job_id": job_id,
                "merchant_uid": merchant_uid,
                "amount": settings.PAYMENT_AMOUNT,
                "payment_name": "Find My Rizz AI 분석",
                "buyer_tel": phone_number
            }
            
        except Exception as e:
            self.db.rollback()
            logger.error(f"[PaymentService] Failed to initialize: {e}", exc_info=True)
            raise AppException(
                status_code=500,
                error_code="E-PAY-002",
                message="Failed to initialize payment"
            )
    
    def _generate_merchant_uid(self) -> str:
        """
        merchant_uid 생성
        
        형식: FMR_YYYYMMDD_HHMMSS_RANDOM6
        예시: FMR_20240115_143022_ABC123
        
        Returns:
            str: merchant_uid
        """
        now = datetime.utcnow()
        date_part = now.strftime("%Y%m%d")
        time_part = now.strftime("%H%M%S")
        
        # 랜덤 6자리 (대문자 + 숫자)
        random_chars = string.ascii_uppercase + string.digits
        random_part = ''.join(random.choices(random_chars, k=6))
        
        merchant_uid = f"FMR_{date_part}_{time_part}_{random_part}"
        
        return merchant_uid
```

---

## 🔍 상세 설명

### merchant_uid 생성 규칙

```python
# 형식: FMR_YYYYMMDD_HHMMSS_RANDOM6
# 예시: FMR_20240115_143022_ABC123

# 구성:
# - FMR: 서비스 prefix
# - 20240115: 날짜 (UTC)
# - 143022: 시간 (UTC)
# - ABC123: 랜덤 6자리 (충돌 방지)
```

### Job 생성

```python
job = Job(
    id=str(uuid.uuid4()),              # UUID v4
    user_phone_number="01012345678",   # 11자리
    status=JobStatus.PENDING_PAYMENT,  # Enum
    expires_at=now + timedelta(hours=24)  # 24시간 후
)
```

### Payment 생성

```python
payment = Payment(
    id=str(uuid.uuid4()),
    job_id=job_id,                     # FK
    merchant_uid=merchant_uid,         # 고유 주문번호
    amount=9900,                       # 고정 금액
    status=PaymentStatus.PENDING       # Enum
)
```

---

## 🔍 사용 예시

```python
from app.services.payment_service import PaymentService
from app.core.database import get_db

db = next(get_db())
service = PaymentService(db)

result = service.initialize_payment("01012345678")

print(result)
# {
#     "job_id": "550e8400-e29b-41d4-a716-446655440000",
#     "merchant_uid": "FMR_20240115_143022_ABC123",
#     "amount": 9900,
#     "payment_name": "Find My Rizz AI 분석",
#     "buyer_tel": "01012345678"
# }
```

---

## ✅ 테스트 시나리오

### 1. 정상 케이스

```python
def test_initialize_payment_success():
    service = PaymentService(db)
    result = service.initialize_payment("01012345678")
    
    assert result["amount"] == 9900
    assert result["merchant_uid"].startswith("FMR_")
    assert len(result["job_id"]) == 36  # UUID
```

### 2. DB 오류

```python
def test_initialize_payment_db_error(mocker):
    mocker.patch.object(db, 'commit', side_effect=Exception("DB error"))
    
    service = PaymentService(db)
    
    with pytest.raises(AppException) as exc:
        service.initialize_payment("01012345678")
    
    assert exc.value.error_code == "E-PAY-002"
```

---

## 📝 체크리스트

- [ ] PaymentService 클래스 생성
- [ ] initialize_payment() 메서드 구현
- [ ] _generate_merchant_uid() 메서드 구현
- [ ] 에러 처리 구현
- [ ] 로깅 추가
- [ ] 단위 테스트 작성

---

## 🚀 다음 단계

결제 초기화 완료 → **webhook.md** (Webhook 처리)

