# LangChain Workflow (LangGraph)

## 📋 개요

LangGraph를 사용한 리포트 생성 Workflow 구현

---

## 🎯 목적

1. **구조화된 Workflow** - 3개 노드로 명확한 단계 분리
2. **상태 관리** - TypedDict 기반 상태 추적
3. **에러 처리** - 각 노드별 에러 핸들링
4. **LangSmith 통합** - 모니터링 및 디버깅

---

## 📐 Workflow 정의

```python
# app/services/langchain_workflow.py

from typing import TypedDict, List, Optional, Annotated
from langgraph.graph import StateGraph, END
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
from pydantic import BaseModel, Field
import logging

logger = logging.getLogger(__name__)


# ===== State 정의 =====

class AnalysisResult(BaseModel):
    """GPT-4o Vision 분석 결과 (스토리텔링 형식)"""
    title: str = Field(description="리포트 제목 (예: '더 선명한 나를 만드는, 3가지 변화')")
    subtitle: str = Field(description="부제목 (예: '다이어트 6Kg + 피부 관리로 완성한 V라인')")
    story_paragraphs: List[str] = Field(description="스토리텔링 문단 리스트 (3-5개)")
    improvement_areas: List[str] = Field(description="개선 영역 태그 (예: ['#다이어트5Kg', '#V라인완성'])")
    confidence_score: float = Field(ge=0.0, le=1.0, description="분석 신뢰도")


class KnowledgeMatch(BaseModel):
    """매칭된 Knowledge 아이템 (Polymorphic)"""
    item_id: str  # P001, SC001
    item_name: str
    item_type: str  # PROCEDURE, SELF_CARE
    category_name: str
    relevance_score: float
    match_reason: str
    effect_duration: Optional[str]
    cost_range: Optional[str]
    downtime: Optional[str]


class ReportState(TypedDict):
    """Workflow 상태"""
    # Input
    job_id: str
    image_urls: List[str]

    # Node 1: analyze_images
    analysis_result: Optional[AnalysisResult]

    # Node 2: match_knowledge
    matched_knowledge: List[KnowledgeMatch]

    # Node 3: generate_report
    report_id: str
    report_content: dict

    # Error handling
    error: Optional[str]
    step: str  # 현재 단계


# ===== Node 1: 이미지 분석 =====

async def analyze_images_node(state: ReportState) -> ReportState:
    """
    Node 1: GPT-4o Vision으로 이미지 분석

    Args:
        state: Workflow 상태

    Returns:
        업데이트된 상태 (analysis_result 포함)
    """
    logger.info(f"[Node 1] Starting image analysis for job_id={state['job_id']}")

    try:
        # Step 1: LLM 초기화 (Structured Output)
        llm = ChatOpenAI(
            model="gpt-4o",
            temperature=0.3,
            max_tokens=2000
        )
        structured_llm = llm.with_structured_output(AnalysisResult)

        # Step 2: 프롬프트 구성
        prompt = f"""
당신은 외모 분석 전문가이자 스토리텔러입니다. 제공된 이미지를 분석하여 자연스러운 스토리텔링 형식의 리포트를 작성하세요.

**리포트 구성:**

1. **title**: 매력적인 제목 (예: "더 선명한 나를 만드는, 3가지 변화")
2. **subtitle**: 구체적인 부제목 (예: "다이어트 6Kg + 피부 관리로 완성한 V라인")
3. **story_paragraphs**: 3-5개의 자연스러운 문단으로 구성된 스토리
   - 첫 문단: 현재 당신의 얼굴 분석 (긍정적 + 개선점)
   - 중간 문단들: 각 개선 영역별 구체적인 관찰과 제안
   - 마지막 문단: 격려와 기대효과
4. **improvement_areas**: 해시태그 형식의 개선 영역 (예: ["#다이어트5Kg", "#V라인완성", "#턱선정리", "#레이저토닝", "#기초화장"])
5. **confidence_score**: 분석 신뢰도 (0.0 ~ 1.0)

**스토리텔링 가이드:**
- 친근하고 격려하는 톤 사용
- 구체적인 수치와 예시 포함 (예: "5kg 감량", "피부톤 23% 개선")
- 부정적 표현 대신 긍정적 변화 강조
- 실현 가능한 목표 제시

이미지 URL: {state['image_urls']}
"""

        # Step 3: 이미지 포함 메시지 생성
        messages = [
            HumanMessage(
                content=[
                    {"type": "text", "text": prompt},
                    *[
                        {"type": "image_url", "image_url": {"url": url}}
                        for url in state['image_urls']
                    ]
                ]
            )
        ]

        # Step 4: LLM 호출
        logger.info(f"[Node 1] Calling GPT-4o Vision with {len(state['image_urls'])} images")
        analysis_result = await structured_llm.ainvoke(messages)

        # Step 5: 상태 업데이트
        state['analysis_result'] = analysis_result
        state['step'] = 'analyze_images_completed'

        logger.info(f"[Node 1] Analysis completed. Title: {analysis_result.title}")
        return state

    except Exception as e:
        logger.error(f"[Node 1] Error: {str(e)}", exc_info=True)
        state['error'] = f"Image analysis failed: {str(e)}"
        state['step'] = 'analyze_images_failed'
        return state


# ===== Node 2: Knowledge 매칭 =====

async def match_knowledge_node(state: ReportState) -> ReportState:
    """
    Node 2: RAG로 Knowledge Base 매칭

    Args:
        state: Workflow 상태

    Returns:
        업데이트된 상태 (matched_knowledge 포함)
    """
    logger.info(f"[Node 2] Starting knowledge matching for job_id={state['job_id']}")

    # 이전 노드 실패 시 스킵
    if state.get('error'):
        logger.warning(f"[Node 2] Skipping due to previous error")
        return state

    try:
        from app.services.knowledge_retriever import KnowledgeRetriever

        # Step 1: Retriever 초기화
        retriever = KnowledgeRetriever()

        # Step 2: 개선 영역별로 Knowledge 검색
        analysis = state['analysis_result']
        all_matches = []

        for area in analysis.improvement_areas:
            logger.info(f"[Node 2] Searching knowledge for area: {area}")

            # Step 3: RAG 검색
            matches = await retriever.search(
                query=area,
                observations=analysis.specific_observations.get(area, ""),
                top_k=3
            )

            all_matches.extend(matches)

        # Step 4: 중복 제거 및 정렬 (relevance_score 기준)
        unique_matches = {m.item_id: m for m in all_matches}
        sorted_matches = sorted(
            unique_matches.values(),
            key=lambda x: x.relevance_score,
            reverse=True
        )[:10]  # 상위 10개

        # Step 5: 상태 업데이트
        state['matched_knowledge'] = sorted_matches
        state['step'] = 'match_knowledge_completed'

        logger.info(f"[Node 2] Matched {len(sorted_matches)} knowledge items")
        return state

    except Exception as e:
        logger.error(f"[Node 2] Error: {str(e)}", exc_info=True)
        state['error'] = f"Knowledge matching failed: {str(e)}"
        state['step'] = 'match_knowledge_failed'
        return state


# ===== Node 3: 리포트 생성 =====

async def generate_report_node(state: ReportState) -> ReportState:
    """
    Node 3: 리포트 생성 및 DB 저장

    Args:
        state: Workflow 상태

    Returns:
        업데이트된 상태 (report_id, report_content 포함)
    """
    logger.info(f"[Node 3] Starting report generation for job_id={state['job_id']}")

    # 이전 노드 실패 시 스킵
    if state.get('error'):
        logger.warning(f"[Node 3] Skipping due to previous error")
        return state

    try:
        from app.services.report_builder import ReportBuilder
        from app.core.database import get_db

        # Step 1: ReportBuilder 초기화
        builder = ReportBuilder()

        # Step 2: 리포트 생성
        async with get_db() as db:
            report = await builder.create_report(
                db=db,
                job_id=state['job_id'],
                analysis_result=state['analysis_result'],
                matched_knowledge=state['matched_knowledge']
            )

        # Step 3: 상태 업데이트
        state['report_id'] = str(report.report_id)
        state['report_content'] = report.to_dict()
        state['step'] = 'generate_report_completed'

        logger.info(f"[Node 3] Report created: {report.report_id}")
        return state

    except Exception as e:
        logger.error(f"[Node 3] Error: {str(e)}", exc_info=True)
        state['error'] = f"Report generation failed: {str(e)}"
        state['step'] = 'generate_report_failed'
        return state


# ===== Workflow 구성 =====

def create_report_workflow() -> StateGraph:
    """
    리포트 생성 Workflow 생성

    Returns:
        LangGraph StateGraph
    """
    # Step 1: Graph 초기화
    workflow = StateGraph(ReportState)

    # Step 2: 노드 추가
    workflow.add_node("analyze_images", analyze_images_node)
    workflow.add_node("match_knowledge", match_knowledge_node)
    workflow.add_node("generate_report", generate_report_node)

    # Step 3: 엣지 정의
    workflow.set_entry_point("analyze_images")
    workflow.add_edge("analyze_images", "match_knowledge")
    workflow.add_edge("match_knowledge", "generate_report")
    workflow.add_edge("generate_report", END)

    # Step 4: 컴파일
    return workflow.compile()


# ===== 실행 함수 =====

async def run_report_workflow(
    job_id: str,
    image_urls: List[str]
) -> dict:
    """
    리포트 생성 Workflow 실행

    Args:
        job_id: Job ID
        image_urls: 이미지 URL 리스트

    Returns:
        최종 상태 딕셔너리

    Raises:
        Exception: Workflow 실행 실패 시
    """
    logger.info(f"Starting report workflow for job_id={job_id}")

    # Step 1: 초기 상태 생성
    initial_state = ReportState(
        job_id=job_id,
        image_urls=image_urls,
        analysis_result=None,
        matched_knowledge=[],
        report_id="",
        report_content={},
        error=None,
        step="initialized"
    )

    # Step 2: Workflow 생성
    workflow = create_report_workflow()

    # Step 3: 실행
    final_state = await workflow.ainvoke(initial_state)

    # Step 4: 에러 체크
    if final_state.get('error'):
        logger.error(f"Workflow failed at step={final_state['step']}: {final_state['error']}")
        raise Exception(final_state['error'])

    logger.info(f"Workflow completed successfully. Report ID: {final_state['report_id']}")
    return final_state
```

