# ReportKnowledgeItem 모델

## SQLAlchemy 모델 정의

```python
from sqlalchemy import Column, String, Integer, ForeignKey, Text, Numeric, TIMESTAMP, Index, UniqueConstraint
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.postgresql import UUID
from datetime import datetime

class ReportKnowledgeItem(Base):
    __tablename__ = "report_knowledge_items"

    # Primary Key
    mapping_id = Column(Integer, primary_key=True, autoincrement=True)

    # Foreign Keys
    report_id = Column(
        UUID(as_uuid=True),
        ForeignKey("reports.report_id", ondelete="CASCADE"),
        nullable=False,
        index=True
    )

    # Knowledge Item Reference (Polymorphic)
    item_id = Column(String(20), nullable=False, index=True)
    item_type = Column(String(20), nullable=False)  # 'PROCEDURE' or 'SELF_CARE'

    # Matching Info
    relevance_score = Column(Numeric(3, 2), nullable=False)
    match_reason = Column(Text, nullable=True)
    display_order = Column(Integer, nullable=True)

    # Timestamps
    matched_at = Column(TIMESTAMP, nullable=False, default=datetime.utcnow)

    # Constraints
    __table_args__ = (
        UniqueConstraint('report_id', 'item_id', name='uq_report_knowledge_items'),
        Index('ix_report_knowledge_report', 'report_id'),
        Index('ix_report_knowledge_item', 'item_id'),
        Index('ix_report_knowledge_item_type', 'item_type'),
    )

    # Relationships
    report = relationship("Report", back_populates="knowledge_mappings")

    # Polymorphic relationships (manual)
    # Use item_id + item_type to query kb_procedures or kb_self_care_items
```

---

## 테이블 생성 SQL

```sql
CREATE TABLE report_knowledge_items (
    mapping_id SERIAL PRIMARY KEY,
    report_id UUID NOT NULL REFERENCES reports(report_id) ON DELETE CASCADE,
    item_id VARCHAR(20) NOT NULL,
    item_type VARCHAR(20) NOT NULL,
    relevance_score NUMERIC(3, 2) NOT NULL,
    match_reason TEXT,
    display_order INTEGER,
    matched_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uq_report_knowledge_items UNIQUE (report_id, item_id)
);

CREATE INDEX ix_report_knowledge_report ON report_knowledge_items(report_id);
CREATE INDEX ix_report_knowledge_item ON report_knowledge_items(item_id);
CREATE INDEX ix_report_knowledge_item_type ON report_knowledge_items(item_type);
```

---

## 사용 예시

### 매칭 생성

```python
# LangGraph match_knowledge_node 결과 저장
mappings = [
    ReportKnowledgeItem(
        report_id=uuid.UUID("..."),
        item_id="P001",  # 보톡스
        item_type="PROCEDURE",
        relevance_score=0.85,
        match_reason="이마 주름 개선에 효과적",
        display_order=1
    ),
    ReportKnowledgeItem(
        report_id=uuid.UUID("..."),
        item_id="SC001",  # 눈썹 정리
        item_type="SELF_CARE",
        relevance_score=0.92,
        match_reason="눈썹 정리로 인상 개선 가능",
        display_order=2
    ),
    ReportKnowledgeItem(
        report_id=uuid.UUID("..."),
        item_id="SC003",  # 썬크림
        item_type="SELF_CARE",
        relevance_score=0.78,
        match_reason="피부톤 개선을 위한 기초 관리",
        display_order=3
    ),
]

db.bulk_save_objects(mappings)
db.commit()
```

### 조회 (Polymorphic)

```python
# 리포트의 모든 매칭 조회
mappings = db.query(ReportKnowledgeItem).filter(
    ReportKnowledgeItem.report_id == report_id
).order_by(ReportKnowledgeItem.display_order).all()

# 타입별 조회
procedure_mappings = db.query(ReportKnowledgeItem).filter(
    ReportKnowledgeItem.report_id == report_id,
    ReportKnowledgeItem.item_type == "PROCEDURE"
).all()

self_care_mappings = db.query(ReportKnowledgeItem).filter(
    ReportKnowledgeItem.report_id == report_id,
    ReportKnowledgeItem.item_type == "SELF_CARE"
).all()
```

### 실제 Knowledge 데이터 조회

```python
# Step 1: 매칭 조회
mappings = db.query(ReportKnowledgeItem).filter(
    ReportKnowledgeItem.report_id == report_id
).all()

# Step 2: 타입별로 실제 데이터 조회
for mapping in mappings:
    if mapping.item_type == "PROCEDURE":
        procedure = db.query(KBProcedure).filter(
            KBProcedure.id == mapping.item_id
        ).first()
        print(f"{procedure.name_ko}: {mapping.relevance_score}")

    elif mapping.item_type == "SELF_CARE":
        self_care = db.query(KBSelfCareItem).filter(
            KBSelfCareItem.id == mapping.item_id
        ).first()
        print(f"{self_care.name_ko}: {mapping.relevance_score}")
```

### 고득점 매칭 조회

```python
# 상위 5개 매칭
top_mappings = db.query(ReportKnowledgeItem).filter(
    ReportKnowledgeItem.report_id == report_id
).order_by(
    ReportKnowledgeItem.relevance_score.desc()
).limit(5).all()
```

---

## 참고

- 스키마 문서: `docs/technical/schema_kb.md`
- LangGraph 아키텍처: `docs/technical/langchain_architecture.md`
- 관련 모델: `report.md`, `kb-procedure.md`, `kb-self-care-item.md`
