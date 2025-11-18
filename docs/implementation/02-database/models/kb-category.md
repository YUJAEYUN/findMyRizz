# KBCategory 모델

## SQLAlchemy 모델 정의

```python
from sqlalchemy import Column, String, Enum, TIMESTAMP
from sqlalchemy.orm import relationship
from datetime import datetime
import enum

class ItemType(enum.Enum):
    PROCEDURE = "PROCEDURE"
    SELF_CARE = "SELF_CARE"

class KBCategory(Base):
    __tablename__ = 'kb_categories'
    
    # Primary Key
    id = Column(String(20), primary_key=True)  # CAT001, CAT002...
    
    # Attributes
    name = Column(String(50), nullable=False, unique=True, index=True)
    item_type = Column(Enum(ItemType), nullable=False, index=True)
    description = Column(String(200), nullable=True)
    
    # Timestamps
    created_at = Column(TIMESTAMP, nullable=False, default=datetime.utcnow)
    updated_at = Column(TIMESTAMP, nullable=False, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Relationships
    procedures = relationship("KBProcedure", back_populates="category", cascade="all, delete-orphan")
    self_care_items = relationship("KBSelfCareItem", back_populates="category", cascade="all, delete-orphan")
```

---

## 테이블 생성 SQL

```sql
CREATE TYPE item_type AS ENUM ('PROCEDURE', 'SELF_CARE');

CREATE TABLE kb_categories (
    id VARCHAR(20) PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    item_type item_type NOT NULL,
    description VARCHAR(200),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_kb_categories_name ON kb_categories(name);
CREATE INDEX idx_kb_categories_type ON kb_categories(item_type);
```

---

## 시드 데이터

```python
# scripts/seed_kb_categories.py
from sqlalchemy.orm import Session
from models import KBCategory, ItemType

def seed_kb_categories(db: Session):
    categories = [
        # 시술정보 (10 rows)
        KBCategory(id="CAT001", name="신경독소 주사", item_type=ItemType.PROCEDURE),
        KBCategory(id="CAT002", name="충전제 주사", item_type=ItemType.PROCEDURE),
        KBCategory(id="CAT003", name="레이저 시술", item_type=ItemType.PROCEDURE),
        KBCategory(id="CAT004", name="고주파 리프팅", item_type=ItemType.PROCEDURE),
        KBCategory(id="CAT005", name="초음파 리프팅", item_type=ItemType.PROCEDURE),
        KBCategory(id="CAT006", name="광선 치료", item_type=ItemType.PROCEDURE),
        KBCategory(id="CAT007", name="미세침 시술", item_type=ItemType.PROCEDURE),
        KBCategory(id="CAT008", name="박피술", item_type=ItemType.PROCEDURE),
        KBCategory(id="CAT009", name="실리프팅", item_type=ItemType.PROCEDURE),
        KBCategory(id="CAT010", name="제모 시술", item_type=ItemType.PROCEDURE),
        
        # 자기관리 (5 rows)
        KBCategory(id="CAT011", name="기초 미용 관리", item_type=ItemType.SELF_CARE),
        KBCategory(id="CAT012", name="데일리 메이크업", item_type=ItemType.SELF_CARE),
        KBCategory(id="CAT013", name="기초 피부 관리", item_type=ItemType.SELF_CARE),
        KBCategory(id="CAT014", name="신체 자기관리", item_type=ItemType.SELF_CARE),
        KBCategory(id="CAT015", name="반영구 미용", item_type=ItemType.SELF_CARE),
    ]
    
    db.bulk_save_objects(categories)
    db.commit()
    
    print(f"✅ Seeded {len(categories)} categories")
```

---

## 사용 예시

### 조회

```python
# 모든 카테고리 조회
categories = db.query(KBCategory).all()

# 타입별 조회
procedure_categories = db.query(KBCategory).filter(
    KBCategory.item_type == ItemType.PROCEDURE
).all()

# 이름으로 조회
category = db.query(KBCategory).filter(
    KBCategory.name == "신경독소 주사"
).first()

# ID로 조회
category = db.query(KBCategory).filter(
    KBCategory.id == "CAT001"
).first()
```

### 통계

```python
# 타입별 카테고리 수
from sqlalchemy import func

stats = db.query(
    KBCategory.item_type,
    func.count(KBCategory.id).label('count')
).group_by(KBCategory.item_type).all()

# 결과: [('PROCEDURE', 10), ('SELF_CARE', 5)]
```

### 관계 조회

```python
# 카테고리의 모든 시술 조회
category = db.query(KBCategory).filter(KBCategory.id == "CAT001").first()
procedures = category.procedures  # KBProcedure 리스트

# 카테고리의 모든 자기관리 조회
category = db.query(KBCategory).filter(KBCategory.id == "CAT011").first()
self_care_items = category.self_care_items  # KBSelfCareItem 리스트
```

---

## 참고

- 스키마 문서: `docs/technical/schema_kb.md`
- 관련 모델: `kb-procedure.md`, `kb-self-care-item.md`

