# SMS 발송 Celery Task

## 📋 목표

Celery Task로 SMS를 발송합니다.

---

## 🔧 구현

### 파일: `app/tasks/sms_notification.py`

```python
"""
SMS 발송 Celery Task
"""
import logging
from datetime import datetime
from zoneinfo import ZoneInfo

from app.core.celery_app import celery_app
from app.core.database import SessionLocal
from app.models.job import Job
from app.models.sms_log import SmsLog, SmsStatus
from app.services.sms_service import SMSService
from app.config import settings

logger = logging.getLogger(__name__)


def get_kst_now():
    """KST 현재 시간 반환"""
    return datetime.now(ZoneInfo(settings.TIMEZONE))


@celery_app.task(name="send_sms_notification")
def send_sms_notification_task(job_id: str):
    """
    SMS 발송 Celery Task
    
    처리:
    1. Job 조회
    2. SMS 발송 (재시도 포함)
    3. SmsLog 저장
    
    Args:
        job_id: Job ID
    """
    db = SessionLocal()
    
    try:
        logger.info(f"[SMSTask] Starting: job_id={job_id}")
        
        # Step 1: Job 조회
        job = db.query(Job).filter(Job.id == job_id).first()
        if not job:
            logger.error(f"[SMSTask] Job not found: {job_id}")
            return
        
        if not job.user_phone_number:
            logger.error(f"[SMSTask] No phone number: {job_id}")
            return
        
        logger.info(f"[SMSTask] Sending to {job.user_phone_number}")
        
        # Step 2: SMS 발송 (재시도 포함)
        sms_service = SMSService()
        
        try:
            result = sms_service.send_result_notification(
                to_number=job.user_phone_number,
                job_id=job_id
            )
            
            # Step 3: SmsLog 저장 (성공)
            sms_log = SmsLog(
                job_id=job_id,
                phone_number=job.user_phone_number,
                message_id=result.get("message_id"),
                status=SmsStatus.SENT,
                sent_at=get_kst_now()
            )
            db.add(sms_log)
            db.commit()
            
            logger.info(f"[SMSTask] SMS sent successfully: {result.get('message_id')}")
            
        except Exception as e:
            logger.error(f"[SMSTask] SMS failed after retries: {e}", exc_info=True)
            
            # Step 3: SmsLog 저장 (실패)
            sms_log = SmsLog(
                job_id=job_id,
                phone_number=job.user_phone_number,
                status=SmsStatus.FAILED,
                error_message=str(e),
                sent_at=get_kst_now()
            )
            db.add(sms_log)
            db.commit()
            
            # 에러 알림 (선택사항)
            # send_error_alert(f"SMS failed for job {job_id}: {e}")
        
    except Exception as e:
        logger.error(f"[SMSTask] Unexpected error: {e}", exc_info=True)
    
    finally:
        db.close()
```

---

## 🔍 사용 예시

### Celery Task 실행

```python
from app.tasks.sms_notification import send_sms_notification_task

# Report 생성 완료 후 호출
send_sms_notification_task.delay(job_id=job_id)
```

---

## 🔍 처리 흐름

```
1. Report 생성 완료
   ↓
2. Celery Task 큐에 추가
   ↓
3. send_sms_notification_task 실행
   ↓
4. Job 조회 및 전화번호 확인
   ↓
5. SMS 발송 (재시도 최대 3회)
   ↓
6. SmsLog 저장 (성공/실패)
```

---

## ✅ 테스트 시나리오

### 1. Task 실행 성공

```python
def test_send_sms_task_success(mocker):
    # Mock SMS Service
    mock_sms_result = {
        "success": True,
        "message_id": "G123",
        "error": None
    }
    
    mocker.patch(
        'app.services.sms_service.SMSService.send_result_notification',
        return_value=mock_sms_result
    )
    
    # Task 실행
    send_sms_notification_task(job_id="test-job")
    
    # 검증
    sms_log = db.query(SmsLog).filter(SmsLog.job_id == "test-job").first()
    assert sms_log.status == SmsStatus.SENT
    assert sms_log.message_id == "G123"
```

---

## 📝 체크리스트

- [ ] app/tasks/sms_notification.py 생성
- [ ] @celery_app.task 데코레이터 적용
- [ ] send_sms_notification_task() 함수 구현
- [ ] SMS 재시도 로직 통합
- [ ] SmsLog 저장
- [ ] KST 시간대 사용
- [ ] 에러 처리
- [ ] 테스트 작성

---

## 🚀 다음 단계

SMS Notification Task 완료 → **Result View**