---

## 📝 리포트 예시

### 입력 이미지

- 사용자 얼굴 사진 (정면, 측면 등)

### 출력 리포트

```json
{
  "title": "더 선명한 나를 만드는, 3가지 변화",
  "subtitle": "다이어트 6Kg + 피부 관리로 완성한 V라인",
  "story_paragraphs": [
    "현재 당신의 얼굴은 약간의 부종과 지방층으로 인해 본래의 날카로운 윤곽이 숨겨져 있는 상태입니다. 특히 턱선이 불분명하고 볼에 둥글게 지방이 분포되어 있어 전체적으로 부드러운 인상을 줍니다.",
    "체계적인 다이어트를 통해 5kg 감량을 목표로 하면 얼굴 지방층이 자연스럽게 감소하며 V라인이 드러날 것입니다. 덕 아래 지방이 감소하면서 목 턱선이 각지게 선명해지고, 광대뼈가 살짝 드러나지면서 입체감이 살아납니다.",
    "덕 아래 지방이 감소하면서 목 턱선이 각지게 선명해지고, 광대뼈가 살짝 드러나지면서 입체감이 살아납니다. 의학 연구에 따르면 5kg 체중 감량 시 얼굴 지방은 평균 23% 감소하며, 턱선도 약 12-15도 더 날카로워집니다.",
    "여기에 기초적인 피부 관리를 더하면 완벽합니다. 레이저 토닝으로 색소를 개선하고 기초 화장으로 피부결을 정돈하면, 건강하고 깨끗한 인상이 완성됩니다. 간단한 관리지만 전체적인 완성도를 높여줄 것입니다."
  ],
  "improvement_areas": [
    "#다이어트5Kg",
    "#V라인완성",
    "#턱선정리",
    "#레이저토닝",
    "#기초화장"
  ],
  "confidence_score": 0.85
}
```

---

## 🔍 사용 예시

```python
# app/api/v1/jobs.py

from app.services.langchain_workflow import run_report_workflow

@router.post("/{job_id}/generate-report")
async def generate_report(job_id: str):
    """리포트 생성 API"""

    # Step 1: Job 조회
    job = await get_job(job_id)

    # Step 2: 이미지 URL 수집
    image_urls = [file.s3_url for file in job.files if file.file_type == "generated"]

    # Step 3: Workflow 실행
    try:
        result = await run_report_workflow(
            job_id=job_id,
            image_urls=image_urls
        )

        return {"report_id": result['report_id']}

    except Exception as e:
        logger.error(f"Report generation failed: {e}")
        raise HTTPException(status_code=500, detail=str(e))
```

---

## 📚 참고

- `docs/technical/langchain_architecture.md` - 아키텍처 상세 설명
- 다음 문서: `knowledge-retriever.md` - RAG Retriever 구현
