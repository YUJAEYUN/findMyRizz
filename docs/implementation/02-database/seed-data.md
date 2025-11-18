# Knowledge Base 시드 데이터

## 📋 목표

Knowledge Base에 초기 데이터를 삽입합니다.

---

## 🔧 구현

### 파일: `scripts/seed_knowledge.py`

```python
"""
Knowledge Base 시드 데이터 삽입
"""
from sqlalchemy.orm import Session
import uuid
import sys
import os

# 프로젝트 루트를 Python 경로에 추가
sys.path.append(os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from app.core.database import SessionLocal
from app.models.knowledge import KnowledgeCategory, KnowledgeItem


def seed_knowledge_base():
    """Knowledge Base 시드 데이터 삽입"""
    db = SessionLocal()
    
    try:
        print("[Seed] Starting Knowledge Base seeding...")
        
        # 기존 데이터 확인
        existing_count = db.query(KnowledgeCategory).count()
        if existing_count > 0:
            print(f"[Seed] Already seeded ({existing_count} categories)")
            return
        
        # 카테고리 및 아이템 데이터
        categories_data = [
            {
                "name": "피부 관리",
                "description": "피부 톤 및 질감 개선",
                "display_order": 1,
                "items": [
                    {
                        "title": "비타민 C 세럼",
                        "description": "피부 톤을 밝게 하고 잡티를 개선하는 비타민 C 세럼을 매일 사용하세요.",
                        "procedure_type": "cosmetic",
                        "estimated_cost": "3만원~10만원",
                        "display_order": 1
                    },
                    {
                        "title": "레이저 토닝",
                        "description": "피부 톤을 균일하게 만들고 잡티를 제거하는 레이저 시술입니다.",
                        "procedure_type": "medical",
                        "estimated_cost": "10만원~30만원/회",
                        "display_order": 2
                    }
                ]
            },
            {
                "name": "윤곽 관리",
                "description": "얼굴 윤곽 및 라인 개선",
                "display_order": 2,
                "items": [
                    {
                        "title": "윤곽 메이크업",
                        "description": "쉐이딩과 하이라이터를 활용한 윤곽 메이크업 기법을 익히세요.",
                        "procedure_type": "cosmetic",
                        "estimated_cost": "5만원~15만원 (제품)",
                        "display_order": 1
                    },
                    {
                        "title": "보톡스",
                        "description": "사각턱 보톡스로 얼굴 라인을 부드럽게 만들 수 있습니다.",
                        "procedure_type": "medical",
                        "estimated_cost": "15만원~40만원",
                        "display_order": 2
                    }
                ]
            },
            {
                "name": "눈 관리",
                "description": "눈매 및 눈 주변 개선",
                "display_order": 3,
                "items": [
                    {
                        "title": "아이라인 기법",
                        "description": "눈매를 또렷하게 만드는 아이라인 그리기 기법을 연습하세요.",
                        "procedure_type": "cosmetic",
                        "estimated_cost": "1만원~5만원 (제품)",
                        "display_order": 1
                    },
                    {
                        "title": "눈 주변 관리",
                        "description": "아이크림을 사용하여 다크서클과 주름을 개선하세요.",
                        "procedure_type": "cosmetic",
                        "estimated_cost": "3만원~10만원",
                        "display_order": 2
                    }
                ]
            },
            {
                "name": "코 관리",
                "description": "코 라인 및 형태 개선",
                "display_order": 4,
                "items": [
                    {
                        "title": "노즈 쉐도우",
                        "description": "노즈 쉐도우로 코를 더 높고 오똑하게 보이게 할 수 있습니다.",
                        "procedure_type": "cosmetic",
                        "estimated_cost": "2만원~8만원 (제품)",
                        "display_order": 1
                    }
                ]
            },
            {
                "name": "입술 관리",
                "description": "입술 볼륨 및 색상 개선",
                "display_order": 5,
                "items": [
                    {
                        "title": "립 케어",
                        "description": "립밤과 립스크럽으로 입술을 촉촉하고 부드럽게 유지하세요.",
                        "procedure_type": "cosmetic",
                        "estimated_cost": "1만원~3만원",
                        "display_order": 1
                    },
                    {
                        "title": "립 메이크업",
                        "description": "립라이너와 립스틱으로 입술 볼륨을 강조하세요.",
                        "procedure_type": "cosmetic",
                        "estimated_cost": "2만원~5만원 (제품)",
                        "display_order": 2
                    }
                ]
            }
        ]
        
        # 데이터 삽입
        for cat_data in categories_data:
            # 카테고리 생성
            category = KnowledgeCategory(
                id=str(uuid.uuid4()),
                name=cat_data["name"],
                description=cat_data["description"],
                display_order=cat_data["display_order"]
            )
            db.add(category)
            db.flush()  # ID 생성
            
            print(f"[Seed] Created category: {category.name}")
            
            # 아이템 생성
            for item_data in cat_data["items"]:
                item = KnowledgeItem(
                    id=str(uuid.uuid4()),
                    category_id=category.id,
                    title=item_data["title"],
                    description=item_data["description"],
                    procedure_type=item_data["procedure_type"],
                    estimated_cost=item_data["estimated_cost"],
                    display_order=item_data["display_order"]
                )
                db.add(item)
                print(f"[Seed]   - Created item: {item.title}")
        
        db.commit()
        print("[Seed] Knowledge Base seeding completed!")
        
    except Exception as e:
        print(f"[Seed] Error: {e}")
        db.rollback()
    finally:
        db.close()


if __name__ == "__main__":
    seed_knowledge_base()
```

---

## 🔧 실행 방법

### 로컬 실행

```bash
python scripts/seed_knowledge.py
```

### Docker에서 실행

```bash
docker-compose exec api python scripts/seed_knowledge.py
```

---

## 🔍 시드 데이터 구조

```
피부 관리
├── 비타민 C 세럼 (cosmetic, 3만원~10만원)
└── 레이저 토닝 (medical, 10만원~30만원/회)

윤곽 관리
├── 윤곽 메이크업 (cosmetic, 5만원~15만원)
└── 보톡스 (medical, 15만원~40만원)

눈 관리
├── 아이라인 기법 (cosmetic, 1만원~5만원)
└── 눈 주변 관리 (cosmetic, 3만원~10만원)

코 관리
└── 노즈 쉐도우 (cosmetic, 2만원~8만원)

입술 관리
├── 립 케어 (cosmetic, 1만원~3만원)
└── 립 메이크업 (cosmetic, 2만원~5만원)
```

---

## ✅ 확인

```python
# Python 쉘에서 확인
from app.core.database import SessionLocal
from app.models.knowledge import KnowledgeCategory

db = SessionLocal()

categories = db.query(KnowledgeCategory).all()
for cat in categories:
    print(f"{cat.name}: {len(cat.items)} items")
```

---

## 📝 체크리스트

- [ ] scripts/seed_knowledge.py 생성
- [ ] 카테고리 데이터 정의
- [ ] 아이템 데이터 정의
- [ ] 실행 테스트
- [ ] 데이터 확인

---

## 🚀 다음 단계

시드 데이터 완료 → **Phase 3: Payment**

