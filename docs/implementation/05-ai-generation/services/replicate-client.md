# Replicate API 클라이언트

## 📋 목표

Replicate AI 이미지 생성 API를 호출하여 AI 이미지를 생성합니다.

---

## 🔧 구현

### 파일: `app/services/replicate_service.py`

```python
"""
Replicate API 서비스
"""
import replicate
import random
import logging
from typing import Dict, List

from app.config import settings
from app.core.exceptions import AppException

logger = logging.getLogger(__name__)


class ReplicateService:
    """Replicate AI 이미지 생성"""

    def __init__(self):
        """Replicate 클라이언트 초기화"""
        self.client = replicate.Client(api_token=settings.REPLICATE_API_TOKEN)
        self.model_version = settings.REPLICATE_MODEL_VERSION  # 모델 버전

    def generate_image(
        self,
        image_url: str,
        seed: int,
        webhook_url: str
    ) -> str:
        """
        AI 이미지 생성 요청

        Args:
            image_url: 원본 이미지 URL (S3 Presigned URL)
            seed: 랜덤 시드
            webhook_url: 완료 시 호출할 Webhook URL

        Returns:
            str: prediction_id

        Raises:
            AppException(E-AI-001): API 호출 실패
        """
        try:
            logger.info(f"[ReplicateService] Generating: seed={seed}")

            # Step 1: 파라미터 준비
            input_params = {
                "image": image_url,
                "seed": seed,
                "num_inference_steps": 50,
                "guidance_scale": 7.5
            }

            # Step 2: API 호출
            prediction = self.client.predictions.create(
                version=self.model_version,
                input=input_params,
                webhook=webhook_url,
                webhook_events_filter=["completed"]
            )

            prediction_id = prediction.id
            logger.info(f"[ReplicateService] Created: {prediction_id}")

            return prediction_id

        except Exception as e:
            logger.error(f"[ReplicateService] Failed: {e}", exc_info=True)
            raise AppException(
                status_code=500,
                error_code="E-AI-001",
                message="Failed to create AI generation request"
            )

    def generate_multiple(
        self,
        image_url: str,
        count: int,
        webhook_url: str
    ) -> List[Dict]:
        """
        여러 이미지 생성 (다른 seed)

        Args:
            image_url: 원본 이미지 URL
            count: 생성할 이미지 수 (기본 3)
            webhook_url: Webhook URL

        Returns:
            List[Dict]: [{"prediction_id": str, "seed": int}, ...]
        """
        results = []

        for i in range(count):
            # 랜덤 시드 생성
            seed = random.randint(1, 1000000)

            # 이미지 생성
            prediction_id = self.generate_image(
                image_url=image_url,
                seed=seed,
                webhook_url=webhook_url
            )

            results.append({
                "prediction_id": prediction_id,
                "seed": seed,
                "index": i
            })

            logger.info(f"[ReplicateService] Generated {i+1}/{count}")

        return results

    def get_prediction(self, prediction_id: str) -> Dict:
        """
        Prediction 상태 조회

        Args:
            prediction_id: Prediction ID

        Returns:
            Dict: {
                "id": str,
                "status": str,  # "starting", "processing", "succeeded", "failed"
                "output": str or None,  # 생성된 이미지 URL
                "error": str or None
            }
        """
        try:
            prediction = self.client.predictions.get(prediction_id)

            return {
                "id": prediction.id,
                "status": prediction.status,
                "output": prediction.output,
                "error": prediction.error
            }

        except Exception as e:
            logger.error(f"[ReplicateService] Get failed: {e}")
            raise AppException(
                status_code=500,
                error_code="E-AI-002",
                message="Failed to get prediction status"
            )
```

---

## 🎨 AI 이미지 생성 파라미터

```python
{
    "image": "https://s3.../original.jpg",  # 원본 이미지 URL
    "seed": 123456,                         # 랜덤 시드 (1~1000000)
    "num_inference_steps": 50,              # 추론 단계 (높을수록 품질 향상)
    "guidance_scale": 7.5                   # 가이드 강도 (높을수록 원본 유사)
}
```

---

## 🔍 사용 예시

### 단일 이미지 생성

```python
from app.services.replicate_service import ReplicateService

service = ReplicateService()

prediction_id = service.generate_image(
    image_url="https://s3.../original.jpg",
    seed=123456,
    webhook_url="https://api.fmr.com/webhooks/replicate"
)

print(prediction_id)
# "pred_abc123xyz"
```

### 3장 생성

```python
results = service.generate_multiple(
    image_url="https://s3.../original.jpg",
    count=3,
    webhook_url="https://api.fmr.com/webhooks/replicate"
)

print(results)
# [
#     {"prediction_id": "pred_1", "seed": 123456, "index": 0},
#     {"prediction_id": "pred_2", "seed": 789012, "index": 1},
#     {"prediction_id": "pred_3", "seed": 345678, "index": 2}
# ]
```

---

## 🔍 Webhook 응답 형식

Replicate가 완료 시 보내는 데이터:

```json
{
  "id": "pred_abc123xyz",
  "status": "succeeded",
  "output": "https://replicate.delivery/pbxt/...jpg",
  "input": {
    "image": "https://s3.../original.jpg",
    "seed": 123456
  }
}
```

---

## ✅ 테스트 시나리오

### 1. 생성 요청 성공

```python
def test_generate_image_success(mocker):
    mock_prediction = mocker.Mock()
    mock_prediction.id = "pred_123"

    mocker.patch.object(
        replicate.Client,
        'predictions.create',
        return_value=mock_prediction
    )

    service = ReplicateService()
    prediction_id = service.generate_image(
        image_url="https://test.jpg",
        seed=123,
        webhook_url="https://webhook.com"
    )

    assert prediction_id == "pred_123"
```

### 2. 3장 생성

```python
def test_generate_multiple():
    service = ReplicateService()
    results = service.generate_multiple(
        image_url="https://test.jpg",
        count=3,
        webhook_url="https://webhook.com"
    )

    assert len(results) == 3
    assert all("prediction_id" in r for r in results)
```

---

## 📝 체크리스트

- [ ] app/services/replicate_service.py 생성
- [ ] ReplicateService 클래스 구현
- [ ] generate_image() 구현
- [ ] generate_multiple() 구현
- [ ] get_prediction() 구현
- [ ] 테스트 작성

---

## 🚀 다음 단계

Replicate 클라이언트 완료 → **webhook-handler.md**
