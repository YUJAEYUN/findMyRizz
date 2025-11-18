# 전화번호 인증 서비스

## 📋 목표

결과 조회 시 전화번호 인증을 처리합니다 (3회 제한, IP 차단).

---

## 🔧 구현

### 파일: `app/services/verification_service.py`

```python
"""
전화번호 인증 서비스
"""
from sqlalchemy.orm import Session
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo
import logging
import jwt

from app.models.job import Job
from app.models.phone_verification import PhoneVerificationAttempt
from app.config import settings
from app.core.exceptions import AppException

logger = logging.getLogger(__name__)


def get_kst_now():
    """KST 현재 시간 반환"""
    return datetime.now(ZoneInfo(settings.TIMEZONE))


class VerificationService:
    """전화번호 인증"""

    MAX_ATTEMPTS = 3
    BLOCK_DURATION_HOURS = 1

    def __init__(self, db: Session):
        self.db = db

    def verify_phone_number(
        self,
        job_id: str,
        phone_number: str,
        ip_address: str
    ) -> dict:
        """
        전화번호 인증

        처리:
        1. Job 조회
        2. IP 차단 확인
        3. 전화번호 일치 확인
        4. 실패 시 시도 기록
        5. 성공 시 JWT 발급

        Args:
            job_id: Job ID
            phone_number: 입력한 전화번호
            ip_address: 요청 IP

        Returns:
            {
                "success": bool,
                "token": str or None,
                "message": str
            }

        Raises:
            AppException(E-AUTH-001): Job 없음
            AppException(E-AUTH-002): IP 차단
            AppException(E-AUTH-003): 최대 시도 초과
        """
        try:
            logger.info(f"[VerificationService] Verify: job_id={job_id}, ip={ip_address}")

            # Step 1: Job 조회
            job = self.db.query(Job).filter(Job.id == job_id).first()
            if not job:
                raise AppException(
                    status_code=404,
                    error_code="E-AUTH-001",
                    message="Job not found"
                )

            # Step 2: IP 차단 확인
            if self._is_ip_blocked(job_id, ip_address):
                raise AppException(
                    status_code=403,
                    error_code="E-AUTH-002",
                    message="IP blocked due to too many failed attempts"
                )

            # Step 3: 전화번호 일치 확인
            if job.user_phone_number == phone_number:
                # 성공
                logger.info(f"[VerificationService] Success: {job_id}")

                # JWT 토큰 발급
                token = self._generate_token(job_id, phone_number)

                return {
                    "success": True,
                    "token": token,
                    "message": "Verification successful"
                }
            else:
                # 실패
                logger.warning(f"[VerificationService] Failed: {job_id}")

                # 시도 기록
                self._record_failed_attempt(job_id, ip_address)

                # 남은 시도 횟수 확인
                remaining = self._get_remaining_attempts(job_id, ip_address)

                if remaining == 0:
                    raise AppException(
                        status_code=403,
                        error_code="E-AUTH-003",
                        message="Maximum attempts exceeded. IP blocked for 1 hour."
                    )

                return {
                    "success": False,
                    "token": None,
                    "message": f"Phone number mismatch. {remaining} attempts remaining."
                }

        except AppException:
            raise
        except Exception as e:
            logger.error(f"[VerificationService] Error: {e}", exc_info=True)
            raise AppException(
                status_code=500,
                error_code="E-AUTH-004",
                message="Verification failed"
            )

    def _is_ip_blocked(self, job_id: str, ip_address: str) -> bool:
        """
        IP 차단 여부 확인

        최근 1시간 내 3회 이상 실패 시 차단
        """
        one_hour_ago = get_kst_now() - timedelta(hours=self.BLOCK_DURATION_HOURS)

        failed_count = self.db.query(PhoneVerificationAttempt).filter(
            PhoneVerificationAttempt.job_id == job_id,
            PhoneVerificationAttempt.ip_address == ip_address,
            PhoneVerificationAttempt.success == False,
            PhoneVerificationAttempt.created_at >= one_hour_ago
        ).count()

        return failed_count >= self.MAX_ATTEMPTS

    def _record_failed_attempt(self, job_id: str, ip_address: str):
        """실패 시도 기록"""
        attempt = PhoneVerificationAttempt(
            id=str(uuid.uuid4()),
            job_id=job_id,
            ip_address=ip_address,
            success=False
        )
        self.db.add(attempt)
        self.db.commit()

    def _get_remaining_attempts(self, job_id: str, ip_address: str) -> int:
        """남은 시도 횟수"""
        one_hour_ago = get_kst_now() - timedelta(hours=self.BLOCK_DURATION_HOURS)

        failed_count = self.db.query(PhoneVerificationAttempt).filter(
            PhoneVerificationAttempt.job_id == job_id,
            PhoneVerificationAttempt.ip_address == ip_address,
            PhoneVerificationAttempt.success == False,
            PhoneVerificationAttempt.created_at >= one_hour_ago
        ).count()

        return max(0, self.MAX_ATTEMPTS - failed_count)

    def _generate_token(self, job_id: str, phone_number: str) -> str:
        """
        JWT 토큰 발급

        유효기간: 1시간
        """
        # JWT 만료 시간 (KST 기준 1시간 후)
        exp_time = get_kst_now() + timedelta(hours=1)

        payload = {
            "job_id": job_id,
            "phone_number": phone_number,
            "exp": exp_time.timestamp()  # Unix timestamp로 변환
        }

        token = jwt.encode(
            payload,
            settings.JWT_SECRET_KEY,
            algorithm="HS256"
        )

        return token
```

---

## 🔍 사용 예시

```python
from app.services.verification_service import VerificationService

service = VerificationService(db)

result = service.verify_phone_number(
    job_id="job-uuid",
    phone_number="01012345678",
    ip_address="192.168.1.1"
)

print(result)
# {
#     "success": True,
#     "token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
#     "message": "Verification successful"
# }
```

---

## ✅ 테스트 시나리오

### 1. 인증 성공

```python
def test_verify_success():
    service = VerificationService(db)
    result = service.verify_phone_number(
        job_id=job_id,
        phone_number="01012345678",
        ip_address="192.168.1.1"
    )

    assert result["success"] == True
    assert result["token"] is not None
```

### 2. 전화번호 불일치

```python
def test_verify_mismatch():
    service = VerificationService(db)
    result = service.verify_phone_number(
        job_id=job_id,
        phone_number="01099999999",
        ip_address="192.168.1.1"
    )

    assert result["success"] == False
    assert "attempts remaining" in result["message"]
```

### 3. IP 차단

```python
def test_verify_ip_blocked():
    service = VerificationService(db)

    # 3회 실패
    for _ in range(3):
        service.verify_phone_number(
            job_id=job_id,
            phone_number="wrong",
            ip_address="192.168.1.1"
        )

    # 4번째 시도
    with pytest.raises(AppException) as exc:
        service.verify_phone_number(
            job_id=job_id,
            phone_number="wrong",
            ip_address="192.168.1.1"
        )

    assert exc.value.error_code == "E-AUTH-003"
```

---

## 📝 체크리스트

- [ ] app/services/verification_service.py 생성
- [ ] VerificationService 클래스 구현
- [ ] verify_phone_number() 구현
- [ ] IP 차단 로직
- [ ] JWT 토큰 발급
- [ ] 테스트 작성

---

## 🚀 다음 단계

전화번호 인증 완료 → **result-fetcher.md**
