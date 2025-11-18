# 전역 에러 처리

## 📋 목표

표준화된 에러 처리 및 응답 형식을 구현합니다.

---

## 🔧 구현

### 파일: `app/core/exceptions.py`

```python
"""
커스텀 예외 클래스
"""
from fastapi import HTTPException


class AppException(HTTPException):
    """
    애플리케이션 예외
    
    표준 에러 코드 형식: E-{CATEGORY}-{NUMBER}
    
    카테고리:
    - PAY: 결제
    - IMG: 이미지
    - AI: AI 생성
    - RPT: 리포트
    - SMS: SMS
    - AUTH: 인증
    - FILE: 파일
    - SYS: 시스템
    """
    
    def __init__(
        self,
        status_code: int,
        error_code: str,
        message: str,
        details: dict = None
    ):
        self.error_code = error_code
        self.message = message
        self.details = details or {}
        
        super().__init__(
            status_code=status_code,
            detail={
                "error_code": error_code,
                "message": message,
                "details": self.details
            }
        )
```

---

## 🔧 전역 Exception Handler

### 파일: `app/core/error_handlers.py`

```python
"""
전역 에러 핸들러
"""
from fastapi import Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
import logging

from app.core.exceptions import AppException

logger = logging.getLogger(__name__)


async def app_exception_handler(request: Request, exc: AppException):
    """
    AppException 핸들러
    """
    logger.error(
        f"[AppException] {exc.error_code}: {exc.message}",
        extra={
            "error_code": exc.error_code,
            "path": request.url.path,
            "method": request.method
        }
    )
    
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error_code": exc.error_code,
            "message": exc.message,
            "details": exc.details
        }
    )


async def validation_exception_handler(request: Request, exc: RequestValidationError):
    """
    Pydantic Validation 에러 핸들러
    """
    logger.warning(
        f"[ValidationError] {exc.errors()}",
        extra={
            "path": request.url.path,
            "method": request.method
        }
    )
    
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={
            "error_code": "E-SYS-001",
            "message": "Validation error",
            "details": exc.errors()
        }
    )


async def general_exception_handler(request: Request, exc: Exception):
    """
    일반 예외 핸들러
    """
    logger.error(
        f"[UnhandledException] {str(exc)}",
        exc_info=True,
        extra={
            "path": request.url.path,
            "method": request.method
        }
    )
    
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "error_code": "E-SYS-999",
            "message": "Internal server error",
            "details": {}
        }
    )
```

---

## 🔧 main.py에 등록

### 파일: `main.py`

```python
from fastapi import FastAPI
from fastapi.exceptions import RequestValidationError

from app.core.exceptions import AppException
from app.core.error_handlers import (
    app_exception_handler,
    validation_exception_handler,
    general_exception_handler
)

app = FastAPI()

# Exception Handlers 등록
app.add_exception_handler(AppException, app_exception_handler)
app.add_exception_handler(RequestValidationError, validation_exception_handler)
app.add_exception_handler(Exception, general_exception_handler)
```

---

## 🔍 에러 코드 체계

### 카테고리별 에러 코드

```python
# 결제 (PAY)
E-PAY-001: 전화번호 형식 오류
E-PAY-002: DB 저장 실패
E-PAY-003: Payment 없음
E-PAY-004: 금액 불일치
E-PAY-005: 검증 실패

# 이미지 (IMG)
E-IMG-001: 파일 크기 초과
E-IMG-002: 잘못된 형식
E-IMG-003: 해상도 오류
E-IMG-004: 업로드 실패
E-IMG-005: 검증 실패

# AI (AI)
E-AI-001: API 호출 실패
E-AI-002: Prediction 조회 실패

# 리포트 (RPT)
E-RPT-001: 분석 실패
E-RPT-002: JSON 파싱 실패
E-RPT-003: Job 없음
E-RPT-004: 원본 이미지 없음
E-RPT-005: 리포트 생성 실패

# SMS (SMS)
E-SMS-001: 발송 실패

# 인증 (AUTH)
E-AUTH-001: Job 없음
E-AUTH-002: IP 차단
E-AUTH-003: 최대 시도 초과
E-AUTH-004: 인증 실패

# 파일 (FILE)
E-FILE-001: S3 업로드 실패
E-FILE-002: Presigned URL 생성 실패

# 시스템 (SYS)
E-SYS-001: Validation 오류
E-SYS-999: 내부 서버 오류
```

---

## 🔍 에러 응답 형식

### 성공 응답
```json
{
  "success": true,
  "data": {...}
}
```

### 에러 응답
```json
{
  "error_code": "E-PAY-001",
  "message": "Phone number must start with 010",
  "details": {
    "field": "phone_number",
    "value": "011-1234-5678"
  }
}
```

---

## 🔍 사용 예시

```python
from app.core.exceptions import AppException

# 에러 발생
if not phone_number.startswith('010'):
    raise AppException(
        status_code=400,
        error_code="E-PAY-001",
        message="Phone number must start with 010",
        details={"field": "phone_number"}
    )
```

---

## ✅ 테스트

```python
def test_app_exception(client):
    response = client.post(
        "/api/v1/payments/initialize",
        json={"phone_number": "011-1234-5678"}
    )
    
    assert response.status_code == 400
    data = response.json()
    assert data["error_code"] == "E-PAY-001"
    assert "message" in data
```

---

## 📝 체크리스트

- [ ] app/core/exceptions.py 생성
- [ ] app/core/error_handlers.py 생성
- [ ] AppException 클래스 구현
- [ ] 전역 핸들러 등록
- [ ] 에러 코드 정의
- [ ] 테스트 작성

---

## 🚀 다음 단계

에러 처리 완료 → **logging.md**

