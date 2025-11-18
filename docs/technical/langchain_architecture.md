# LangGraph 아키텍처 설계 (리포트 생성)

## 📋 개요

단순 OpenAI API 호출이 아닌, **LangGraph 기반 RAG (Retrieval-Augmented Generation)** 패턴으로 리포트 생성 시스템을 구축합니다.

**핵심 기술 스택:**

- **LangGraph**: 상태 기반 워크플로우 관리 (메인)
- **LangChain**: LLM 통합 및 프롬프트 관리 (보조)
- **LangSmith**: 전체 프로세스 모니터링 및 디버깅

---

## 🎯 설계 목표

1. **Knowledge Base 활용**: DB의 구조화된 지식을 LLM에 효과적으로 전달
2. **일관성 보장**: 체계적인 워크플로우로 리포트 품질 유지
3. **확장성**: 새로운 분석 단계 추가 용이
4. **추적 가능성**: LangSmith로 전체 프로세스 모니터링

---

## 🏗️ 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                    LangGraph Workflow                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │  Image   │───▶│ Analysis │───▶│ Knowledge│              │
│  │ Analysis │    │ Parsing  │    │ Matching │              │
│  └──────────┘    └──────────┘    └──────────┘              │
│       │               │                 │                    │
│       ▼               ▼                 ▼                    │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │ GPT-4o   │    │ Pydantic │    │   RAG    │              │
│  │  Vision  │    │  Parser  │    │ Retriever│              │
│  └──────────┘    └──────────┘    └──────────┘              │
│                                        │                     │
│                                        ▼                     │
│                                  ┌──────────┐               │
│                                  │PostgreSQL│               │
│                                  │Knowledge │               │
│                                  │   Base   │               │
│                                  └──────────┘               │
│                                                               │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │ Report   │◀───│ Content  │◀───│ Template │              │
│  │Generation│    │ Builder  │    │ Selector │              │
│  └──────────┘    └──────────┘    └──────────┘              │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 📦 기술 스택

```python
# requirements.txt 추가
langchain==0.1.0
langchain-openai==0.0.5
langchain-community==0.0.20
langgraph==0.0.20
langsmith==0.0.77

# Vector Store (향후)
pgvector==0.2.4
langchain-postgres==0.0.3
```

---

## 🔄 LangGraph Workflow 설계

### 1. State 정의

```python
from typing import TypedDict, List, Optional
from pydantic import BaseModel

class AnalysisResult(BaseModel):
    """GPT-4o Vision 분석 결과"""
    overall_impression: str
    improvement_areas: List[str]  # ['피부', '윤곽', '눈', '코', '입술']
    specific_observations: dict   # {'피부': '모공이 보임', '윤곽': 'V라인 필요'}
    confidence_score: float       # 0.0 ~ 1.0

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
    """LangGraph State"""
    # Input
    job_id: str
    image_urls: List[str]  # 3개 생성 이미지

    # Intermediate
    analysis_result: Optional[AnalysisResult]
    matched_knowledge: List[KnowledgeMatch]

    # Output
    report_id: str
    report_content: dict

    # Metadata
    error: Optional[str]
    step: str
```

### 2. Graph 노드 정의

