# OpenAI Client (LangChain 통합)

## 📋 개요

**⚠️ 중요: 이 파일은 LangChain Workflow로 대체되었습니다.**

OpenAI GPT-4o Vision API는 이제 `langchain-workflow.md`의 `analyze_images_node`에서 사용됩니다.

---

## 🔄 변경사항

### Before (직접 OpenAI API 호출)

```python
from openai import OpenAI

client = OpenAI(api_key=settings.OPENAI_API_KEY)
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[...]
)
```

### After (LangChain Structured Output)

```python
from langchain_openai import ChatOpenAI
from pydantic import BaseModel

class AnalysisResult(BaseModel):
    overall_impression: str
    improvement_areas: List[str]
    specific_observations: dict
    confidence_score: float

llm = ChatOpenAI(model="gpt-4o")
structured_llm = llm.with_structured_output(AnalysisResult)
result = await structured_llm.ainvoke(messages)
```

---

## 🎯 장점

1. **타입 안정성** - Pydantic으로 응답 구조 보장
2. **에러 처리** - LangChain의 자동 재시도
3. **모니터링** - LangSmith 통합
4. **테스트 용이** - Mock 가능

---

## 📚 참고

- **새로운 구현**: `langchain-workflow.md` - analyze_images_node
- **아키텍처**: `docs/technical/langchain_architecture.md`

---

## 💡 레거시 코드 (참고용)

기존 OpenAI API 직접 호출 방식:

```python
# app/services/openai_service.py (DEPRECATED)

from openai import OpenAI
import logging

logger = logging.getLogger(__name__)


class OpenAIService:
    """
    OpenAI GPT-4o Vision 분석

    ⚠️ DEPRECATED: Use langchain_workflow.analyze_images_node instead
    """

    def __init__(self):
        """OpenAI 클라이언트 초기화"""
        self.client = OpenAI(api_key=settings.OPENAI_API_KEY)
        self.model = "gpt-4o"

    def analyze_image(self, image_url: str, prompt: str) -> str:
        """
        이미지 분석 (DEPRECATED)

        Use: langchain_workflow.run_report_workflow() instead
        """
        logger.warning("OpenAIService.analyze_image is deprecated")

        # Step 1: 메시지 구성
        messages = [
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": prompt},
                    {"type": "image_url", "image_url": {"url": image_url}}
                ]
            }
        ]

        # Step 2: API 호출
        response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                max_tokens=2000,
                temperature=0.7
            )

            # Step 3: 응답 추출
            analysis_result = response.choices[0].message.content

            logger.info(f"[OpenAIService] Analysis completed")
            return analysis_result

        except Exception as e:
            logger.error(f"[OpenAIService] Failed: {e}", exc_info=True)
            raise AppException(
                status_code=500,
                error_code="E-RPT-001",
                message="Failed to analyze image"
            )

    def analyze_with_structured_output(
        self,
        image_url: str
    ) -> Dict:
        """
        구조화된 분석 결과 반환

        Args:
            image_url: 이미지 URL

        Returns:
            Dict: {
                "skin_tone": str,
                "facial_features": List[str],
                "improvement_areas": List[str],
                "recommendations": List[str]
            }
        """
        # 구조화된 프롬프트
        prompt = """
        Analyze this person's facial features and provide a detailed assessment.

        Return your analysis in the following JSON format:
        {
            "skin_tone": "description of skin tone",
            "facial_features": ["feature1", "feature2", ...],
            "improvement_areas": ["area1", "area2", ...],
            "recommendations": ["recommendation1", "recommendation2", ...]
        }

        Focus on:
        1. Skin tone and texture
        2. Facial symmetry
        3. Eye shape and size
        4. Nose shape
        5. Lip shape and fullness
        6. Jawline and face shape

        Provide constructive and professional feedback.
        """

        # 분석 실행
        result_str = self.analyze_image(image_url, prompt)

        # JSON 파싱
        import json
        try:
            result_dict = json.loads(result_str)
            return result_dict
        except json.JSONDecodeError:
            logger.error(f"[OpenAIService] Failed to parse JSON: {result_str}")
            raise AppException(
                status_code=500,
                error_code="E-RPT-002",
                message="Failed to parse analysis result"
            )
```

---

## 🔍 프롬프트 예시

### 기본 분석 프롬프트

```python
prompt = """
Analyze this person's facial features and provide a detailed assessment.

Return your analysis in the following JSON format:
{
    "skin_tone": "description of skin tone",
    "facial_features": ["feature1", "feature2", ...],
    "improvement_areas": ["area1", "area2", ...],
    "recommendations": ["recommendation1", "recommendation2", ...]
}

Focus on:
1. Skin tone and texture
2. Facial symmetry
3. Eye shape and size
4. Nose shape
5. Lip shape and fullness
6. Jawline and face shape

Provide constructive and professional feedback.
"""
```

---

## 🔍 사용 예시

### 이미지 분석

```python
from app.services.openai_service import OpenAIService

service = OpenAIService()

result = service.analyze_with_structured_output(
    image_url="https://s3.../original.jpg"
)

print(result)
# {
#     "skin_tone": "밝은 편이며 균일한 톤",
#     "facial_features": ["큰 눈", "높은 코", "작은 입"],
#     "improvement_areas": ["피부 톤 개선", "윤곽 강조"],
#     "recommendations": ["비타민 C 세럼", "윤곽 메이크업"]
# }
```

---

## 📊 응답 형식

```json
{
  "skin_tone": "밝은 편이며 균일한 톤",
  "facial_features": ["큰 눈", "높은 코", "작은 입", "V라인 턱선"],
  "improvement_areas": ["피부 톤 개선", "윤곽 강조", "눈매 교정"],
  "recommendations": [
    "비타민 C 세럼 사용",
    "윤곽 메이크업 연습",
    "아이라인 기법 개선"
  ]
}
```

---

## ✅ 테스트 시나리오

### 1. 분석 성공

```python
def test_analyze_image_success(mocker):
    mock_response = mocker.Mock()
    mock_response.choices = [
        mocker.Mock(message=mocker.Mock(content='{"skin_tone": "밝음"}'))
    ]

    mocker.patch.object(
        OpenAI,
        'chat.completions.create',
        return_value=mock_response
    )

    service = OpenAIService()
    result = service.analyze_image(
        image_url="https://test.jpg",
        prompt="Analyze this"
    )

    assert "skin_tone" in result
```

---

## 📝 체크리스트

- [ ] app/services/openai_service.py 생성
- [ ] OpenAIService 클래스 구현
- [ ] analyze_image() 구현
- [ ] analyze_with_structured_output() 구현
- [ ] 프롬프트 최적화
- [ ] 테스트 작성

---

## 🚀 다음 단계

OpenAI 클라이언트 완료 → **knowledge-matcher.md**
