# 결제 초기화 API - 완전한 슈도코드 명세

## 📋 개요

사용자가 결제를 시작할 때 호출되는 API의 **라인 바이 라인 구현 명세**

**중요 변경사항:**

- PortOne V2 사용 (V1 아님)
- 전화번호는 결제 성공 후 별도 API(`PATCH /api/v1/jobs/{job_id}/phone`)로 등록
- 참고 문서: `docs/technical/portone_v2_integration.md`

---

## 🎯 API 스펙

### 엔드포인트

```
POST /api/v1/payments/initialize
Content-Type: application/json
```

### 요청 바디

```json
{
  "amount": 9900
}
```

### 성공 응답 (200)

```json
{
  "success": true,
  "data": {
    "job_id": "550e8400-e29b-41d4-a716-446655440000",
    "merchant_uid": "FMR_20240115_143022_ABC123",
    "amount": 9900,
    "payment_name": "Find My Rizz AI 분석"
  }
}
```

### 에러 응답

| HTTP 코드 | 에러 코드        | 메시지                       | 발생 조건            |
| --------- | ---------------- | ---------------------------- | -------------------- |
| 422       | validation_error | Amount must be 9900          | 금액이 9900원이 아님 |
| 500       | E-PAY-002        | Failed to initialize payment | DB 오류              |

---

## 📁 파일 구조

```
app/
├── api/v1/payments.py          # ← 이 파일에 API 엔드포인트 구현
├── schemas/payment.py          # ← Pydantic 스키마 정의
├── services/payment_service.py # ← 비즈니스 로직 구현
├── models/job.py               # ← Job SQLAlchemy 모델
├── models/payment.py           # ← Payment SQLAlchemy 모델
└── core/
    ├── database.py             # ← get_db() 의존성
    └── exceptions.py           # ← AppException 정의
```

---

## 🔧 구현 1: Pydantic 스키마

### 파일: `app/schemas/payment.py`

```python
"""
결제 관련 Pydantic 스키마
"""
from pydantic import BaseModel, Field, validator
import re


class PaymentInitializeRequest(BaseModel):
    """
    결제 초기화 요청 스키마

    FastAPI가 자동으로 요청 바디를 검증함
    """
    amount: int = Field(
        ...,  # 필수 필드
        description="결제 금액 (고정: 9900원)",
        example=9900
    )

    @validator('amount')
    def validate_amount(cls, v: int) -> int:
        """
        금액 검증 로직

        검증 규칙:
        1. 정확히 9900원인지 확인

        Args:
            v (int): 입력된 금액

        Returns:
            int: 검증된 금액

        Raises:
            ValueError: 검증 실패 시
        """
        if v != 9900:
            raise ValueError("Amount must be 9900")

        return v


class PaymentInitializeResponse(BaseModel):
    """결제 초기화 응답 데이터"""
    job_id: str = Field(..., description="생성된 Job ID (UUID)")
    merchant_uid: str = Field(..., description="가맹점 주문번호")
    amount: int = Field(..., description="결제 금액 (원)")
    payment_name: str = Field(..., description="결제 상품명")


class PaymentInitializeResponseWrapper(BaseModel):
    """API 응답 래퍼 (표준 응답 형식)"""
    success: bool = True
    data: PaymentInitializeResponse
```

---

## 🔧 구현 2: 비즈니스 로직 서비스

### 파일: `app/services/payment_service.py`

