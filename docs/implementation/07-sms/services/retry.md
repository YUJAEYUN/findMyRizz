# SMS 재시도 로직

## 📋 목표

SMS 발송 실패 시 재시도 로직을 구현합니다.

---

## 🔧 구현

### 파일: `app/services/sms_retry.py`

```python
"""
SMS 재시도 서비스
"""
import time
import logging
from typing import Callable

logger = logging.getLogger(__name__)


class SMSRetryService:
    """
    SMS 재시도 서비스

    Exponential Backoff 전략 사용
    """

    def __init__(
        self,
        max_retries: int = 3,
        base_delay: float = 1.0,
        max_delay: float = 60.0
    ):
        """
        Args:
            max_retries: 최대 재시도 횟수
            base_delay: 기본 대기 시간 (초)
            max_delay: 최대 대기 시간 (초)
        """
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay

    def execute_with_retry(
        self,
        func: Callable,
        *args,
        **kwargs
    ):
        """
        재시도 로직으로 함수 실행

        Args:
            func: 실행할 함수
            *args: 함수 인자
            **kwargs: 함수 키워드 인자

        Returns:
            함수 실행 결과

        Raises:
            마지막 시도의 예외
        """
        last_exception = None

        for attempt in range(self.max_retries + 1):
            try:
                logger.info(f"[SMSRetry] Attempt {attempt + 1}/{self.max_retries + 1}")

                # 함수 실행
                result = func(*args, **kwargs)

                logger.info(f"[SMSRetry] Success on attempt {attempt + 1}")
                return result

            except Exception as e:
                last_exception = e
                logger.warning(
                    f"[SMSRetry] Attempt {attempt + 1} failed: {e}"
                )

                # 마지막 시도면 예외 발생
                if attempt == self.max_retries:
                    logger.error(
                        f"[SMSRetry] All {self.max_retries + 1} attempts failed"
                    )
                    raise last_exception

                # Exponential Backoff 계산
                delay = self._calculate_delay(attempt)
                logger.info(f"[SMSRetry] Waiting {delay:.2f}s before retry")
                time.sleep(delay)

        # 여기 도달하면 안 됨
        raise last_exception

    def _calculate_delay(self, attempt: int) -> float:
        """
        Exponential Backoff 지연 시간 계산

        Args:
            attempt: 현재 시도 횟수 (0부터 시작)

        Returns:
            대기 시간 (초)
        """
        # 2^attempt * base_delay
        delay = self.base_delay * (2 ** attempt)

        # max_delay 초과 방지
        return min(delay, self.max_delay)
```

---

## 🔧 SMSService에 적용

### 파일: `app/services/sms_service.py` (수정)

```python
from app.services.sms_retry import SMSRetryService

class SMSService:
    def __init__(self):
        self.api_key = settings.COOLSMS_API_KEY
        self.api_secret = settings.COOLSMS_API_SECRET
        self.from_number = settings.COOLSMS_FROM_NUMBER
        self.retry_service = SMSRetryService(
            max_retries=3,
            base_delay=1.0,
            max_delay=30.0
        )

    def send_result_notification(self, to_number: str, job_id: str):
        """
        결과 알림 SMS 발송 (재시도 포함)
        """
        logger.info(f"[SMSService] Sending notification to {to_number}")

        # 재시도 로직으로 실행
        return self.retry_service.execute_with_retry(
            self._send_sms_internal,
            to_number,
            job_id
        )

    def _send_sms_internal(self, to_number: str, job_id: str):
        """
        실제 SMS 발송 (내부 메서드)
        """
        result_url = f"{settings.FRONTEND_URL}/results/{job_id}"
        message = f"""[Find My Rizz]
AI 분석 결과가 준비되었습니다!

결과 확인: {result_url}

※ 24시간 후 자동 삭제됩니다."""

        # CoolSMS API 호출
        response = self._call_coolsms_api(to_number, message)

        return response
```

---

## 🔍 재시도 전략

### Exponential Backoff

```
시도 1: 즉시
시도 2: 1초 후 (2^0 * 1.0)
시도 3: 2초 후 (2^1 * 1.0)
시도 4: 4초 후 (2^2 * 1.0)
```

### 최대 대기 시간 제한

```python
delay = min(2^attempt * base_delay, max_delay)

# 예시 (max_delay=30)
시도 1: 0초
시도 2: 1초
시도 3: 2초
시도 4: 4초
시도 5: 8초
시도 6: 16초
시도 7: 30초 (max_delay)
시도 8: 30초 (max_delay)
```

---

## 🔍 사용 예시

### Celery Task에서 사용

```python
from app.services.sms_service import SMSService

@celery_app.task(name="send_sms_notification")
def send_sms_notification_task(job_id: str, phone_number: str):
    """
    SMS 발송 Celery Task
    """
    sms_service = SMSService()

    try:
        # 재시도 로직 포함
        sms_service.send_result_notification(phone_number, job_id)

        logger.info(f"[Task] SMS sent successfully to {phone_number}")

    except Exception as e:
        logger.error(f"[Task] SMS failed after all retries: {e}")
        # 알림 전송 (Slack 등)
        send_error_alert(f"SMS failed for job {job_id}: {e}")
```

---

## ✅ 테스트

```python
def test_sms_retry_success():
    """재시도 성공 테스트"""
    retry_service = SMSRetryService(max_retries=3)

    # 2번 실패 후 성공하는 함수
    call_count = 0
    def flaky_function():
        nonlocal call_count
        call_count += 1
        if call_count < 3:
            raise Exception("Temporary failure")
        return "success"

    # 실행
    result = retry_service.execute_with_retry(flaky_function)

    # 검증
    assert result == "success"
    assert call_count == 3


def test_sms_retry_failure():
    """재시도 실패 테스트"""
    retry_service = SMSRetryService(max_retries=2)

    # 항상 실패하는 함수
    def always_fail():
        raise Exception("Permanent failure")

    # 실행 및 검증
    with pytest.raises(Exception, match="Permanent failure"):
        retry_service.execute_with_retry(always_fail)
```

---

## 📝 체크리스트

- [ ] app/services/sms_retry.py 생성
- [ ] SMSRetryService 클래스 구현
- [ ] execute_with_retry() 메서드
- [ ] Exponential Backoff 계산
- [ ] SMSService에 적용
- [ ] 테스트 작성

---

## 🚀 다음 단계

SMS Retry 완료 → **Result View schemas**
