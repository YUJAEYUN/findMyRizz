# Payment 스키마

## 📋 목표

결제 관련 Pydantic 스키마를 정의합니다.

---

## 🔧 구현

### 파일: `app/schemas/payment.py`

```python
"""
결제 관련 Pydantic 스키마
"""
from pydantic import BaseModel, Field, validator
from typing import Optional
from datetime import datetime


class PaymentInitializeRequest(BaseModel):
    """결제 초기화 요청"""
    phone_number: str = Field(
        ...,
        description="사용자 전화번호",
        example="01012345678",
        min_length=10,
        max_length=13
    )
    
    @validator('phone_number')
    def validate_phone_number(cls, v: str) -> str:
        """
        전화번호 검증
        
        규칙:
        - 하이픈 제거
        - 010으로 시작
        - 11자리
        """
        # 하이픈, 공백 제거
        phone = v.replace('-', '').replace(' ', '')
        
        # 숫자만 포함
        if not phone.isdigit():
            raise ValueError("Phone number must contain only digits")
        
        # 010으로 시작
        if not phone.startswith('010'):
            raise ValueError("Phone number must start with 010")
        
        # 11자리
        if len(phone) != 11:
            raise ValueError("Phone number must be 11 digits")
        
        return phone


class PaymentInitializeResponse(BaseModel):
    """결제 초기화 응답"""
    job_id: str = Field(..., description="생성된 Job ID")
    merchant_uid: str = Field(..., description="가맹점 주문번호")
    amount: int = Field(..., description="결제 금액")
    payment_name: str = Field(..., description="결제 상품명")
    buyer_tel: str = Field(..., description="구매자 전화번호")


class PortOneWebhookRequest(BaseModel):
    """
    PortOne Webhook 요청
    
    PortOne에서 전송하는 데이터 형식
    """
    imp_uid: str = Field(..., description="PortOne 결제 고유번호")
    merchant_uid: str = Field(..., description="가맹점 주문번호")
    status: str = Field(..., description="결제 상태 (paid, failed, cancelled)")
    amount: Optional[int] = Field(None, description="결제 금액")
    paid_at: Optional[int] = Field(None, description="결제 시간 (Unix timestamp)")
    fail_reason: Optional[str] = Field(None, description="실패 사유")


class PaymentVerificationResponse(BaseModel):
    """결제 검증 응답"""
    success: bool = Field(..., description="검증 성공 여부")
    job_id: str = Field(..., description="Job ID")
    status: str = Field(..., description="결제 상태")
    message: str = Field(..., description="메시지")


class PaymentStatusResponse(BaseModel):
    """결제 상태 조회 응답"""
    job_id: str
    merchant_uid: str
    imp_uid: Optional[str]
    amount: int
    status: str
    paid_at: Optional[datetime]
    created_at: datetime
```

---

## 📝 스키마 설명

### PaymentInitializeRequest
- **용도:** 결제 초기화 API 요청
- **검증:** 전화번호 형식 자동 검증
- **변환:** 하이픈 자동 제거

### PaymentInitializeResponse
- **용도:** 결제 초기화 API 응답
- **포함:** 프론트엔드에서 PortOne SDK에 전달할 정보

### PortOneWebhookRequest
- **용도:** PortOne Webhook 데이터 파싱
- **필수:** imp_uid, merchant_uid, status
- **선택:** amount, paid_at, fail_reason

### PaymentVerificationResponse
- **용도:** 결제 검증 결과
- **포함:** 성공 여부, Job ID, 상태, 메시지

---

## 🔍 사용 예시

### 요청 검증

```python
from app.schemas.payment import PaymentInitializeRequest

# 자동 검증
request = PaymentInitializeRequest(phone_number="010-1234-5678")
# → phone_number = "01012345678" (하이픈 제거)

# 검증 실패
try:
    request = PaymentInitializeRequest(phone_number="011-1234-5678")
except ValueError as e:
    print(e)  # "Phone number must start with 010"
```

### 응답 생성

```python
from app.schemas.payment import PaymentInitializeResponse

response = PaymentInitializeResponse(
    job_id="job-uuid",
    merchant_uid="FMR_20240115_143022_ABC123",
    amount=9900,
    payment_name="Find My Rizz AI 분석",
    buyer_tel="01012345678"
)
```

---

## 📝 체크리스트

- [ ] app/schemas/payment.py 생성
- [ ] PaymentInitializeRequest 정의
- [ ] PaymentInitializeResponse 정의
- [ ] PortOneWebhookRequest 정의
- [ ] PaymentVerificationResponse 정의
- [ ] validator 구현
- [ ] 테스트 작성

---

## 🚀 다음 단계

스키마 완료 → **services/initialize.md** (결제 초기화 로직)

