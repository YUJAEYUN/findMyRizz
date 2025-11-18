# KBProcedure 모델

## SQLAlchemy 모델 정의

```python
from sqlalchemy import Column, String, Text, ARRAY, TIMESTAMP, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.postgresql import ARRAY
from datetime import datetime

class KBProcedure(Base):
    __tablename__ = 'kb_procedures'
    
    # Primary Key
    id = Column(String(20), primary_key=True)  # P001, P002...
    
    # Foreign Key
    category_id = Column(String(20), ForeignKey('kb_categories.id'), nullable=False, index=True)
    
    # Basic Info
    name_ko = Column(String(100), nullable=False, index=True)
    name_en = Column(String(200), nullable=True)
    principle = Column(Text, nullable=True)
    
    # Timing
    effect_onset = Column(String(100), nullable=True)
    effect_duration = Column(String(100), nullable=True)
    recovery_period = Column(String(100), nullable=True)
    downtime = Column(String(50), nullable=True)
    
    # Effects & Areas
    main_effects = Column(ARRAY(Text), nullable=False)
    target_areas = Column(ARRAY(String(50)), nullable=False)
    
    # Procedure Details
    procedure_type = Column(String(100), nullable=True, index=True)
    procedure_method = Column(Text, nullable=True)
    
    # Side Effects & Precautions
    common_side_effects = Column(ARRAY(Text), nullable=True)
    rare_side_effects = Column(ARRAY(Text), nullable=True)
    precautions = Column(ARRAY(Text), nullable=True)
    contraindications = Column(ARRAY(Text), nullable=True)
    
    # Cost & Frequency
    cost_range = Column(String(100), nullable=True)
    repeat_cycle = Column(String(100), nullable=True)
    session_count = Column(String(50), nullable=True)
    cumulative_effect = Column(String(20), nullable=True)
    
    # Restrictions
    skin_tone_restriction = Column(String(50), nullable=True)
    age_restriction = Column(String(50), nullable=True)
    pregnancy_safe = Column(String(50), nullable=True)
    fda_approved = Column(String(20), nullable=True)
    
    # Pros & Cons
    pros = Column(ARRAY(Text), nullable=True)
    cons = Column(ARRAY(Text), nullable=True)
    treatment_target = Column(Text, nullable=True)
    
    # Timestamps
    created_at = Column(TIMESTAMP, nullable=False, default=datetime.utcnow)
    updated_at = Column(TIMESTAMP, nullable=False, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Relationships
    category = relationship("KBCategory", back_populates="procedures")
    report_items = relationship("ReportKnowledgeItem", back_populates="procedure")
```

---

## 테이블 생성 SQL

```sql
CREATE TABLE kb_procedures (
    id VARCHAR(20) PRIMARY KEY,
    category_id VARCHAR(20) NOT NULL REFERENCES kb_categories(id),
    name_ko VARCHAR(100) NOT NULL,
    name_en VARCHAR(200),
    principle TEXT,
    effect_onset VARCHAR(100),
    effect_duration VARCHAR(100),
    recovery_period VARCHAR(100),
    downtime VARCHAR(50),
    main_effects TEXT[] NOT NULL,
    target_areas VARCHAR(50)[] NOT NULL,
    procedure_type VARCHAR(100),
    procedure_method TEXT,
    common_side_effects TEXT[],
    rare_side_effects TEXT[],
    precautions TEXT[],
    contraindications TEXT[],
    cost_range VARCHAR(100),
    repeat_cycle VARCHAR(100),
    session_count VARCHAR(50),
    cumulative_effect VARCHAR(20),
    skin_tone_restriction VARCHAR(50),
    age_restriction VARCHAR(50),
    pregnancy_safe VARCHAR(50),
    fda_approved VARCHAR(20),
    pros TEXT[],
    cons TEXT[],
    treatment_target TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_kb_procedures_category_id ON kb_procedures(category_id);
CREATE INDEX idx_kb_procedures_name_ko ON kb_procedures(name_ko);
CREATE INDEX idx_kb_procedures_procedure_type ON kb_procedures(procedure_type);
```

---

## CSV 임포트 스크립트