```python
from langgraph.graph import StateGraph, END
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate

# ============================================
# Node 1: Image Analysis (GPT-4o Vision)
# ============================================
async def analyze_images_node(state: ReportState) -> ReportState:
    """
    3개 이미지를 GPT-4o Vision으로 분석

    Returns:
        AnalysisResult with structured output
    """
    llm = ChatOpenAI(
        model="gpt-4o",
        temperature=0.3,
        max_tokens=2000
    )

    # Structured Output with Pydantic
    structured_llm = llm.with_structured_output(AnalysisResult)

    prompt = ChatPromptTemplate.from_messages([
        ("system", """당신은 전문 피부과 의사이자 미용 컨설턴트입니다.

사용자의 AI 생성 이미지 3장을 분석하여:
1. 전체적인 인상
2. 개선이 필요한 영역 (피부, 윤곽, 눈, 코, 입술 중 선택)
3. 각 영역별 구체적인 관찰 사항
4. 분석 신뢰도

를 제공하세요.

**중요**: 과장하지 말고 현실적이고 달성 가능한 개선 사항만 언급하세요."""),
        ("user", [
            {"type": "text", "text": "다음 3장의 이미지를 분석해주세요:"},
            {"type": "image_url", "image_url": {"url": state["image_urls"][0]}},
            {"type": "image_url", "image_url": {"url": state["image_urls"][1]}},
            {"type": "image_url", "image_url": {"url": state["image_urls"][2]}},
        ])
    ])

    chain = prompt | structured_llm

    try:
        result = await chain.ainvoke({})
        state["analysis_result"] = result
        state["step"] = "analysis_complete"
    except Exception as e:
        state["error"] = f"Image analysis failed: {str(e)}"
        state["step"] = "error"

    return state


# ============================================
# Node 2: Knowledge Matching (RAG)
# ============================================
async def match_knowledge_node(state: ReportState) -> ReportState:
    """
    분석 결과를 기반으로 Knowledge Base에서 관련 항목 검색

    Uses:
        - Semantic search (향후 Vector Store)
        - Metadata filtering
        - Relevance scoring
    """
    from app.services.knowledge_retriever import KnowledgeRetriever

    retriever = KnowledgeRetriever()
    analysis = state["analysis_result"]

    matched_items = []

    # Step 1: 개선 영역별로 Knowledge 검색 (Polymorphic)
    for area in analysis.improvement_areas:
        # 예: area = '피부'
        observation = analysis.specific_observations.get(area, "")

        # 시술 검색
        procedures = await retriever.search(
            query=area,
            observations=observation,
            item_type="PROCEDURE",
            top_k=3
        )
        matched_items.extend(procedures)

        # 자기관리 검색
        self_care = await retriever.search(
            query=area,
            observations=observation,
            item_type="SELF_CARE",
            top_k=2
        )
        matched_items.extend(self_care)

    # Step 3: Relevance score로 정렬 및 중복 제거
    unique_items = {}
    for item in matched_items:
        if item.item_id not in unique_items:
            unique_items[item.item_id] = item
        elif item.relevance_score > unique_items[item.item_id].relevance_score:
            unique_items[item.item_id] = item

    state["matched_knowledge"] = sorted(
        unique_items.values(),
        key=lambda x: x.relevance_score,
        reverse=True
    )[:10]  # 최대 10개

    state["step"] = "matching_complete"
    return state


# ============================================
# Node 3: Report Generation
# ============================================
async def generate_report_node(state: ReportState) -> ReportState:
    """
    분석 결과 + 매칭된 Knowledge를 기반으로 리포트 생성
    """
    from app.services.report_builder import ReportBuilder

    builder = ReportBuilder()

    report_content = await builder.build(
        analysis=state["analysis_result"],
        knowledge_items=state["matched_knowledge"]
    )

    # DB에 저장
    from app.models.report import Report
    from app.core.database import get_db

    async with get_db() as db:
        report = Report(
            job_id=state["job_id"],
            analysis_summary=state["analysis_result"].overall_impression,
            improvement_areas=state["analysis_result"].improvement_areas,
        )
        db.add(report)
        await db.commit()

        # Knowledge 매칭 저장 (Polymorphic)
        for idx, item in enumerate(state["matched_knowledge"]):
            mapping = ReportKnowledgeItem(
                report_id=report.report_id,
                item_id=item.item_id,  # P001, SC001
                item_type=item.item_type,  # PROCEDURE, SELF_CARE
                relevance_score=item.relevance_score,
                match_reason=item.match_reason,
                display_order=idx + 1
            )
            db.add(mapping)

        await db.commit()

        state["report_id"] = str(report.report_id)
        state["report_content"] = report_content
        state["step"] = "complete"

    return state


# ============================================
# Graph 구성
# ============================================
def create_report_workflow() -> StateGraph:
    """LangGraph Workflow 생성"""

    workflow = StateGraph(ReportState)

    # 노드 추가
    workflow.add_node("analyze_images", analyze_images_node)
    workflow.add_node("match_knowledge", match_knowledge_node)
    workflow.add_node("generate_report", generate_report_node)

    # 엣지 정의
    workflow.set_entry_point("analyze_images")

    workflow.add_conditional_edges(
        "analyze_images",
        lambda state: "error" if state.get("error") else "continue",
        {
            "continue": "match_knowledge",
            "error": END
        }
    )

    workflow.add_edge("match_knowledge", "generate_report")
    workflow.add_edge("generate_report", END)

    return workflow.compile()
```

---

## 🔍 Knowledge Retriever (RAG)

