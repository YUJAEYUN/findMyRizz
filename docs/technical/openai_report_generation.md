# OpenAI GPT-4o 리포트 생성 스펙

## 1. 개요

- **모델**: GPT-4o (Vision 지원)
- **목적**: 원본 이미지와 AI 생성 이미지 3장을 비교 분석하여 맞춤형 리포트 생성
- **출력 형식**: JSON

---

## 2. 리포트 구조

### 2.1 최종 출력 형식

```json
{
  "title": "더 선명한 나를 만드는, 3가지 변화",
  "subtitle": "다이어트 5Kg + 피부 관리로 완성한 V라인",
  "content": "현재 당신의 얼굴은 약간의 부종과 지방층으로 인해\n본래의 날렵한 윤곽이 묻혀 있는 상태입니다.\n특히 턱선이 불분명하고 볼이 둥글어 보이는 인상을 줍니다.\n\n체계적인 다이어트를 통한 5kg 감량은 얼굴 전체의 지방을 줄여\n자연스러운 V라인을 드러냅니다.\n턱 아래 지방이 감소하면서 목-턱 경계가 선명해지고,\n광대뼈가 살짝 도드라지면서 입체감이 살아납니다.\n의학 연구에 따르면 5kg 체중 감량 시 얼굴 지방은 평균 23% 감소하며, 턱선 각도는 약 12-15도 더 날카로워집니다.\n\n여기에 기초적인 피부 관리를 더하면 완벽합니다.\n레이저 토닝으로 색소를 개선하고 기초 화장으로 피부 톤을 균일하게 하면, 건강한 광채가 더해져 전체적으로 생기 있고 젊어 보이는 인상으로 완성됩니다.",
  "recommendations": [
    {
      "category": "skincare",
      "codes": ["SC003", "SC007"]
    },
    {
      "category": "procedure",
      "codes": ["P001", "P005"]
    }
  ],
  "timeline": "8-12주",
  "estimated_cost_range": {
    "min": 300000,
    "max": 800000
  }
}
```

---

## 3. GPT-4o 프롬프트

### 3.1 시스템 프롬프트

```python
SYSTEM_PROMPT = """
당신은 외모 개선 전문 컨설턴트입니다. 
원본 이미지와 AI가 생성한 개선된 이미지 3장을 분석하여, 
사용자가 어떤 변화를 통해 더 나은 모습으로 발전할 수 있는지 
구체적이고 실현 가능한 조언을 제공해야 합니다.

## 리포트 작성 가이드라인

### 1. 제목 (title)
- 형식: "더 선명한 나를 만드는, N가지 변화"
- N은 2-4 사이의 숫자

### 2. 부제목 (subtitle)
- 핵심 개선 방법을 간결하게 요약
- 예시: "다이어트 5Kg + 피부 관리로 완성한 V라인"

### 3. 본문 (content)
3개의 문단으로 구성:

**첫 번째 문단: 현재 상태 분석**
- 원본 이미지에서 개선이 필요한 부분을 객관적으로 서술
- 부정적이지 않고 중립적인 톤 유지

**두 번째 문단: 주요 개선 방법 (다이어트/운동)**
- 체중 감량이 얼굴에 미치는 영향 설명
- 구체적인 수치와 의학적 근거 포함
- 예: "5kg 체중 감량 시 얼굴 지방은 평균 23% 감소"

**세 번째 문단: 보조 개선 방법 (피부 관리/시술)**
- 피부 관리나 간단한 시술로 완성도를 높이는 방법
- 자연스럽고 건강한 변화 강조

### 4. 추천 관리법 (recommendations)
- knowledge_base에서 적절한 항목 2-4개 선택
- skincare와 procedure 카테고리를 균형있게 포함

### 5. 예상 기간 (timeline)
- "N-M주" 형식
- 현실적인 기간 제시 (8-16주 범위)

### 6. 예상 비용 (estimated_cost_range)
- min: 최소 비용 (원 단위)
- max: 최대 비용 (원 단위)
- 추천한 관리법들의 총합 기준

## 출력 형식
JSON 형식으로 출력하며, 위의 구조를 정확히 따라야 합니다.
"""
```

### 3.2 사용자 프롬프트

```python
USER_PROMPT = f"""
다음 이미지들을 분석하여 리포트를 생성해주세요:

- 원본 이미지: {original_image_url}
- 개선 이미지 1: {generated_image_1_url}
- 개선 이미지 2: {generated_image_2_url}
- 개선 이미지 3: {generated_image_3_url}

사용 가능한 관리법 목록:
{knowledge_base_items}

위 가이드라인에 따라 JSON 형식으로 리포트를 생성해주세요.
"""
```

---

## 4. API 호출 예시

```python
# app/services/report_service.py

async def generate_report(
    original_image_url: str,
    generated_image_urls: List[str],
    knowledge_base_items: List[dict]
) -> dict:
    """
    GPT-4o를 사용하여 리포트 생성
    """
    client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)
    
    # Knowledge Base 항목을 문자열로 변환
    kb_text = "\n".join([
        f"- {item['code']}: {item['name']} ({item['category']})"
        for item in knowledge_base_items
    ])
    
    response = await client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": USER_PROMPT.format(
                        original_image_url=original_image_url,
                        generated_image_1_url=generated_image_urls[0],
                        generated_image_2_url=generated_image_urls[1],
                        generated_image_3_url=generated_image_urls[2],
                        knowledge_base_items=kb_text
                    )},
                    {"type": "image_url", "image_url": {"url": original_image_url}},
                    {"type": "image_url", "image_url": {"url": generated_image_urls[0]}},
                    {"type": "image_url", "image_url": {"url": generated_image_urls[1]}},
                    {"type": "image_url", "image_url": {"url": generated_image_urls[2]}}
                ]
            }
        ],
        response_format={"type": "json_object"},
        max_tokens=2000,
        temperature=0.7
    )
    
    return json.loads(response.choices[0].message.content)
```

---

## 5. 에러 처리

### 5.1 재시도 로직

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10)
)
async def generate_report_with_retry(*args, **kwargs):
    return await generate_report(*args, **kwargs)
```

### 5.2 Fallback 템플릿

```python
FALLBACK_REPORT = {
    "title": "더 선명한 나를 만드는, 3가지 변화",
    "subtitle": "체계적인 관리로 완성하는 나만의 스타일",
    "content": "AI 분석 결과를 바탕으로 맞춤형 관리법을 추천드립니다...",
    "recommendations": [],
    "timeline": "8-12주",
    "estimated_cost_range": {"min": 300000, "max": 800000}
}
```

---

## 6. 비용 최적화

- **이미지 크기**: 최대 512x512로 리사이즈 (Vision API 비용 절감)
- **캐싱**: 동일 이미지 조합에 대한 결과 캐싱 (선택사항)
- **토큰 제한**: max_tokens=2000으로 제한

---

## 7. 참고 문서

- [OpenAI Vision API](https://platform.openai.com/docs/guides/vision)
- [GPT-4o 모델 가이드](https://platform.openai.com/docs/models/gpt-4o)