```python
"""
결제 비즈니스 로직
"""
from sqlalchemy.orm import Session
from datetime import datetime, timedelta
import uuid
import logging
import random
import string

from app.models.job import Job, JobStatus
from app.models.payment import Payment, PaymentStatus
from app.config import settings
from app.core.exceptions import AppException

logger = logging.getLogger(__name__)


class PaymentService:
    """
    결제 서비스 클래스

    책임:
    - 결제 초기화 (Job + Payment 생성)
    - merchant_uid 생성
    """

    def __init__(self, db: Session):
        """
        Args:
            db: SQLAlchemy 세션 (의존성 주입)
        """
        self.db = db

    def initialize_payment(self, amount: int) -> dict:
        """
        결제 초기화 메인 로직

        처리 흐름:
        1. merchant_uid 생성 (FMR_YYYYMMDD_HHMMSS_RANDOM6)
        2. Job 레코드 생성
           - id: UUID
           - user_phone_number: NULL (결제 성공 후 입력)
           - status: pending_payment
           - expires_at: 현재시간 + 24시간
        3. Payment 레코드 생성
           - id: UUID
           - job_id: 위에서 생성한 Job ID
           - merchant_uid: 생성한 merchant_uid
           - amount: 9900 (설정값)
           - status: pending
        4. DB 커밋
        5. 응답 데이터 반환

        Args:
            phone_number (str): 검증된 전화번호 (11자리, 010으로 시작)

        Returns:
            dict: {
                "job_id": str,
                "merchant_uid": str,
                "amount": int,
                "payment_name": str,
                "buyer_tel": str
            }

        Raises:
            AppException: DB 오류 발생 시 (E-PAY-002)
        """
        try:
            # ===== Step 1: merchant_uid 생성 =====
            merchant_uid = self._generate_merchant_uid()
            logger.info(f"[PaymentService] Generated merchant_uid: {merchant_uid}")

            # ===== Step 2: Job 생성 =====
            job_id = str(uuid.uuid4())
            current_time = datetime.utcnow()
            expires_at = current_time + timedelta(hours=settings.JOB_EXPIRATION_HOURS)

            job = Job(
                id=job_id,
                user_phone_number=phone_number,
                status=JobStatus.PENDING_PAYMENT,  # Enum 값
                expires_at=expires_at,
                created_at=current_time,
                updated_at=current_time
            )
            self.db.add(job)
            logger.info(f"[PaymentService] Created Job: id={job_id}, phone={phone_number}")

            # ===== Step 3: Payment 생성 =====
            payment_id = str(uuid.uuid4())
            payment = Payment(
                id=payment_id,
                job_id=job_id,
                merchant_uid=merchant_uid,
                amount=settings.PAYMENT_AMOUNT,  # 9900
                status=PaymentStatus.PENDING,  # Enum 값
                created_at=current_time,
                updated_at=current_time
            )
            self.db.add(payment)
            logger.info(f"[PaymentService] Created Payment: id={payment_id}, merchant_uid={merchant_uid}, amount={settings.PAYMENT_AMOUNT}")

            # ===== Step 4: DB 커밋 =====
            self.db.commit()
            logger.info(f"[PaymentService] Payment initialized successfully: job_id={job_id}")

            # ===== Step 5: 응답 데이터 반환 =====
            response_data = {
                "job_id": job_id,
                "merchant_uid": merchant_uid,
                "amount": settings.PAYMENT_AMOUNT,
                "payment_name": "Find My Rizz AI 분석",
                "buyer_tel": phone_number
            }

            return response_data

        except Exception as e:
            # DB 롤백
            self.db.rollback()
            logger.error(f"[PaymentService] Failed to initialize payment: {e}", exc_info=True)

            # 커스텀 예외 발생
            raise AppException(
                status_code=500,
                error_code="E-PAY-002",
                message="Failed to initialize payment"
            )

    def _generate_merchant_uid(self) -> str:
        """
        merchant_uid 생성 (내부 메서드)

        형식: FMR_YYYYMMDD_HHMMSS_RANDOM6
        예시: FMR_20240115_143022_ABC123

        구성:
        - FMR: 서비스 prefix
        - YYYYMMDD: 날짜 (UTC)
        - HHMMSS: 시간 (UTC)
        - RANDOM6: 랜덤 6자리 (대문자 + 숫자)

        Returns:
            str: 생성된 merchant_uid
        """
        # 현재 시간 (UTC)
        now = datetime.utcnow()

        # 날짜 부분 (YYYYMMDD)
        date_part = now.strftime("%Y%m%d")

        # 시간 부분 (HHMMSS)
        time_part = now.strftime("%H%M%S")

        # 랜덤 6자리 (대문자 A-Z + 숫자 0-9)
        random_chars = string.ascii_uppercase + string.digits
        random_part = ''.join(random.choices(random_chars, k=6))

        # 조합
        merchant_uid = f"FMR_{date_part}_{time_part}_{random_part}"

        return merchant_uid
```

---

## 🔧 구현 3: API 라우터

