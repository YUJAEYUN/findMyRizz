# Payment API 엔드포인트

## 📋 목표

결제 초기화 및 Webhook API를 구현합니다.

---

## 🔧 구현

### 파일: `app/api/v1/payment.py`

```python
"""
결제 API 엔드포인트
"""
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
import logging

from app.core.database import get_db
from app.schemas.payment import (
    PaymentInitializeRequest,
    PaymentInitializeResponse,
    PortOneWebhookRequest,
    PaymentVerificationResponse
)
from app.services.payment_service import PaymentService
from app.core.exceptions import AppException

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/payments", tags=["payments"])


@router.post("/initialize", response_model=PaymentInitializeResponse)
async def initialize_payment(
    request: PaymentInitializeRequest,
    db: Session = Depends(get_db)
):
    """
    결제 초기화

    처리:
    1. 금액 검증 (Pydantic validator)
    2. Job 생성 (전화번호 NULL)
    3. Payment 생성
    4. 응답 반환

    Args:
        request: {
            "amount": 9900
        }

    Returns:
        {
            "job_id": str,
            "merchant_uid": str,
            "amount": int,
            "payment_name": str
        }
    """
    try:
        logger.info(f"[PaymentAPI] Initialize: amount={request.amount}")

        # 서비스 호출
        service = PaymentService(db)
        result = service.initialize_payment(request.amount)

        return PaymentInitializeResponse(**result)

    except AppException as e:
        logger.error(f"[PaymentAPI] Failed: {e.message}")
        raise HTTPException(status_code=e.status_code, detail=e.message)
    except Exception as e:
        logger.error(f"[PaymentAPI] Unexpected error: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="Internal server error")


@router.post("/webhooks/portone", response_model=PaymentVerificationResponse)
async def portone_webhook(
    webhook: PortOneWebhookRequest,
    db: Session = Depends(get_db)
):
    """
    PortOne Webhook 수신

    처리:
    1. merchant_uid로 Payment 조회
    2. PortOne API로 검증
    3. 금액 일치 확인
    4. Payment 상태 업데이트
    5. Job 상태 업데이트

    Args:
        webhook: {
            "imp_uid": str,
            "merchant_uid": str,
            "status": str,
            "amount": int,
            "paid_at": int
        }

    Returns:
        {
            "success": bool,
            "job_id": str,
            "status": str,
            "message": str
        }
    """
    try:
        logger.info(f"[PaymentAPI] Webhook: {webhook.merchant_uid}")

        # 서비스 호출
        service = PaymentService(db)
        result = service.process_webhook(webhook.dict())

        return PaymentVerificationResponse(
            success=result["success"],
            job_id=result["job_id"],
            status="paid",
            message=result["message"]
        )

    except AppException as e:
        logger.error(f"[PaymentAPI] Webhook failed: {e.message}")
        raise HTTPException(status_code=e.status_code, detail=e.message)
    except Exception as e:
        logger.error(f"[PaymentAPI] Unexpected error: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="Internal server error")
```

---

## 🔧 라우터 등록

### 파일: `app/api/v1/__init__.py`

```python
"""
API v1 라우터
"""
from fastapi import APIRouter
from app.api.v1 import payment

api_router = APIRouter()

api_router.include_router(payment.router)
```

### 파일: `main.py`

```python
from app.api.v1 import api_router

app.include_router(api_router, prefix="/api/v1")
```

---

## 🔍 API 사용 예시

### 1. 결제 초기화

```bash
curl -X POST http://localhost:8000/api/v1/payments/initialize \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 9900
  }'
```

**응답:**

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "merchant_uid": "FMR_20240115_143022_ABC123",
  "amount": 9900,
  "payment_name": "Find My Rizz AI 분석"
}
```

### 2. Webhook (PortOne → 서버)

```bash
curl -X POST http://localhost:8000/api/v1/payments/webhooks/portone \
  -H "Content-Type: application/json" \
  -d '{
    "imp_uid": "imp_123456789",
    "merchant_uid": "FMR_20240115_143022_ABC123",
    "status": "paid",
    "amount": 9900,
    "paid_at": 1705305022
  }'
```

**응답:**

```json
{
  "success": true,
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "paid",
  "message": "Payment verified successfully"
}
```

---

## ✅ 테스트 시나리오

### 1. 결제 초기화 성공

```python
def test_initialize_payment_success(client):
    response = client.post(
        "/api/v1/payments/initialize",
        json={"amount": 9900}
    )

    assert response.status_code == 200
    data = response.json()
    assert data["amount"] == 9900
    assert "job_id" in data
```

### 2. 잘못된 금액

```python
def test_initialize_invalid_amount(client):
    response = client.post(
        "/api/v1/payments/initialize",
        json={"amount": 10000}
    )

    assert response.status_code == 422  # Validation error
```

### 3. Webhook 성공

```python
def test_webhook_success(client):
    response = client.post(
        "/api/v1/payments/webhooks/portone",
        json={
            "imp_uid": "imp_123",
            "merchant_uid": "FMR_...",
            "status": "paid",
            "amount": 9900,
            "paid_at": 1705305022
        }
    )

    assert response.status_code == 200
    data = response.json()
    assert data["success"] == True
```

---

## 📝 체크리스트

- [ ] app/api/v1/payment.py 생성
- [ ] initialize_payment 엔드포인트 구현
- [ ] portone_webhook 엔드포인트 구현
- [ ] 라우터 등록
- [ ] 에러 처리
- [ ] 테스트 작성

---

## 🚀 다음 단계

Payment API 완료 → **Phase 4: Image Upload**
