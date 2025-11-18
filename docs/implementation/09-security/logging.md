# 구조화된 로깅

## 📋 목표

구조화된 로깅 시스템을 구현합니다.

---

## 🔧 구현

### 파일: `app/core/logging_config.py`

```python
"""
로깅 설정
"""
import logging
import sys
from datetime import datetime


def setup_logging():
    """
    로깅 설정
    
    - 콘솔 출력: INFO 레벨
    - 파일 출력: DEBUG 레벨
    - 포맷: JSON 형식
    """
    # 루트 로거
    logger = logging.getLogger()
    logger.setLevel(logging.DEBUG)
    
    # 기존 핸들러 제거
    logger.handlers.clear()
    
    # 포맷터
    formatter = logging.Formatter(
        fmt='%(asctime)s | %(levelname)-8s | %(name)s | %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S'
    )
    
    # 콘솔 핸들러
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setLevel(logging.INFO)
    console_handler.setFormatter(formatter)
    logger.addHandler(console_handler)
    
    # 파일 핸들러
    file_handler = logging.FileHandler('logs/app.log')
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(formatter)
    logger.addHandler(file_handler)
    
    # 에러 파일 핸들러
    error_handler = logging.FileHandler('logs/error.log')
    error_handler.setLevel(logging.ERROR)
    error_handler.setFormatter(formatter)
    logger.addHandler(error_handler)
    
    return logger
```

---

## 🔧 main.py에 적용

### 파일: `main.py`

```python
from fastapi import FastAPI
import logging
import os

from app.core.logging_config import setup_logging

# 로그 디렉토리 생성
os.makedirs('logs', exist_ok=True)

# 로깅 설정
setup_logging()
logger = logging.getLogger(__name__)

app = FastAPI()

@app.on_event("startup")
async def startup_event():
    logger.info("Application starting up")

@app.on_event("shutdown")
async def shutdown_event():
    logger.info("Application shutting down")
```

---

## 🔧 요청 로깅 미들웨어

### 파일: `app/middleware/logging_middleware.py`

```python
"""
요청 로깅 미들웨어
"""
from fastapi import Request
import logging
import time

logger = logging.getLogger(__name__)


async def log_requests(request: Request, call_next):
    """
    모든 요청 로깅
    """
    # 시작 시간
    start_time = time.time()
    
    # 요청 정보
    logger.info(
        f"Request started",
        extra={
            "method": request.method,
            "path": request.url.path,
            "client_ip": request.client.host
        }
    )
    
    # 요청 처리
    response = await call_next(request)
    
    # 처리 시간
    process_time = time.time() - start_time
    
    # 응답 로깅
    logger.info(
        f"Request completed",
        extra={
            "method": request.method,
            "path": request.url.path,
            "status_code": response.status_code,
            "process_time": f"{process_time:.3f}s"
        }
    )
    
    return response
```

### main.py에 등록

```python
from app.middleware.logging_middleware import log_requests

app.middleware("http")(log_requests)
```

---

## 🔍 로깅 사용 예시

### 서비스 레이어

```python
import logging

logger = logging.getLogger(__name__)

class PaymentService:
    def initialize_payment(self, phone_number: str):
        logger.info(
            f"[PaymentService] Initializing payment",
            extra={"phone_number": phone_number}
        )
        
        try:
            # 처리 로직
            logger.debug(f"[PaymentService] Creating job")
            job = self._create_job(phone_number)
            
            logger.info(
                f"[PaymentService] Payment initialized",
                extra={"job_id": job.id, "merchant_uid": payment.merchant_uid}
            )
            
            return payment
            
        except Exception as e:
            logger.error(
                f"[PaymentService] Payment initialization failed: {e}",
                exc_info=True,
                extra={"phone_number": phone_number}
            )
            raise
```

---

## 🔍 로그 레벨

```python
# DEBUG: 상세한 디버깅 정보
logger.debug("Detailed debug information")

# INFO: 일반 정보
logger.info("Payment initialized successfully")

# WARNING: 경고 (처리는 계속)
logger.warning("Retry attempt 3/5")

# ERROR: 에러 (처리 실패)
logger.error("Payment verification failed", exc_info=True)

# CRITICAL: 심각한 에러
logger.critical("Database connection lost")
```

---

## 🔍 로그 파일 구조

```
logs/
├── app.log       # 모든 로그 (DEBUG 이상)
└── error.log     # 에러 로그만 (ERROR 이상)
```

---

## 🔧 로그 로테이션

### 파일: `app/core/logging_config.py` (수정)

```python
from logging.handlers import RotatingFileHandler

def setup_logging():
    # ... 기존 코드 ...
    
    # 파일 핸들러 (로테이션)
    file_handler = RotatingFileHandler(
        'logs/app.log',
        maxBytes=10 * 1024 * 1024,  # 10MB
        backupCount=5
    )
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(formatter)
    logger.addHandler(file_handler)
```

---

## ✅ 테스트

```python
import logging

logger = logging.getLogger(__name__)

def test_logging():
    logger.debug("This is a debug message")
    logger.info("This is an info message")
    logger.warning("This is a warning message")
    logger.error("This is an error message")
    logger.critical("This is a critical message")
```

---

## 📝 체크리스트

- [ ] app/core/logging_config.py 생성
- [ ] setup_logging() 구현
- [ ] main.py에 적용
- [ ] 요청 로깅 미들웨어
- [ ] 로그 디렉토리 생성
- [ ] 테스트

---

## 🚀 다음 단계

로깅 완료 → **ci-cd.md**

