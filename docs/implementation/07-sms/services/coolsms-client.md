# CoolSMS API 클라이언트

## 📋 목표

CoolSMS API를 사용하여 SMS를 발송합니다.

---

## 🔧 구현

### 파일: `app/services/sms_service.py`

```python
"""
SMS 발송 서비스
"""
import requests
import logging
from datetime import datetime
from typing import Optional

from app.config import settings
from app.core.exceptions import AppException

logger = logging.getLogger(__name__)


class SMSService:
    """CoolSMS API 클라이언트"""
    
    def __init__(self):
        """CoolSMS 초기화"""
        self.api_key = settings.COOLSMS_API_KEY
        self.api_secret = settings.COOLSMS_API_SECRET
        self.from_number = settings.COOLSMS_FROM_NUMBER
        self.base_url = "https://api.coolsms.co.kr/messages/v4/send"
    
    def send_sms(
        self,
        to_number: str,
        message: str
    ) -> dict:
        """
        SMS 발송
        
        Args:
            to_number: 수신 전화번호 (01012345678)
            message: 메시지 내용
            
        Returns:
            dict: {
                "success": bool,
                "message_id": str or None,
                "error": str or None
            }
            
        Raises:
            AppException(E-SMS-001): SMS 발송 실패
        """
        try:
            logger.info(f"[SMSService] Sending to {to_number}")
            
            # Step 1: 요청 데이터 준비
            payload = {
                "message": {
                    "to": to_number,
                    "from": self.from_number,
                    "text": message
                }
            }
            
            # Step 2: 인증 헤더
            headers = {
                "Authorization": f"Bearer {self.api_key}",
                "Content-Type": "application/json"
            }
            
            # Step 3: API 호출
            response = requests.post(
                self.base_url,
                json=payload,
                headers=headers,
                timeout=10
            )
            
            # Step 4: 응답 처리
            if response.status_code == 200:
                result = response.json()
                message_id = result.get("groupId")
                
                logger.info(f"[SMSService] Sent: {message_id}")
                return {
                    "success": True,
                    "message_id": message_id,
                    "error": None
                }
            else:
                error_msg = response.text
                logger.error(f"[SMSService] Failed: {error_msg}")
                return {
                    "success": False,
                    "message_id": None,
                    "error": error_msg
                }
                
        except Exception as e:
            logger.error(f"[SMSService] Exception: {e}", exc_info=True)
            raise AppException(
                status_code=500,
                error_code="E-SMS-001",
                message="Failed to send SMS"
            )
    
    def send_result_notification(
        self,
        to_number: str,
        job_id: str
    ) -> dict:
        """
        결과 알림 SMS 발송
        
        Args:
            to_number: 수신 전화번호
            job_id: Job ID
            
        Returns:
            dict: 발송 결과
        """
        # 결과 URL 생성
        result_url = f"{settings.FRONTEND_URL}/results/{job_id}"
        
        # 메시지 작성
        message = f"""[Find My Rizz]
AI 분석 결과가 준비되었습니다!

결과 확인: {result_url}

※ 24시간 후 자동 삭제됩니다."""
        
        # SMS 발송
        return self.send_sms(to_number, message)
```

---

## 📱 SMS 메시지 템플릿

### 결과 알림

```
[Find My Rizz]
AI 분석 결과가 준비되었습니다!

결과 확인: https://fmr.com/results/{job_id}

※ 24시간 후 자동 삭제됩니다.
```

---

## 🔍 사용 예시

### 결과 알림 발송

```python
from app.services.sms_service import SMSService

sms_service = SMSService()

result = sms_service.send_result_notification(
    to_number="01012345678",
    job_id="job-uuid"
)

print(result)
# {
#     "success": True,
#     "message_id": "G4V20230115143022ABC123",
#     "error": None
# }
```

### 커스텀 메시지 발송

```python
result = sms_service.send_sms(
    to_number="01012345678",
    message="테스트 메시지입니다."
)
```

---

## 🔍 CoolSMS API 응답

### 성공

```json
{
  "groupId": "G4V20230115143022ABC123",
  "to": "01012345678",
  "from": "01000000000",
  "type": "SMS",
  "statusMessage": "정상",
  "country": "82",
  "messageId": "M4V20230115143022ABC123",
  "statusCode": "2000",
  "accountId": "12345678"
}
```

### 실패

```json
{
  "errorCode": "InvalidPhoneNumber",
  "errorMessage": "Invalid phone number format"
}
```

---

## ✅ 테스트 시나리오

### 1. 발송 성공

```python
def test_send_sms_success(mocker):
    mock_response = mocker.Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"groupId": "G123"}
    
    mocker.patch('requests.post', return_value=mock_response)
    
    service = SMSService()
    result = service.send_sms("01012345678", "Test")
    
    assert result["success"] == True
    assert result["message_id"] == "G123"
```

### 2. 발송 실패

```python
def test_send_sms_failure(mocker):
    mock_response = mocker.Mock()
    mock_response.status_code = 400
    mock_response.text = "Invalid number"
    
    mocker.patch('requests.post', return_value=mock_response)
    
    service = SMSService()
    result = service.send_sms("invalid", "Test")
    
    assert result["success"] == False
```

---

## 📝 체크리스트

- [ ] app/services/sms_service.py 생성
- [ ] SMSService 클래스 구현
- [ ] send_sms() 구현
- [ ] send_result_notification() 구현
- [ ] 메시지 템플릿 작성
- [ ] 테스트 작성

---

## 🚀 다음 단계

CoolSMS 클라이언트 완료 → **retry.md** (재시도 로직)

