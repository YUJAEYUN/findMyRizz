# Knowledge Retriever (RAG)

## 📋 개요

RAG (Retrieval-Augmented Generation) 기반 Knowledge Base 검색

**새 스키마 반영:**

- kb_procedures (26 rows, 100% RDBMS)
- kb_self_care_items (5 rows, RDBMS + JSONB)
- Polymorphic 검색 지원

---

## 🎯 목적

1. **Polymorphic Search** - 두 테이블 통합 검색
2. **Metadata Filtering** - 카테고리, 타입 필터링
3. **Relevance Scoring** - 관련성 점수 계산
4. **LLM 기반 매칭** - GPT로 매칭 근거 생성

---

## 📐 구현

```python
# app/services/knowledge_retriever.py

from typing import List, Optional, Dict, Union
from sqlalchemy import select, func, or_
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import joinedload
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
from pydantic import BaseModel
import logging

from app.models.kb_procedure import KBProcedure
from app.models.kb_self_care_item import KBSelfCareItem
from app.models.kb_category import KBCategory

logger = logging.getLogger(__name__)


# ===== Response Model =====

class KnowledgeMatch(BaseModel):
    """매칭된 Knowledge 아이템"""
    item_id: str  # P001, SC001
    item_name: str
    item_type: str  # PROCEDURE, SELF_CARE
    category_name: str

    # 매칭 정보
    relevance_score: float  # 0.0 ~ 1.0
    match_reason: str

    # 아이템 정보
    effect_duration: Optional[str]
    cost_range: Optional[str]
    downtime: Optional[str]


# ===== Retriever =====

class KnowledgeRetriever:
    """
    Knowledge Base RAG Retriever (Polymorphic)

    kb_procedures + kb_self_care_items 통합 검색
    """

    def __init__(self):
        """초기화"""
        # LLM (매칭 근거 생성용)
        self.llm = ChatOpenAI(
            model="gpt-4o-mini",
            temperature=0.3
        )

        logger.info("KnowledgeRetriever initialized")

    async def search(
        self,
        query: str,
        observations: str = "",
        top_k: int = 5,
        item_type: Optional[str] = None,  # "PROCEDURE" or "SELF_CARE"
        db: Optional[AsyncSession] = None
    ) -> List[KnowledgeMatch]:
        """
        Knowledge 검색 (Polymorphic)

        Args:
            query: 검색 쿼리 (예: "피부")
            observations: 구체적 관찰 사항 (예: "모공이 보임")
            top_k: 반환할 최대 개수
            item_type: 아이템 타입 필터 ("PROCEDURE" or "SELF_CARE")
            db: DB 세션

        Returns:
            매칭된 Knowledge 리스트
        """
        logger.info(f"Searching knowledge: query='{query}', observations='{observations}'")

        # Step 1: DB 세션 확보
        if db is None:
            from app.core.database import get_db
            async with get_db() as db:
                return await self._search_internal(query, observations, top_k, item_type, db)
        else:
            return await self._search_internal(query, observations, top_k, item_type, db)

    async def _search_internal(
        self,
        query: str,
        observations: str,
        top_k: int,
        item_type: Optional[str],
        db: AsyncSession
    ) -> List[KnowledgeMatch]:
        """내부 검색 로직 (Polymorphic)"""

        # Step 1: 키워드 기반 후보 검색 (두 테이블 통합)
        candidates = await self._keyword_search(
            db=db,
            query=query,
            item_type=item_type,
            limit=top_k * 3  # 후보는 많이
        )

        if not candidates:
            logger.warning(f"No candidates found for query='{query}'")
            return []

        # Step 2: LLM으로 관련성 점수 및 매칭 근거 생성
        matches = await self._score_candidates(
            candidates=candidates,
            query=query,
            observations=observations
        )

        # Step 3: 점수 기준 정렬 및 상위 K개 반환
        sorted_matches = sorted(
            matches,
            key=lambda x: x.relevance_score,
            reverse=True
        )[:top_k]

        logger.info(f"Found {len(sorted_matches)} matches")
        return sorted_matches

    async def _keyword_search(
        self,
        db: AsyncSession,
        query: str,
        item_type: Optional[str],
        limit: int
    ) -> List[Dict]:
        """
        키워드 기반 후보 검색 (Polymorphic)

        Args:
            db: DB 세션
            query: 검색 쿼리
            item_type: 타입 필터 ("PROCEDURE" or "SELF_CARE")
            limit: 최대 개수

        Returns:
            후보 아이템 리스트 (dict 형태)
        """
        candidates = []
        search_pattern = f"%{query}%"

        # Step 1: kb_procedures 검색
        if item_type is None or item_type == "PROCEDURE":
            stmt = select(KBProcedure).options(
                joinedload(KBProcedure.category)
            ).where(
                or_(
                    KBProcedure.name_ko.ilike(search_pattern),
                    KBProcedure.name_en.ilike(search_pattern),
                    KBProcedure.principle.ilike(search_pattern)
                )
            ).limit(limit)

            result = await db.execute(stmt)
            procedures = result.unique().scalars().all()

            for proc in procedures:
                candidates.append({
                    "item_id": proc.id,
                    "item_name": proc.name_ko,
                    "item_type": "PROCEDURE",
                    "category_name": proc.category.name,
                    "effect_duration": proc.effect_duration,
                    "cost_range": proc.cost_range,
                    "downtime": proc.downtime,
                    "main_effects": proc.main_effects,
                    "pros": proc.pros,
                    "cons": proc.cons
                })

        # Step 2: kb_self_care_items 검색
        if item_type is None or item_type == "SELF_CARE":
            stmt = select(KBSelfCareItem).options(
                joinedload(KBSelfCareItem.category)
            ).where(
                or_(
                    KBSelfCareItem.name_ko.ilike(search_pattern),
                    KBSelfCareItem.name_en.ilike(search_pattern),
                    KBSelfCareItem.goal.ilike(search_pattern)
                )
            ).limit(limit)

            result = await db.execute(stmt)
            self_care_items = result.unique().scalars().all()

            for item in self_care_items:
                candidates.append({
                    "item_id": item.id,
                    "item_name": item.name_ko,
                    "item_type": "SELF_CARE",
                    "category_name": item.category.name,
                    "effect_duration": item.effect_duration,
                    "cost_range": item.cost_range,
                    "downtime": item.downtime,
                    "effects": item.effects,
                    "pros": item.pros,
                    "cons": item.cons
                })

        logger.info(f"Keyword search found {len(candidates)} candidates")
        return candidates[:limit]

    async def _score_candidates(
        self,
        candidates: List[Dict],
        query: str,
        observations: str
    ) -> List[KnowledgeMatch]:
        """
        LLM으로 후보 점수 매기기

        Args:
            candidates: 후보 아이템 리스트 (dict)
            query: 검색 쿼리
            observations: 구체적 관찰

        Returns:
            점수가 매겨진 매칭 리스트
        """
        matches = []

        for item in candidates:
            # Step 1: 프롬프트 구성
            effects_text = ", ".join(item.get("main_effects") or item.get("effects") or [])
            pros_text = ", ".join(item.get("pros") or [])
            cons_text = ", ".join(item.get("cons") or [])

            prompt = f"""
당신은 외모 개선 추천 전문가입니다.

**사용자 요구사항:**
- 개선 영역: {query}
- 구체적 관찰: {observations}

**Knowledge 아이템:**
- 이름: {item['item_name']}
- 타입: {item['item_type']}
- 카테고리: {item['category_name']}
- 주요 효과: {effects_text}
- 지속기간: {item['effect_duration']}
- 비용: {item['cost_range']}
- 다운타임: {item['downtime']}
- 장점: {pros_text}
- 단점: {cons_text}

**질문:**
1. 이 아이템이 사용자 요구사항에 얼마나 관련이 있습니까? (0.0 ~ 1.0)
2. 추천 근거를 한 문장으로 설명하세요.

**응답 형식 (JSON):**
{{
    "relevance_score": 0.85,
    "match_reason": "피부 모공 개선에 효과적인 레이저 토닝"
}}
"""

            # Step 2: LLM 호출
            try:
                response = await self.llm.ainvoke([
                    HumanMessage(content=prompt)
                ])

                # Step 3: JSON 파싱
                import json
                result = json.loads(response.content)

                # Step 4: KnowledgeMatch 생성
                match = KnowledgeMatch(
                    item_id=item['item_id'],
                    item_name=item['item_name'],
                    item_type=item['item_type'],
                    category_name=item['category_name'],
                    relevance_score=result['relevance_score'],
                    match_reason=result['match_reason'],
                    effect_duration=item['effect_duration'],
                    cost_range=item['cost_range'],
                    downtime=item['downtime']
                )

                matches.append(match)

            except Exception as e:
                logger.error(f"Failed to score item {item['item_id']}: {e}")
                # 실패 시 기본 점수
                matches.append(KnowledgeMatch(
                    item_id=item['item_id'],
                    item_name=item['item_name'],
                    item_type=item['item_type'],
                    category_name=item['category_name'],
                    relevance_score=0.5,
                    match_reason="자동 매칭",
                    effect_duration=item['effect_duration'],
                    cost_range=item['cost_range'],
                    downtime=item['downtime']
                ))

        return matches
```

---

## 🔍 사용 예시

```python
# Workflow에서 사용
from app.services.knowledge_retriever import KnowledgeRetriever

retriever = KnowledgeRetriever()

# 검색
matches = await retriever.search(
    query="피부",
    observations="모공이 보이고 피부톤이 고르지 않음",
    top_k=5
)

for match in matches:
    print(f"{match.item_name}: {match.relevance_score:.2f} - {match.match_reason}")
```

---

## ✅ 테스트 시나리오

```python
import pytest
from app.services.knowledge_retriever import KnowledgeRetriever

@pytest.mark.asyncio
async def test_knowledge_search(db_session):
    """Knowledge 검색 테스트"""
    # Given
    retriever = KnowledgeRetriever()

    # When
    matches = await retriever.search(
        query="피부",
        observations="모공이 보임",
        top_k=3,
        db=db_session
    )

    # Then
    assert len(matches) > 0
    assert all(0.0 <= m.relevance_score <= 1.0 for m in matches)
    assert matches[0].relevance_score >= matches[-1].relevance_score  # 정렬 확인
```

---

## 📚 참고

- `langchain-workflow.md` - Workflow에서 사용
- `docs/technical/langchain_architecture.md` - RAG 아키텍처
