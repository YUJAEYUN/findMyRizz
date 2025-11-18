# Report 모델

## 📋 개요

AI 분석 리포트 테이블 (LangChain/LangGraph 결과 저장)

---

## 🎯 목적

- LangGraph Workflow 결과 저장
- GPT-4o Vision 분석 결과 구조화
- Knowledge Base 매칭 결과 연결

---

## 📐 모델 정의

```python
# app/models/report.py

from sqlalchemy import Column, String, Text, ForeignKey, Numeric, DateTime, Index, func
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.postgresql import UUID, JSONB
from app.core.database import Base
import uuid

class Report(Base):
    """
    AI 분석 리포트
    
    Job과 1:1 관계
    LangGraph Workflow 결과 저장
    """
    __tablename__ = "reports"
    
    # Primary Key
    report_id = Column(
        UUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid4
    )
    
    # Foreign Keys
    job_id = Column(
        UUID(as_uuid=True),
        ForeignKey("jobs.job_id", ondelete="CASCADE"),
        nullable=False,
        unique=True
    )
    
    # AI 분석 결과 (구조화)
    analysis_summary = Column(
        Text,
        nullable=False,
        comment="전체 요약 (overall_impression)"
    )
    
    improvement_areas = Column(
        JSONB,
        nullable=False,
        comment="개선 영역 리스트 ['피부', '윤곽', '눈']"
    )
    
    specific_observations = Column(
        JSONB,
        nullable=True,
        comment="영역별 구체적 관찰 {'피부': '모공이 보임', '윤곽': 'V라인 필요'}"
    )
    
    confidence_score = Column(
        Numeric(3, 2),
        nullable=True,
        comment="분석 신뢰도 (0.00 ~ 1.00)"
    )
    
    # 메타데이터
    generated_at = Column(DateTime(timezone=True), server_default=func.now())
    
    # Indexes
    __table_args__ = (
        Index('ix_reports_job_id', 'job_id'),
    )
    
    # Relationships
    job = relationship("Job", back_populates="report")
    
    # Knowledge 매칭 (N:M)
    knowledge_mappings = relationship(
        "ReportKnowledgeItem",
        back_populates="report",
        cascade="all, delete-orphan"
    )
    
    def __repr__(self):
        return f"<Report(id={self.report_id}, job_id={self.job_id})>"
    
    def to_dict(self) -> dict:
        """딕셔너리 변환"""
        return {
            "report_id": str(self.report_id),
            "job_id": str(self.job_id),
            "analysis_summary": self.analysis_summary,
            "improvement_areas": self.improvement_areas,
            "specific_observations": self.specific_observations,
            "confidence_score": float(self.confidence_score) if self.confidence_score else None,
            "generated_at": self.generated_at.isoformat() if self.generated_at else None,
        }
```

---

## 🔍 주요 속성

### analysis_summary
- **타입**: Text
- **설명**: GPT-4o Vision의 전체 요약
- **예시**: "전반적으로 깔끔한 인상이나, 피부 관리와 윤곽 정리가 필요합니다."

### improvement_areas
- **타입**: JSONB (Array)
- **설명**: 개선이 필요한 영역 리스트
- **예시**: `["피부", "윤곽", "눈"]`

### specific_observations
- **타입**: JSONB (Object)
- **설명**: 영역별 구체적 관찰 사항
- **예시**:
  ```json
  {
    "피부": "모공이 보이고 피부톤이 고르지 않음",
    "윤곽": "V라인 정리가 필요함",
    "눈": "눈썹 정리가 필요함"
  }
  ```

### confidence_score
- **타입**: Numeric(3, 2)
- **설명**: 분석 신뢰도 (0.00 ~ 1.00)
- **예시**: 0.85

---

## 📊 시드 데이터 예시

```python
# 예시 리포트
report = Report(
    job_id=uuid.UUID("..."),
    analysis_summary="전반적으로 깔끔한 인상이나, 피부 관리와 윤곽 정리가 필요합니다.",
    improvement_areas=["피부", "윤곽", "눈"],
    specific_observations={
        "피부": "모공이 보이고 피부톤이 고르지 않음",
        "윤곽": "V라인 정리가 필요함",
        "눈": "눈썹 정리가 필요함"
    },
    confidence_score=0.85
)
```

---

## 🔍 쿼리 예시

### 1. Job으로 리포트 조회

```python
async def get_report_by_job(
    db: AsyncSession,
    job_id: uuid.UUID
) -> Optional[Report]:
    """
    Job ID로 리포트 조회
    
    Args:
        job_id: Job ID
    
    Returns:
        Report 또는 None
    """
    stmt = select(Report).where(
        Report.job_id == job_id
    ).options(
        joinedload(Report.knowledge_mappings).joinedload(ReportKnowledgeItem.knowledge_item)
    )
    
    result = await db.execute(stmt)
    return result.unique().scalar_one_or_none()
```

### 2. 개선 영역으로 검색

```python
async def search_by_improvement_area(
    db: AsyncSession,
    area: str
) -> List[Report]:
    """
    개선 영역으로 리포트 검색
    
    Args:
        area: 개선 영역 (예: '피부')
    
    Returns:
        해당 영역이 포함된 리포트 리스트
    """
    stmt = select(Report).where(
        Report.improvement_areas.contains([area])
    )
    
    result = await db.execute(stmt)
    return result.scalars().all()
```

---

## 📚 참고

- `docs/technical/langchain_architecture.md` - LangGraph Workflow
- 다음 문서: `report-knowledge-item.md` - Report ↔ Knowledge 매칭
