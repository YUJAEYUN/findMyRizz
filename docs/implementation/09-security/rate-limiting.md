# Rate Limiting

## 📋 목표

slowapi를 사용하여 API Rate Limiting을 구현합니다.

---

## 🔧 구현

### 파일: `app/core/rate_limit.py`

```python
"""
Rate Limiting 설정
"""
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi import Request
import logging

logger = logging.getLogger(__name__)

# Limiter 생성
limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["60/minute"]  # 기본: 분당 60회
)


def get_client_ip(request: Request) -> str:
    """
    클라이언트 IP 추출
    
    X-Forwarded-For 헤더 우선 사용
    """
    forwarded = request.headers.get("X-Forwarded-For")
    if forwarded:
        return forwarded.split(",")[0].strip()
    return request.client.host
```

---

## 🔧 main.py에 적용

### 파일: `main.py`

```python
from fastapi import FastAPI
from slowapi import _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded

from app.core.rate_limit import limiter

app = FastAPI()

# Rate Limiter 등록
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
```

---

## 🔧 API에 적용

### 파일: `app/api/v1/payment.py`

```python
from fastapi import APIRouter, Request
from app.core.rate_limit import limiter

router = APIRouter()

@router.post("/initialize")
@limiter.limit("10/minute")  # 분당 10회
async def initialize_payment(request: Request, ...):
    ...
```

---

## 🔍 Rate Limit 설정

### 전역 설정
```python
limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["60/minute"]
)
```

### 엔드포인트별 설정
```python
@limiter.limit("10/minute")  # 결제 초기화: 분당 10회
@limiter.limit("3/hour")     # 전화번호 인증: 시간당 3회
@limiter.limit("100/minute") # 결과 조회: 분당 100회
```

---

## ✅ 테스트

```python
def test_rate_limit(client):
    # 11번 요청
    for i in range(11):
        response = client.post("/api/v1/payments/initialize", ...)
        
        if i < 10:
            assert response.status_code == 200
        else:
            assert response.status_code == 429  # Too Many Requests
```

---

## 📝 체크리스트

- [ ] app/core/rate_limit.py 생성
- [ ] Limiter 설정
- [ ] main.py에 등록
- [ ] 각 API에 적용
- [ ] 테스트 작성

---

## 🚀 다음 단계

Rate Limiting 완료 → **auto-deletion.md**