### 파일: `app/api/v1/payments.py`

```python
"""
결제 API 라우터
"""
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
import logging

from app.core.database import get_db
from app.schemas.payment import (
    PaymentInitializeRequest,
    PaymentInitializeResponse,
    PaymentInitializeResponseWrapper
)
from app.services.payment_service import PaymentService
from app.core.exceptions import AppException

# 라우터 생성
router = APIRouter(prefix="/payments", tags=["payments"])

# 로거
logger = logging.getLogger(__name__)


@router.post(
    "/initialize",
    response_model=PaymentInitializeResponseWrapper,
    status_code=status.HTTP_200_OK,
    summary="결제 초기화",
    description="""
    사용자 전화번호를 받아 결제를 초기화합니다.

    처리 내용:
    1. 전화번호 검증 (Pydantic 자동)
    2. Job 생성 (status=pending_payment)
    3. Payment 생성 (status=pending)
    4. merchant_uid 반환 (프론트엔드에서 PortOne SDK에 전달)
    """,
    responses={
        200: {
            "description": "결제 초기화 성공",
            "content": {
                "application/json": {
                    "example": {
                        "success": True,
                        "data": {
                            "job_id": "550e8400-e29b-41d4-a716-446655440000",
                            "merchant_uid": "FMR_20240115_143022_ABC123",
                            "amount": 9900,
                            "payment_name": "Find My Rizz AI 분석",
                            "buyer_tel": "01012345678"
                        }
                    }
                }
            }
        },
        422: {
            "description": "전화번호 검증 실패",
            "content": {
                "application/json": {
                    "example": {
                        "detail": [
                            {
                                "loc": ["body", "phone_number"],
                                "msg": "Phone number must start with 010",
                                "type": "value_error"
                            }
                        ]
                    }
                }
            }
        },
        500: {
            "description": "서버 오류",
            "content": {
                "application/json": {
                    "example": {
                        "success": False,
                        "error": {
                            "code": "E-PAY-002",
                            "message": "Failed to initialize payment"
                        }
                    }
                }
            }
        }
    }
)
async def initialize_payment(
    request: PaymentInitializeRequest,
    db: Session = Depends(get_db)
) -> PaymentInitializeResponseWrapper:
    """
    결제 초기화 API 엔드포인트

    처리 흐름:
    1. FastAPI가 요청 바디를 PaymentInitializeRequest로 파싱
    2. Pydantic validator가 전화번호 검증 (자동)
    3. PaymentService.initialize_payment() 호출
    4. 응답 반환

    Args:
        request (PaymentInitializeRequest): 요청 바디
            - phone_number: 전화번호 (Pydantic이 검증)
        db (Session): DB 세션 (FastAPI 의존성 주입)

    Returns:
        PaymentInitializeResponseWrapper: 결제 초기화 결과

    Raises:
        HTTPException 422: 전화번호 검증 실패 (Pydantic이 자동 처리)
        HTTPException 500: 서버 오류
    """
    # 로깅: 요청 시작
    logger.info(f"[API] Payment initialization requested: phone={request.phone_number}")

    try:
        # ===== Step 1: 서비스 인스턴스 생성 =====
        service = PaymentService(db)

        # ===== Step 2: 비즈니스 로직 호출 =====
        result = service.initialize_payment(request.phone_number)

        # ===== Step 3: 응답 생성 =====
        response_data = PaymentInitializeResponse(**result)
        response = PaymentInitializeResponseWrapper(
            success=True,
            data=response_data
        )

        # 로깅: 성공
        logger.info(f"[API] Payment initialized successfully: job_id={result['job_id']}, merchant_uid={result['merchant_uid']}")

        return response

    except AppException as e:
        # 커스텀 예외는 전역 핸들러가 처리
        logger.error(f"[API] AppException occurred: {e.error_code} - {e.message}")
        raise

    except Exception as e:
        # 예상치 못한 예외
        logger.error(f"[API] Unexpected error: {e}", exc_info=True)
        raise HTTPException(
            status_code=500,
            detail={
                "success": False,
                "error": {
                    "code": "E-SYS-999",
                    "message": "Internal server error"
                }
            }
        )
```

---

