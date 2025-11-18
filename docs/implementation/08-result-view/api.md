# Result View API

## 📋 목표

결과 조회 API를 구현합니다.

---

## 🔧 구현

### 파일: `app/api/v1/results.py`

```python
"""
Result View API
"""
from fastapi import APIRouter, Depends, HTTPException, Request
from sqlalchemy.orm import Session

from app.core.database import get_db
from app.schemas.result import (
    PhoneVerificationRequest,
    PhoneVerificationResponse,
    ResultResponse
)
from app.services.verification_service import VerificationService
from app.services.result_fetcher import ResultFetcher
from app.core.auth import verify_token, extract_job_id_from_token

router = APIRouter(prefix="/results", tags=["results"])


@router.post("/{job_id}/verify", response_model=PhoneVerificationResponse)
async def verify_phone_number(
    job_id: str,
    request_data: PhoneVerificationRequest,
    request: Request,
    db: Session = Depends(get_db)
):
    """
    전화번호 인증

    Args:
        job_id: Job ID
        request_data: 전화번호

    Returns:
        인증 결과 (성공 시 JWT 토큰)

    Raises:
        404: Job 없음
        429: IP 차단 (1시간)
        403: 최대 시도 초과
    """
    # 클라이언트 IP
    ip_address = request.client.host

    # 인증 서비스
    verification_service = VerificationService(db)

    # 전화번호 인증
    result = verification_service.verify_phone_number(
        job_id=job_id,
        phone_number=request_data.phone_number,
        ip_address=ip_address
    )

    return PhoneVerificationResponse(**result)


@router.get("/{job_id}", response_model=ResultResponse)
async def get_result(
    job_id: str,
    token: str = Depends(verify_token),
    db: Session = Depends(get_db)
):
    """
    결과 조회

    Args:
        job_id: Job ID
        token: JWT 토큰 (Authorization 헤더)

    Returns:
        결과 데이터 (파일, 리포트)

    Raises:
        401: 토큰 없음/만료
        403: 권한 없음
        404: Job 없음
        410: Job 만료됨
    """
    # 토큰에서 job_id 추출
    token_job_id = extract_job_id_from_token(token)

    # job_id 일치 확인
    if token_job_id != job_id:
        raise HTTPException(
            status_code=403,
            detail="Forbidden"
        )

    # 결과 조회
    fetcher = ResultFetcher(db)
    result = fetcher.fetch_result(job_id)

    return ResultResponse(**result)
```

---

## 🔧 JWT 인증

### 파일: `app/core/auth.py`

```python
"""
JWT 인증
"""
from fastapi import Header, HTTPException
from jose import JWTError, jwt
from datetime import datetime
from zoneinfo import ZoneInfo

from app.config import settings


def get_kst_now():
    """KST 현재 시간 반환"""
    return datetime.now(ZoneInfo(settings.TIMEZONE))


def verify_token(authorization: str = Header(...)) -> str:
    """
    JWT 토큰 검증

    Args:
        authorization: Authorization 헤더 (Bearer {token})

    Returns:
        토큰 문자열

    Raises:
        401: 토큰 없음/만료/잘못됨
    """
    # Bearer 제거
    if not authorization.startswith("Bearer "):
        raise HTTPException(
            status_code=401,
            detail="Invalid authorization header"
        )

    token = authorization.replace("Bearer ", "")

    try:
        # 토큰 디코딩
        payload = jwt.decode(
            token,
            settings.JWT_SECRET_KEY,
            algorithms=["HS256"]
        )

        # 만료 확인
        exp = payload.get("exp")
        if exp:
            exp_time = datetime.fromtimestamp(exp, tz=ZoneInfo(settings.TIMEZONE))
            if exp_time < get_kst_now():
                raise HTTPException(
                    status_code=401,
                    detail="Token expired"
                )

        return token

    except JWTError:
        raise HTTPException(
            status_code=401,
            detail="Invalid token"
        )


def extract_job_id_from_token(token: str) -> str:
    """
    토큰에서 job_id 추출
    """
    payload = jwt.decode(
        token,
        settings.JWT_SECRET_KEY,
        algorithms=["HS256"]
    )

    return payload.get("job_id")
```

---

## 🔍 API 흐름

### 1. 전화번호 인증

```bash
POST /api/v1/results/{job_id}/verify
Content-Type: application/json

{
  "phone_number": "010-1234-5678"
}
```

**응답 (성공):**

```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "message": null,
  "remaining_attempts": null
}
```

**응답 (실패):**

```json
{
  "success": false,
  "token": null,
  "message": "Phone number does not match",
  "remaining_attempts": 2
}
```

---

### 2. 결과 조회

```bash
GET /api/v1/results/{job_id}
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**응답:**

```json
{
  "job_id": "job-uuid",
  "status": "completed",
  "files": [...],
  "report": {...},
  "expires_at": "2024-01-16T14:30:22.123456"
}
```

---

## ✅ 테스트

### curl 명령어

```bash
# 1. 전화번호 인증
curl -X POST "http://localhost:8000/api/v1/results/{job_id}/verify" \
  -H "Content-Type: application/json" \
  -d '{"phone_number": "010-1234-5678"}'

# 2. 결과 조회
curl -X GET "http://localhost:8000/api/v1/results/{job_id}" \
  -H "Authorization: Bearer {token}"
```

### pytest

```python
def test_verify_phone_number_success(client, db):
    # Job 생성
    job = Job(
        id=str(uuid.uuid4()),
        user_phone_number="010-1234-5678"
    )
    db.add(job)
    db.commit()

    # 인증 요청
    response = client.post(
        f"/api/v1/results/{job.id}/verify",
        json={"phone_number": "010-1234-5678"}
    )

    # 검증
    assert response.status_code == 200
    data = response.json()
    assert data["success"] is True
    assert data["token"] is not None


def test_get_result_success(client, db):
    # Job 및 토큰 생성
    job = Job(id=str(uuid.uuid4()), ...)
    db.add(job)
    db.commit()

    token = generate_token(job.id, job.user_phone_number)

    # 결과 조회
    response = client.get(
        f"/api/v1/results/{job.id}",
        headers={"Authorization": f"Bearer {token}"}
    )

    # 검증
    assert response.status_code == 200
    data = response.json()
    assert data["job_id"] == job.id
```

---

## 📝 체크리스트

- [ ] app/api/v1/results.py 생성
- [ ] app/core/auth.py 생성
- [ ] verify_phone_number() 엔드포인트
- [ ] get_result() 엔드포인트
- [ ] JWT 인증 구현
- [ ] 테스트 작성

---

## 🚀 완료!

Result View API 완료! 🎉