```python
# app/services/knowledge_retriever.py

from typing import List, Dict, Optional
from sqlalchemy import select, and_, or_
from sqlalchemy.ext.asyncio import AsyncSession

class KnowledgeRetriever:
    """Knowledge Base RAG Retriever"""

    def __init__(self, db: AsyncSession):
        self.db = db

    async def search(
        self,
        query: str,
        filters: Optional[Dict] = None,
        top_k: int = 5
    ) -> List[KnowledgeMatch]:
        """
        Semantic search + Metadata filtering

        Args:
            query: 검색 쿼리 (예: "피부 개선: 모공이 보임")
            filters: 메타데이터 필터 (예: {"category_type": "procedure"})
            top_k: 반환할 최대 개수

        Returns:
            매칭된 Knowledge 아이템 리스트
        """
        # Step 1: Keyword matching (현재)
        # 향후: Vector Store로 업그레이드

        stmt = select(KBItem).where(KBItem.is_active == True)

        # Filters 적용
        if filters:
            if "category_type" in filters:
                stmt = stmt.join(KBCategory).where(
                    KBCategory.category_type == filters["category_type"]
                )
            if "difficulty_level" in filters:
                stmt = stmt.join(KBSelfCareDetails).where(
                    KBSelfCareDetails.difficulty_level == filters["difficulty_level"]
                )

        # Keyword search (간단한 버전)
        keywords = self._extract_keywords(query)
        if keywords:
            conditions = [
                or_(
                    KBItem.name.ilike(f"%{kw}%"),
                    KBItem.description.ilike(f"%{kw}%")
                )
                for kw in keywords
            ]
            stmt = stmt.where(or_(*conditions))

        result = await self.db.execute(stmt.limit(top_k))
        items = result.scalars().all()

        # Step 2: Relevance scoring
        matches = []
        for item in items:
            score = self._calculate_relevance(query, item)
            matches.append(KnowledgeMatch(
                item_id=item.item_id,
                item_code=item.item_code,
                item_name=item.name,
                item_type=item.item_type,
                relevance_score=score,
                match_reason=self._generate_match_reason(query, item)
            ))

        return sorted(matches, key=lambda x: x.relevance_score, reverse=True)

    def _extract_keywords(self, query: str) -> List[str]:
        """쿼리에서 키워드 추출"""
        # 간단한 버전: 공백으로 분리
        # 향후: 형태소 분석기 (KoNLPy) 사용
        return [w for w in query.split() if len(w) > 1]

    def _calculate_relevance(self, query: str, item: KBItem) -> float:
        """Relevance score 계산 (0.0 ~ 1.0)"""
        score = 0.0
        keywords = self._extract_keywords(query)

        for kw in keywords:
            if kw in item.name:
                score += 0.5
            if kw in (item.description or ""):
                score += 0.3

        return min(score, 1.0)

    def _generate_match_reason(self, query: str, item: KBItem) -> str:
        """매칭 이유 생성"""
        return f"'{query}'와 관련된 {item.item_type} 항목"
```

---

## 📊 LangSmith 통합

```python
# app/core/config.py

class Settings(BaseSettings):
    # ... 기존 설정

    # LangSmith
    LANGCHAIN_TRACING_V2: bool = True
    LANGCHAIN_API_KEY: str
    LANGCHAIN_PROJECT: str = "find-my-rizz"
```

```python
# app/services/report_service.py

from langsmith import traceable

@traceable(name="generate_report_full_workflow")
async def generate_report(job_id: str) -> dict:
    """
    전체 리포트 생성 워크플로우

    LangSmith로 추적됨
    """
    workflow = create_report_workflow()

    # 이미지 URL 가져오기
    image_urls = await get_generated_image_urls(job_id)

    # Workflow 실행
    result = await workflow.ainvoke({
        "job_id": job_id,
        "image_urls": image_urls,
        "step": "start"
    })

    return result
```

---

## 🚀 향후 고도화

### 1. Vector Store 도입

```python
from langchain_postgres import PGVector

# pgvector 확장 설치
# CREATE EXTENSION vector;

embeddings = OpenAIEmbeddings()

vector_store = PGVector(
    connection_string=settings.DATABASE_URL,
    embedding_function=embeddings,
    collection_name="knowledge_base"
)

# Knowledge 임베딩 생성
for item in kb_items:
    vector_store.add_texts(
        texts=[f"{item.name}: {item.description}"],
        metadatas=[{
            "item_id": item.item_id,
            "item_type": item.item_type,
            "category_id": item.category_id
        }]
    )
```

### 2. Multi-Agent 시스템

```python
# 전문가 Agent들
skin_expert = create_agent("피부 전문가")
contour_expert = create_agent("윤곽 전문가")
eye_expert = create_agent("눈 전문가")

# Supervisor Agent
supervisor = create_supervisor([skin_expert, contour_expert, eye_expert])
```

---

## ✅ 장점

1. **체계적인 워크플로우**: LangGraph로 명확한 단계 정의
2. **Knowledge 활용**: DB의 구조화된 지식을 효과적으로 사용
3. **추적 가능성**: LangSmith로 전체 프로세스 모니터링
4. **확장성**: 새로운 노드 추가 용이
5. **일관성**: Pydantic으로 타입 안정성 보장