```python
# scripts/import_procedures.py
import csv
from sqlalchemy.orm import Session
from models import KBProcedure

CATEGORY_MAPPING = {
    "신경독소 주사": "CAT001",
    "충전제 주사": "CAT002",
    "생체활성제 주사": "CAT002",
    "보습 주사": "CAT002",
    "레이저 시술": "CAT003",
    "고주파 리프팅": "CAT004",
    "고주파 지방 용해": "CAT004",
    "초음파 리프팅": "CAT005",
    "광선 치료": "CAT006",
    "광치료": "CAT006",
    "광역학 치료": "CAT006",
    "미세침 시술": "CAT007",
    "박피술": "CAT008",
    "실리프팅": "CAT009",
    "제모 시술": "CAT010",
}

def import_procedures(db: Session, csv_path: str):
    with open(csv_path, 'r', encoding='utf-8') as f:
        reader = csv.DictReader(f)
        procedures = []
        
        for row in reader:
            procedure = KBProcedure(
                id=row['절차_ID'],
                category_id=CATEGORY_MAPPING[row['카테고리']],
                name_ko=row['시술_이름'],
                name_en=row['영문_이명'] or None,
                principle=row['작용_원리'] or None,
                effect_onset=row['효과_발현_시간'] or None,
                main_effects=row['주요_효과'].split(', ') if row['주요_효과'] else [],
                target_areas=row['사용_부위'].split(', ') if row['사용_부위'] else [],
                procedure_type=row['시술_유형'] or None,
                procedure_method=row['시술_방법'] or None,
                recovery_period=row['회복_기간'] or None,
                effect_duration=row['효과_지속기간'] or None,
                downtime=row['다운타임'] or None,
                common_side_effects=row['흔한_부작용'].split(', ') if row['흔한_부작용'] else None,
                rare_side_effects=row['드운_부작용'].split(', ') if row['드운_부작용'] else None,
                precautions=row['주의사항'].split(', ') if row['주의사항'] else None,
                contraindications=row['금기사항'].split(', ') if row['금기사항'] else None,
                cost_range=row['비용_범위'] or None,
                repeat_cycle=row['권장_반복_주기'] or None,
                session_count=row['시술_횟수'] or None,
                cumulative_effect=row['효과_누적'] or None,
                skin_tone_restriction=row['피부톤_제약'] or None,
                age_restriction=row['나이_제약'] or None,
                pregnancy_safe=row['임신_중_안전성'] or None,
                fda_approved=row['FDA_승인'] or None,
                pros=row['장점'].split(', ') if row['장점'] else None,
                cons=row['단점'].split(', ') if row['단점'] else None,
                treatment_target=row['치료_대상'] or None,
            )
            procedures.append(procedure)
        
        db.bulk_save_objects(procedures)
        db.commit()
        
        print(f"✅ Imported {len(procedures)} procedures")

# 실행
if __name__ == "__main__":
    from database import SessionLocal
    db = SessionLocal()
    try:
        import_procedures(db, "시술정보_데이터셋.csv")
    finally:
        db.close()
```

---

## 사용 예시

### 조회

```python
# 모든 시술 조회
procedures = db.query(KBProcedure).all()

# 카테고리별 조회
procedures = db.query(KBProcedure).filter(
    KBProcedure.category_id == "CAT001"
).all()

# 이름으로 검색
procedures = db.query(KBProcedure).filter(
    KBProcedure.name_ko.like('%보톡스%')
).all()

# 시술 유형별 조회
procedures = db.query(KBProcedure).filter(
    KBProcedure.procedure_type == "주사형 비침습적"
).all()
```

### JOIN 조회

```python
# 카테고리 정보와 함께 조회
from models import KBCategory

results = db.query(KBProcedure, KBCategory).join(
    KBCategory, KBProcedure.category_id == KBCategory.id
).all()

for procedure, category in results:
    print(f"{procedure.name_ko} - {category.name}")
```

---

## 참고

- 스키마 문서: `docs/technical/schema_kb.md`
- 관련 모델: `kb-category.md`, `report-knowledge-item.md`
- CSV 파일: `시술정보_데이터셋.csv`

