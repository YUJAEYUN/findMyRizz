# Result View 스키마

## 📋 목표

결과 조회 관련 Pydantic 스키마를 정의합니다.

---

## 🔧 구현

### 파일: `app/schemas/result.py`

```python
"""
Result View 스키마
"""
from pydantic import BaseModel, Field, validator
from typing import List
from datetime import datetime
import re


class PhoneVerificationRequest(BaseModel):
    """전화번호 인증 요청"""
    phone_number: str = Field(
        ...,
        description="전화번호 (010-XXXX-XXXX)",
        example="010-1234-5678"
    )
    
    @validator('phone_number')
    def validate_phone_number(cls, v):
        """전화번호 형식 검증"""
        # 하이픈 제거
        phone = v.replace('-', '')
        
        # 형식 검증
        if not re.match(r'^010\d{8}$', phone):
            raise ValueError('Phone number must be in format 010-XXXX-XXXX')
        
        return v


class PhoneVerificationResponse(BaseModel):
    """전화번호 인증 응답"""
    success: bool
    token: str | None = None
    message: str | None = None
    remaining_attempts: int | None = None


class JobFileSchema(BaseModel):
    """Job File 스키마"""
    id: str
    file_type: str
    s3_key: str
    presigned_url: str
    prediction_id: str | None
    seed: int | None
    
    class Config:
        from_attributes = True


class ResultResponse(BaseModel):
    """결과 조회 응답"""
    job_id: str
    status: str
    files: List[JobFileSchema]
    report: dict | None
    expires_at: datetime
    
    class Config:
        from_attributes = True
```

---

## 🔍 스키마 설명

### PhoneVerificationRequest

```python
{
    "phone_number": "010-1234-5678"
}
```

**검증:**
- 010으로 시작
- 총 11자리 (하이픈 제외)
- 형식: 010-XXXX-XXXX

---

### PhoneVerificationResponse (성공)

```python
{
    "success": true,
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "message": null,
    "remaining_attempts": null
}
```

---

### PhoneVerificationResponse (실패)

```python
{
    "success": false,
    "token": null,
    "message": "Phone number does not match",
    "remaining_attempts": 2
}
```

---

### ResultResponse

```python
{
    "job_id": "job-uuid",
    "status": "completed",
    "files": [
        {
            "id": "file-uuid-1",
            "file_type": "original",
            "s3_key": "jobs/job-uuid/original/file-uuid-1.jpg",
            "presigned_url": "https://s3.amazonaws.com/...",
            "prediction_id": null,
            "seed": null
        },
        {
            "id": "file-uuid-2",
            "file_type": "generated",
            "s3_key": "jobs/job-uuid/generated/file-uuid-2.jpg",
            "presigned_url": "https://s3.amazonaws.com/...",
            "prediction_id": "pred-123",
            "seed": 42
        }
    ],
    "report": {
        "id": "report-uuid",
        "analysis_result": {...},
        "knowledge_items": [...]
    },
    "expires_at": "2024-01-16T14:30:22.123456"
}
```

---

## ✅ 테스트

```python
def test_phone_verification_request_valid():
    """유효한 전화번호"""
    data = {"phone_number": "010-1234-5678"}
    request = PhoneVerificationRequest(**data)
    assert request.phone_number == "010-1234-5678"


def test_phone_verification_request_invalid():
    """잘못된 전화번호"""
    data = {"phone_number": "011-1234-5678"}
    
    with pytest.raises(ValueError, match="Phone number must be"):
        PhoneVerificationRequest(**data)


def test_phone_verification_response_success():
    """인증 성공 응답"""
    response = PhoneVerificationResponse(
        success=True,
        token="jwt-token"
    )
    assert response.success is True
    assert response.token == "jwt-token"


def test_phone_verification_response_failure():
    """인증 실패 응답"""
    response = PhoneVerificationResponse(
        success=False,
        message="Phone number does not match",
        remaining_attempts=2
    )
    assert response.success is False
    assert response.remaining_attempts == 2
```

---

## 📝 체크리스트

- [ ] app/schemas/result.py 생성
- [ ] PhoneVerificationRequest 스키마
- [ ] PhoneVerificationResponse 스키마
- [ ] JobFileSchema 스키마
- [ ] ResultResponse 스키마
- [ ] 전화번호 validator
- [ ] 테스트 작성

---

## 🚀 다음 단계

Result View 스키마 완료 → **result-fetcher.md**