## ✅ 완전한 테스트 시나리오

### 테스트 1: 정상 케이스

```bash
curl -X POST http://localhost:8000/api/v1/payments/initialize \
  -H "Content-Type: application/json" \
  -d '{"phone_number": "01012345678"}'
```

**예상 응답 (200):**

```json
{
  "success": true,
  "data": {
    "job_id": "550e8400-e29b-41d4-a716-446655440000",
    "merchant_uid": "FMR_20240115_143022_ABC123",
    "amount": 9900,
    "payment_name": "Find My Rizz AI 분석",
    "buyer_tel": "01012345678"
  }
}
```

**DB 확인:**

```sql
-- Job 테이블
SELECT * FROM jobs WHERE id = '550e8400-e29b-41d4-a716-446655440000';
-- 결과: status='pending_payment', user_phone_number='01012345678'

-- Payment 테이블
SELECT * FROM payments WHERE merchant_uid = 'FMR_20240115_143022_ABC123';
-- 결과: amount=9900, status='pending'
```

### 테스트 2: 하이픈 포함 전화번호 (정상 처리)

```bash
curl -X POST http://localhost:8000/api/v1/payments/initialize \
  -H "Content-Type: application/json" \
  -d '{"phone_number": "010-1234-5678"}'
```

**예상 응답 (200):** 하이픈이 제거되고 정상 처리됨

### 테스트 3: 잘못된 전화번호 - 011로 시작

```bash
curl -X POST http://localhost:8000/api/v1/payments/initialize \
  -H "Content-Type: application/json" \
  -d '{"phone_number": "01112345678"}'
```

**예상 응답 (422):**

```json
{
  "detail": [
    {
      "loc": ["body", "phone_number"],
      "msg": "Phone number must start with 010",
      "type": "value_error"
    }
  ]
}
```

### 테스트 4: 전화번호 길이 오류

```bash
curl -X POST http://localhost:8000/api/v1/payments/initialize \
  -H "Content-Type: application/json" \
  -d '{"phone_number": "0101234567"}'
```

**예상 응답 (422):**

```json
{
  "detail": [
    {
      "loc": ["body", "phone_number"],
      "msg": "Phone number must be 11 digits",
      "type": "value_error"
    }
  ]
}
```

---

## 🔍 구현 체크리스트

### 코드 작성

- [ ] `app/schemas/payment.py` 작성
  - [ ] PaymentInitializeRequest 클래스
  - [ ] validate_phone_number() validator
  - [ ] PaymentInitializeResponse 클래스
  - [ ] PaymentInitializeResponseWrapper 클래스
- [ ] `app/services/payment_service.py` 작성
  - [ ] PaymentService 클래스
  - [ ] initialize_payment() 메서드
  - [ ] \_generate_merchant_uid() 메서드
- [ ] `app/api/v1/payments.py` 작성
  - [ ] router 생성
  - [ ] initialize_payment() 엔드포인트

### 의존성 확인

- [ ] Job 모델 정의 완료 (`app/models/job.py`)
- [ ] Payment 모델 정의 완료 (`app/models/payment.py`)
- [ ] JobStatus Enum 정의
- [ ] PaymentStatus Enum 정의
- [ ] get_db() 의존성 구현 (`app/core/database.py`)
- [ ] AppException 정의 (`app/core/exceptions.py`)
- [ ] settings 설정 완료 (`app/config.py`)

### 테스트

- [ ] 단위 테스트: `test_payment_service.py`
  - [ ] test_initialize_payment_success()
  - [ ] test_generate_merchant_uid_format()
  - [ ] test_initialize_payment_db_error()
- [ ] 통합 테스트: `test_payment_api.py`
  - [ ] test_initialize_payment_api_success()
  - [ ] test_initialize_payment_invalid_phone()
  - [ ] test_initialize_payment_phone_with_hyphens()
- [ ] 수동 테스트: Postman/curl
  - [ ] 정상 케이스
  - [ ] 모든 에러 케이스

### 배포 전 확인

- [ ] 로깅 확인 (모든 단계에서 로그 출력)
- [ ] DB 트랜잭션 확인 (커밋/롤백)
- [ ] 에러 핸들링 확인
- [ ] API 문서 확인 (/docs)
