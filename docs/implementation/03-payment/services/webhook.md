# 결제 Webhook 처리

## 📋 목표

PortOne Webhook을 수신하고 결제를 검증합니다.

---

## 🔧 구현

### 파일: `app/services/payment_service.py` (추가)

```python
def process_webhook(self, webhook_data: dict) -> dict:
    """
    PortOne Webhook 처리

    처리 흐름:
    1. merchant_uid로 Payment 조회
    2. imp_uid로 PortOne API 호출하여 검증
    3. 금액 일치 확인
    4. Payment 상태 업데이트
    5. Job 상태 업데이트 (pending_upload)

    Args:
        webhook_data: {
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
            "message": str
        }

    Raises:
        AppException(E-PAY-003): Payment 없음
        AppException(E-PAY-004): 금액 불일치
        AppException(E-PAY-005): 검증 실패
    """
    try:
        # Step 1: merchant_uid로 Payment 조회
        merchant_uid = webhook_data.get("merchant_uid")
        payment = self.db.query(Payment).filter(
            Payment.merchant_uid == merchant_uid
        ).first()

        if not payment:
            logger.error(f"[PaymentService] Payment not found: {merchant_uid}")
            raise AppException(
                status_code=404,
                error_code="E-PAY-003",
                message="Payment not found"
            )

        logger.info(f"[PaymentService] Processing webhook: {merchant_uid}")

        # Step 2: PortOne API로 검증
        imp_uid = webhook_data.get("imp_uid")
        verified_data = self._verify_payment_with_portone(imp_uid)

        # Step 3: 금액 일치 확인
        if verified_data["amount"] != payment.amount:
            logger.error(
                f"[PaymentService] Amount mismatch: "
                f"expected={payment.amount}, actual={verified_data['amount']}"
            )
            raise AppException(
                status_code=400,
                error_code="E-PAY-004",
                message="Payment amount mismatch"
            )

        # Step 4: Payment 상태 업데이트
        payment.imp_uid = imp_uid
        payment.status = PaymentStatus.PAID
        payment.paid_at = datetime.fromtimestamp(webhook_data.get("paid_at"))

        # Step 5: Job 상태 업데이트 (pending_phone: 전화번호 입력 대기)
        job = payment.job
        job.status = JobStatus.PENDING_PHONE
        job.updated_at = get_kst_now()

        self.db.commit()
        logger.info(f"[PaymentService] Webhook processed: job_id={job.id}")

        return {
            "success": True,
            "job_id": job.id,
            "message": "Payment verified successfully"
        }

    except AppException:
        self.db.rollback()
        raise
    except Exception as e:
        self.db.rollback()
        logger.error(f"[PaymentService] Webhook failed: {e}", exc_info=True)
        raise AppException(
            status_code=500,
            error_code="E-PAY-005",
            message="Failed to process webhook"
        )

def _verify_payment_with_portone(self, imp_uid: str) -> dict:
    """
    PortOne API로 결제 검증

    Args:
        imp_uid: PortOne 결제 고유번호

    Returns:
        {
            "imp_uid": str,
            "merchant_uid": str,
            "amount": int,
            "status": str
        }
    """
    import requests

    # Step 1: Access Token 발급
    token_url = "https://api.iamport.kr/users/getToken"
    token_data = {
        "imp_key": settings.PORTONE_API_KEY,
        "imp_secret": settings.PORTONE_API_SECRET
    }

    token_response = requests.post(token_url, json=token_data)
    access_token = token_response.json()["response"]["access_token"]

    # Step 2: 결제 정보 조회
    payment_url = f"https://api.iamport.kr/payments/{imp_uid}"
    headers = {"Authorization": access_token}

    payment_response = requests.get(payment_url, headers=headers)
    payment_data = payment_response.json()["response"]

    return {
        "imp_uid": payment_data["imp_uid"],
        "merchant_uid": payment_data["merchant_uid"],
        "amount": payment_data["amount"],
        "status": payment_data["status"]
    }
```

---

## 🔍 Webhook 데이터 형식

```json
{
  "imp_uid": "imp_123456789",
  "merchant_uid": "FMR_20240115_143022_ABC123",
  "status": "paid",
  "amount": 9900,
  "paid_at": 1705305022
}
```

---

## 🔍 사용 예시

```python
from app.services.payment_service import PaymentService

service = PaymentService(db)

webhook_data = {
    "imp_uid": "imp_123456789",
    "merchant_uid": "FMR_20240115_143022_ABC123",
    "status": "paid",
    "amount": 9900,
    "paid_at": 1705305022
}

result = service.process_webhook(webhook_data)

print(result)
# {
#     "success": True,
#     "job_id": "job-uuid",
#     "message": "Payment verified successfully"
# }
```

---

## ✅ 테스트 시나리오

### 1. 정상 처리

```python
def test_webhook_success():
    service = PaymentService(db)
    result = service.process_webhook({
        "imp_uid": "imp_123",
        "merchant_uid": "FMR_...",
        "status": "paid",
        "amount": 9900,
        "paid_at": 1705305022
    })

    assert result["success"] == True
```

### 2. Payment 없음

```python
def test_webhook_payment_not_found():
    service = PaymentService(db)

    with pytest.raises(AppException) as exc:
        service.process_webhook({
            "merchant_uid": "INVALID"
        })

    assert exc.value.error_code == "E-PAY-003"
```

### 3. 금액 불일치

```python
def test_webhook_amount_mismatch():
    service = PaymentService(db)

    with pytest.raises(AppException) as exc:
        service.process_webhook({
            "merchant_uid": "FMR_...",
            "amount": 5000  # 잘못된 금액
        })

    assert exc.value.error_code == "E-PAY-004"
```

---

## 📝 체크리스트

- [ ] process_webhook() 메서드 구현
- [ ] \_verify_payment_with_portone() 구현
- [ ] 금액 검증 로직
- [ ] Job 상태 업데이트
- [ ] 에러 처리
- [ ] 테스트 작성

---

## 🚀 다음 단계

Webhook 처리 완료 → **api.md** (API 엔드포인트)
