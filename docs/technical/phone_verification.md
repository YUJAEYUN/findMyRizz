# 전화번호 인증 스펙 (단순 매칭 방식)

## 1. 개요

- **인증 방식**: 단순 매칭 (OTP 없음)
- **목적**: 결과 조회 시 본인 확인
- **보안 수준**: 기본 (MVP용)

---

## 2. 전화번호 등록 (결제 후)

### 2.1 API 엔드포인트

```
PATCH /api/v1/jobs/{job_id}/phone
```

### 2.2 요청 예시

```json
{
  "user_phone_number": "010-1234-5678"
}
```

### 2.3 검증 규칙

```python
import re
import phonenumbers

def validate_phone_number(phone: str) -> bool:
    """
    전화번호 형식 검증
    - 한국 전화번호만 허용
    - 010, 011, 016, 017, 018, 019 허용
    """
    # 하이픈 제거
    phone_clean = phone.replace("-", "").replace(" ", "")
    
    # 정규식 검증
    pattern = r'^01[0-9]{1}[0-9]{7,8}$'
    if not re.match(pattern, phone_clean):
        return False
    
    # phonenumbers 라이브러리로 추가 검증
    try:
        parsed = phonenumbers.parse(phone, "KR")
        return phonenumbers.is_valid_number(parsed)
    except:
        return False
```

### 2.4 응답 예시

```json
{
  "success": true,
  "message": "전화번호가 등록되었습니다",
  "data": {
    "job_id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "pending_upload"
  }
}
```

---

## 3. 전화번호 인증 (결과 조회 시)

### 3.1 API 엔드포인트

```
POST /api/v1/results/{job_id}/verify
```

### 3.2 요청 예시

```json
{
  "phone_number": "010-1234-5678"
}
```

### 3.3 인증 로직

```python
# app/api/results.py

@router.post("/{job_id}/verify")
async def verify_phone_number(
    job_id: str,
    phone_number: str,
    request: Request,
    db: AsyncSession = Depends(get_db)
):
    # 1. Job 조회
    job = await db.get(Job, job_id)
    if not job:
        raise HTTPException(status_code=404, detail="Job을 찾을 수 없습니다")
    
    # 2. 만료 확인
    if job.expires_at < datetime.now(timezone.utc):
        raise HTTPException(status_code=410, detail="결과가 만료되었습니다")
    
    # 3. 전화번호 정규화
    phone_clean = phone_number.replace("-", "").replace(" ", "")
    job_phone_clean = job.user_phone_number.replace("-", "").replace(" ", "")
    
    # 4. 전화번호 매칭
    if phone_clean != job_phone_clean:
        # 실패 로그 저장
        await log_verification_attempt(
            db=db,
            job_id=job_id,
            phone_number=phone_number,
            ip_address=request.client.host,
            success=False
        )
        
        # Rate Limiting 체크
        await check_rate_limit(db, job_id, request.client.host)
        
        raise HTTPException(status_code=401, detail="전화번호가 일치하지 않습니다")
    
    # 5. 성공 로그 저장
    await log_verification_attempt(
        db=db,
        job_id=job_id,
        phone_number=phone_number,
        ip_address=request.client.host,
        success=True
    )
    
    # 6. JWT 토큰 발급
    access_token = create_access_token(
        data={"job_id": job_id, "phone": phone_clean},
        expires_delta=timedelta(hours=1)
    )
    
    return {
        "success": True,
        "data": {
            "verified": True,
            "access_token": access_token,
            "expires_in": 3600
        }
    }
```

### 3.4 응답 예시

**성공 (200 OK):**
```json
{
  "success": true,
  "data": {
    "verified": true,
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expires_in": 3600
  }
}
```

**실패 (401 Unauthorized):**
```json
{
  "success": false,
  "error": {
    "code": "E008",
    "message": "전화번호가 일치하지 않습니다"
  }
}
```

---

## 4. Rate Limiting

### 4.1 제한 규칙

- **IP 기반**: 동일 IP에서 3회 실패 시 1시간 차단
- **Job 기반**: 동일 Job에 대해 10회 실패 시 영구 차단

### 4.2 구현 예시

```python
async def check_rate_limit(
    db: AsyncSession,
    job_id: str,
    ip_address: str
):
    # 최근 1시간 내 실패 횟수 조회
    one_hour_ago = datetime.now(timezone.utc) - timedelta(hours=1)
    
    result = await db.execute(
        select(func.count(PhoneVerificationAttempt.id))
        .where(
            PhoneVerificationAttempt.job_id == job_id,
            PhoneVerificationAttempt.ip_address == ip_address,
            PhoneVerificationAttempt.success == False,
            PhoneVerificationAttempt.attempted_at >= one_hour_ago
        )
    )
    
    fail_count = result.scalar()
    
    if fail_count >= 3:
        raise HTTPException(
            status_code=429,
            detail="인증 시도 횟수를 초과했습니다. 1시간 후 다시 시도해주세요."
        )
```

---

## 5. JWT 토큰

### 5.1 토큰 생성

```python
from jose import jwt
from datetime import datetime, timedelta

def create_access_token(data: dict, expires_delta: timedelta):
    to_encode = data.copy()
    expire = datetime.utcnow() + expires_delta
    to_encode.update({"exp": expire})
    
    encoded_jwt = jwt.encode(
        to_encode,
        settings.JWT_SECRET_KEY,
        algorithm="HS256"
    )
    
    return encoded_jwt
```

### 5.2 토큰 검증

```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def verify_token(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> dict:
    try:
        payload = jwt.decode(
            credentials.credentials,
            settings.JWT_SECRET_KEY,
            algorithms=["HS256"]
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="토큰이 만료되었습니다")
    except jwt.JWTError:
        raise HTTPException(status_code=401, detail="유효하지 않은 토큰입니다")
```

---

## 6. 보안 고려사항

### 6.1 현재 구현 (MVP)

- ✅ 전화번호 단순 매칭
- ✅ Rate Limiting (IP + Job 기반)
- ✅ JWT 토큰 (1시간 유효)
- ✅ 실패 로그 저장

### 6.2 향후 개선 (V2)

- ⬜ SMS OTP 인증
- ⬜ 전화번호 해싱 저장
- ⬜ 2FA (Two-Factor Authentication)

---

## 7. 참고 문서

- [python-jose JWT](https://python-jose.readthedocs.io/)
- [phonenumbers 라이브러리](https://github.com/daviddrysdale/python-phonenumbers)

