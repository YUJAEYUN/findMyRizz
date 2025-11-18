# Report Builder (LangChain 통합)

## 📋 개요

LangGraph Workflow 결과를 DB에 저장하는 Report Builder

---

## 🎯 목적

1. **Report 생성** - Workflow 결과를 DB에 저장
2. **Knowledge 매핑** - RAG 매칭 결과 저장
3. **트랜잭션 관리** - 원자성 보장

---

## 📐 구현

```python
# app/services/report_builder.py

from sqlalchemy.ext.asyncio import AsyncSession
from typing import List
import logging
import uuid

from app.models.report import Report
from app.models.report_knowledge_item import ReportKnowledgeItem
from app.models.job import Job, JobStatus
from app.services.langchain_workflow import AnalysisResult, KnowledgeMatch

logger = logging.getLogger(__name__)


class ReportBuilder:
    """
    Report 생성 및 저장
    
    LangGraph Workflow 결과를 DB에 저장
    """
    
    async def create_report(
        self,
        db: AsyncSession,
        job_id: str,
        analysis_result: AnalysisResult,
        matched_knowledge: List[KnowledgeMatch]
    ) -> Report:
        """
        리포트 생성 및 저장
        
        Args:
            db: DB 세션
            job_id: Job ID
            analysis_result: GPT-4o Vision 분석 결과
            matched_knowledge: RAG 매칭 결과
        
        Returns:
            생성된 Report
        
        Raises:
            Exception: DB 저장 실패 시
        """
        logger.info(f"Creating report for job_id={job_id}")
        
        try:
            # Step 1: Report 생성
            report = Report(
                job_id=uuid.UUID(job_id),
                analysis_summary=analysis_result.overall_impression,
                improvement_areas=analysis_result.improvement_areas,
                specific_observations=analysis_result.specific_observations,
                confidence_score=analysis_result.confidence_score
            )
            
            db.add(report)
            await db.flush()  # report_id 생성
            
            logger.info(f"Report created: {report.report_id}")
            
            # Step 2: Knowledge 매핑 생성
            for idx, match in enumerate(matched_knowledge):
                mapping = ReportKnowledgeItem(
                    report_id=report.report_id,
                    item_id=match.item_id,
                    relevance_score=match.relevance_score,
                    match_reason=match.match_reason,
                    display_order=idx + 1
                )
                db.add(mapping)
            
            logger.info(f"Created {len(matched_knowledge)} knowledge mappings")
            
            # Step 3: Job 상태 업데이트
            job = await db.get(Job, uuid.UUID(job_id))
            if job:
                job.status = JobStatus.COMPLETED
                logger.info(f"Job status updated to COMPLETED")
            
            # Step 4: 커밋
            await db.commit()
            await db.refresh(report)
            
            logger.info(f"Report saved successfully: {report.report_id}")
            return report
            
        except Exception as e:
            logger.error(f"Failed to create report: {e}", exc_info=True)
            await db.rollback()
            raise
    
    async def get_report_with_knowledge(
        self,
        db: AsyncSession,
        report_id: uuid.UUID
    ) -> Report:
        """
        리포트 조회 (Knowledge 포함)
        
        Args:
            db: DB 세션
            report_id: Report ID
        
        Returns:
            Report (knowledge_mappings 포함)
        """
        from sqlalchemy import select
        from sqlalchemy.orm import joinedload
        
        stmt = select(Report).where(
            Report.report_id == report_id
        ).options(
            joinedload(Report.knowledge_mappings).joinedload(
                ReportKnowledgeItem.knowledge_item
            )
        )
        
        result = await db.execute(stmt)
        report = result.unique().scalar_one_or_none()
        
        if not report:
            raise ValueError(f"Report not found: {report_id}")
        
        return report
    
    def format_report_for_display(self, report: Report) -> dict:
        """
        리포트를 화면 표시용으로 포맷
        
        Args:
            report: Report 객체
        
        Returns:
            포맷된 딕셔너리
        """
        return {
            "report_id": str(report.report_id),
            "job_id": str(report.job_id),
            
            # 분석 결과
            "analysis": {
                "summary": report.analysis_summary,
                "improvement_areas": report.improvement_areas,
                "observations": report.specific_observations,
                "confidence": float(report.confidence_score) if report.confidence_score else None
            },
            
            # 추천 Knowledge
            "recommendations": [
                {
                    "item_id": mapping.item_id,
                    "item_name": mapping.knowledge_item.name,
                    "item_type": mapping.knowledge_item.item_type.value,
                    "category": mapping.knowledge_item.category.category_name,
                    "relevance_score": float(mapping.relevance_score),
                    "reason": mapping.match_reason,
                    "duration": mapping.knowledge_item.get_duration_text(),
                    "cost": mapping.knowledge_item.get_cost_text(),
                }
                for mapping in sorted(
                    report.knowledge_mappings,
                    key=lambda x: x.display_order
                )
            ],
            
            "generated_at": report.generated_at.isoformat() if report.generated_at else None
        }
```

---

## 🔍 사용 예시

### 1. Workflow에서 사용

```python
# langchain_workflow.py의 generate_report_node에서

from app.services.report_builder import ReportBuilder

builder = ReportBuilder()

report = await builder.create_report(
    db=db,
    job_id=state['job_id'],
    analysis_result=state['analysis_result'],
    matched_knowledge=state['matched_knowledge']
)
```

### 2. API에서 조회

```python
# app/api/v1/reports.py

from app.services.report_builder import ReportBuilder

@router.get("/{report_id}")
async def get_report(
    report_id: str,
    db: AsyncSession = Depends(get_db)
):
    """리포트 조회"""
    builder = ReportBuilder()
    
    report = await builder.get_report_with_knowledge(
        db=db,
        report_id=uuid.UUID(report_id)
    )
    
    return builder.format_report_for_display(report)
```

---

## ✅ 테스트 시나리오

```python
import pytest
from app.services.report_builder import ReportBuilder
from app.services.langchain_workflow import AnalysisResult, KnowledgeMatch

@pytest.mark.asyncio
async def test_create_report(db_session):
    """리포트 생성 테스트"""
    # Given
    builder = ReportBuilder()
    
    analysis = AnalysisResult(
        overall_impression="전반적으로 깔끔한 인상",
        improvement_areas=["피부", "윤곽"],
        specific_observations={"피부": "모공이 보임"},
        confidence_score=0.85
    )
    
    matches = [
        KnowledgeMatch(
            item_id=1,
            item_code="P001",
            item_name="보톡스",
            relevance_score=0.9,
            match_reason="이마 주름 개선"
        )
    ]
    
    # When
    report = await builder.create_report(
        db=db_session,
        job_id=str(uuid.uuid4()),
        analysis_result=analysis,
        matched_knowledge=matches
    )
    
    # Then
    assert report.report_id is not None
    assert len(report.knowledge_mappings) == 1
    assert report.confidence_score == 0.85
```

---

## 🔄 변경사항

### Before (직접 구현)

```python
# 복잡한 로직
- GPT-4o Vision 직접 호출
- Knowledge 매칭 로직
- Report 생성
```

### After (Workflow 통합)

```python
# 단순화
- Workflow 결과만 DB에 저장
- 트랜잭션 관리에 집중
```

---

## 📚 참고

- `langchain-workflow.md` - generate_report_node에서 사용
- `docs/implementation/02-database/models/report.md` - Report 모델
- `docs/implementation/02-database/models/report-knowledge-item.md` - 매핑 모델
