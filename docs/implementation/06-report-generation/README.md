# Phase 6: Report Generation (LangChain/LangGraph)

## 📋 개요

OpenAI GPT-4o 기반 AI 리포트 생성 시스템

**중요:** LangChain/LangGraph는 MVP에서 제외되었습니다. 단순 OpenAI API 사용으로 변경되었습니다.
**참고 문서:** `docs/technical/openai_report_generation.md`

---

## 🎯 목표 (MVP 단순화)

1. **OpenAI GPT-4o Vision** - 이미지 분석 및 리포트 생성
2. **JSON 출력** - 구조화된 리포트 형식
3. **Knowledge Base 매칭** - 추천 관리법 코드 반환
4. **에러 처리** - 재시도 로직 및 Fallback

---

## 🔄 주요 변경사항

### Knowledge Base 스키마 업데이트

**Before (통합 테이블):**

```python
# 단일 테이블
kb_items (item_id, item_type, category_id, ...)
```

**After (타입별 분리):**

```python
# Polymorphic 구조
kb_procedures (26 rows, 28 columns, 100% RDBMS)
kb_self_care_items (5 rows, 38 columns, 92% RDBMS + 8% JSONB)

# Polymorphic 검색
retriever.search(query="피부", item_type="PROCEDURE")  # 시술만
retriever.search(query="피부", item_type="SELF_CARE")  # 자기관리만
retriever.search(query="피부")  # 통합 검색
```

### LangChain/LangGraph 도입

**Before (단순 OpenAI API):**

```python
client = OpenAI(api_key=...)
response = client.chat.completions.create(...)
```

**After (LangGraph Workflow):**

```python
workflow = create_report_workflow()
result = await workflow.ainvoke(initial_state)
```

---

## 📊 Workflow 구조

```
┌─────────────────────┐
│  analyze_images     │  Node 1: GPT-4o Vision 분석
│  (Structured Output)│
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  match_knowledge    │  Node 2: RAG 기반 매칭
│  (Semantic Search)  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  generate_report    │  Node 3: DB 저장
│  (Transaction)      │
└─────────────────────┘
```

---

## 📁 파일 구조

```
docs/implementation/06-report-generation/
├── README.md (이 파일)
├── api.md
├── tasks.md                       ✨ NEW: Celery Task
└── services/
    ├── langchain-workflow.md      ✨ NEW: LangGraph Workflow
    ├── knowledge-retriever.md     ✨ NEW: RAG Retriever
    ├── openai-client.md           🔧 UPDATED: Deprecated
    └── report-builder.md          🔧 UPDATED: 단순화
```

---

## 🚀 구현 순서

### Step 1: Celery Task 구현

**파일:** `tasks.md`

**핵심 내용:**

- @celery_app.task 데코레이터
- LangChain Workflow 실행
- Job 상태 업데이트 (completed)
- KST 시간대 사용

### Step 2: LangChain Workflow 구현

**파일:** `services/langchain-workflow.md`

**핵심 내용:**

- 3개 노드 (analyze_images, match_knowledge, generate_report)
- TypedDict 기반 상태 관리
- Pydantic Structured Output

### Step 3: Knowledge Retriever 구현 (Polymorphic)

**파일:** `services/knowledge-retriever.md`

**핵심 내용:**

- **Polymorphic Search**: kb_procedures + kb_self_care_items 통합 검색
- **Keyword Search**: 두 테이블에서 후보 검색
- **LLM Scoring**: 관련성 점수 계산
- **Top-K 반환**: 점수 기준 정렬

### Step 4: Report Builder 단순화

**파일:** `services/report-builder.md`

**핵심 내용:**

- Workflow 결과만 DB에 저장
- 트랜잭션 관리
- 포맷팅 함수

---

## �� 사용 예시

### 전체 Workflow 실행

```python
from app.services.langchain_workflow import run_report_workflow

# Workflow 실행
result = await run_report_workflow(
    job_id="123e4567-e89b-12d3-a456-426614174000",
    image_urls=[
        "https://s3.amazonaws.com/bucket/image1.jpg",
        "https://s3.amazonaws.com/bucket/image2.jpg",
        "https://s3.amazonaws.com/bucket/image3.jpg"
    ]
)

# 결과
print(f"Report ID: {result['report_id']}")
print(f"Analysis: {result['analysis_result']}")
print(f"Matched: {len(result['matched_knowledge'])} items")
```

### API 엔드포인트

```python
# app/api/v1/jobs.py

@router.post("/{job_id}/generate-report")
async def generate_report(job_id: str):
    """리포트 생성 API"""

    # Step 1: Job 조회
    job = await get_job(job_id)

    # Step 2: 이미지 URL 수집
    image_urls = [
        file.s3_url
        for file in job.files
        if file.file_type == "generated"
    ]

    # Step 3: Workflow 실행
    result = await run_report_workflow(
        job_id=job_id,
        image_urls=image_urls
    )

    return {"report_id": result['report_id']}
```

---

## 🎯 장점

### 1. 타입 안정성

```python
# Pydantic으로 응답 구조 보장
class AnalysisResult(BaseModel):
    overall_impression: str
    improvement_areas: List[str]
    specific_observations: dict
    confidence_score: float  # 0.0 ~ 1.0
```

### 2. 에러 처리

```python
# 각 노드별 에러 핸들링
if state.get('error'):
    logger.warning(f"Skipping due to previous error")
    return state
```

### 3. 모니터링

```python
# LangSmith 자동 통합
# - 각 노드 실행 시간
# - LLM 호출 로그
# - 에러 추적
```

### 4. 테스트 용이

```python
# Mock 가능
async def mock_analyze_images_node(state):
    state['analysis_result'] = AnalysisResult(...)
    return state
```

---

## ✅ 체크리스트

- [x] LangGraph Workflow 구현
- [x] RAG Retriever 구현
- [x] Structured Output 적용
- [x] Report Builder 단순화
- [x] OpenAI Client Deprecated 표시
- [ ] LangSmith 설정
- [ ] 통합 테스트
- [ ] 성능 최적화

---

## 📚 참고 문서

- `docs/technical/langchain_architecture.md` - 아키텍처 상세 설명
- `docs/implementation/02-database/models/report.md` - Report 모델
- `docs/implementation/02-database/models/report-knowledge-item.md` - 매핑 모델

---

## 🔧 기술 스택

- **LangChain:** 0.1.0
- **LangGraph:** 0.0.20
- **LangSmith:** 0.0.77
- **OpenAI:** GPT-4o, text-embedding-3-small
- **Pydantic:** 2.0+
