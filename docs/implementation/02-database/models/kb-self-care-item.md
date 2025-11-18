# KBSelfCareItem 모델

## SQLAlchemy 모델 정의

```python
from sqlalchemy import Column, String, Text, ARRAY, TIMESTAMP, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.postgresql import ARRAY, JSONB
from datetime import datetime

class KBSelfCareItem(Base):
    __tablename__ = 'kb_self_care_items'

    # Primary Key
    id = Column(String(20), primary_key=True)  # SC001, SC002...

    # Foreign Key
    category_id = Column(String(20), ForeignKey('kb_categories.id'), nullable=False, index=True)

    # Basic Info
    name_ko = Column(String(100), nullable=False, index=True)
    name_en = Column(String(200), nullable=True)
    goal = Column(Text, nullable=True)
    method = Column(Text, nullable=True)

    # Timing
    effect_onset = Column(String(100), nullable=True)
    effect_duration = Column(String(100), nullable=True)
    downtime = Column(String(50), nullable=True)
    recovery_period = Column(String(100), nullable=True)

    # Effects & Areas
    effects = Column(ARRAY(Text), nullable=False)
    target_areas = Column(ARRAY(String(50)), nullable=False)

    # Side Effects & Precautions
    common_side_effects = Column(ARRAY(Text), nullable=True)
    serious_side_effects = Column(ARRAY(Text), nullable=True)
    precautions = Column(ARRAY(Text), nullable=True)
    contraindications = Column(ARRAY(Text), nullable=True)

    # Cost & Frequency
    cost_range = Column(String(100), nullable=True)
    repeat_cycle = Column(String(100), nullable=True)
    monthly_frequency = Column(String(50), nullable=True)
    cumulative_effect = Column(String(20), nullable=True)

    # Restrictions
    skin_tone_suitability = Column(String(100), nullable=True)
    age_restriction = Column(String(50), nullable=True)
    pregnancy_safe = Column(String(50), nullable=True)
    male_applicable = Column(String(50), nullable=True)

    # Pros & Cons
    pros = Column(ARRAY(Text), nullable=True)
    cons = Column(ARRAY(Text), nullable=True)
    optimal_user = Column(Text, nullable=True)
    additional_tips = Column(ARRAY(Text), nullable=True)

    # Product Info
    product_recommendations = Column(Text, nullable=True)

    # Duration
    duration = Column(String(100), nullable=True)
    duration_period = Column(String(100), nullable=True)
    monthly_activity = Column(String(100), nullable=True)

    # Additional Fields
    english_alias = Column(String(200), nullable=True)
    procedure_type = Column(String(100), nullable=True)
    session_count = Column(String(50), nullable=True)

    # JSONB for flexible data
    details = Column(JSONB, nullable=True)

    # Timestamps
    created_at = Column(TIMESTAMP, nullable=False, default=datetime.utcnow)
    updated_at = Column(TIMESTAMP, nullable=False, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    category = relationship("KBCategory", back_populates="self_care_items")
    report_items = relationship("ReportKnowledgeItem", back_populates="self_care_item")
```

---

## 테이블 생성 SQL

```sql
CREATE TABLE kb_self_care_items (
    id VARCHAR(20) PRIMARY KEY,
    category_id VARCHAR(20) NOT NULL REFERENCES kb_categories(id),
    name_ko VARCHAR(100) NOT NULL,
    name_en VARCHAR(200),
    goal TEXT,
    method TEXT,
    effect_onset VARCHAR(100),
    effect_duration VARCHAR(100),
    downtime VARCHAR(50),
    recovery_period VARCHAR(100),
    effects TEXT[] NOT NULL,
    target_areas VARCHAR(50)[] NOT NULL,
    common_side_effects TEXT[],
    serious_side_effects TEXT[],
    precautions TEXT[],
    contraindications TEXT[],
    cost_range VARCHAR(100),
    repeat_cycle VARCHAR(100),
    monthly_frequency VARCHAR(50),
    cumulative_effect VARCHAR(20),
    skin_tone_suitability VARCHAR(100),
    age_restriction VARCHAR(50),
    pregnancy_safe VARCHAR(50),
    male_applicable VARCHAR(50),
    pros TEXT[],
    cons TEXT[],
    optimal_user TEXT,
    additional_tips TEXT[],
    product_recommendations TEXT,
    duration VARCHAR(100),
    duration_period VARCHAR(100),
    monthly_activity VARCHAR(100),
    english_alias VARCHAR(200),
    procedure_type VARCHAR(100),
    session_count VARCHAR(50),
    details JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_kb_self_care_items_category_id ON kb_self_care_items(category_id);
CREATE INDEX idx_kb_self_care_items_name_ko ON kb_self_care_items(name_ko);
```

---

## details JSONB 구조

```python
# SC001 (눈썹 정리) 예시
details = {
    "steps": {
        "단계1": "트리머로 긴 눈썹 정리 (방향: 眉毛 성장 방향)",
        "단계2": "핀셋으로 원하지 않는 모발 제거",
        "단계3": "면도기로 미세한 잔털 정리",
        "단계4": "아이브로우 펜슬로 완성 (선택사항)"
    }
}

# SC002 (파운데이션) 예시
details = {
    "steps": {
        "단계1": "프라이머 도포 (선택사항)",
        "단계2": "파운데이션을 얼굴 중앙에 점으로 도포",
        "단계3": "스펀지 또는 브러시로 펴바르기",
        "단계4": "필요시 컨실러로 결점 보정",
        "단계5": "파우더로 설정"
    },
    "product_selection_guide": {
        "건성피부": "수분크림 파운데이션",
        "지성피부": "매트 파운데이션",
        "복합피부": "쿠션 또는 BB크림"
    }
}

# SC003 (썬크림) 예시
details = {
    "steps": {
        "단계1": "외출 15분 전 충분한 양을 얼굴에 도포",
        "단계2": "목과 팔에도 얇게 펴바르기",
        "단계3": "2-3시간마다 재도포 (특히 야외활동)",
        "단계4": "수분 또는 땀난 후 즉시 재도포"
    },
    "product_selection_guide": {
        "건성피부": "수분/영양 썬크림",
        "지성피부": "매트 또는 논-코메도제닉 썬크림",
        "민감피부": "광물 썬크림 (산화아연/이산화티탄)",
        "여드름피부": "가벼운 제형 + 올바른 클렌징"
    },
    "spf_guide": {
        "SPF30": "일상생활 (UV차단율 97%)",
        "SPF50": "야외활동/비치 (UV차단율 98%)",
        "SPF50+": "극강한 햇빛/수상활동"
    }
}

# SC005 (다이어트) 예시
details = {
    "steps": {
        "단계1_식단": "기초대사량 계산 후 500-750kcal 결핍",
        "단계2_식단": "단백질 섭취 증가 (근손실 방지)",
        "단계3_식단": "저탄수화물/고지방 또는 저지방/고탄수 선택",
        "단계4_운동": "유산소 운동 (주 3회, 30-45분)",
        "단계5_운동": "근력운동 (주 3회, 30분)"
    },
    "diet_guide": {
        "아침": "단백질+탄수 (계란, 오트밀)",
        "점심": "닭가슴살/생선+채소",
        "저녁": "채소/샐러드+(선택) 저단백질",
        "간식": "프로틴 요거트/무염견과류"
    },
    "exercise_guide": {
        "유산소": "조깅/자전거/수영 (30-45분, 주3회)",
        "근력": "스쿼트/푸시업/플랭크 (30분, 주3회)",
        "유연성": "요가/스트레칭 (15분, 주2회)"
    },
    "result_criteria": {
        "1주": "0.5-1kg 감량, 붓기 감소",
        "4주": "2-3kg 감량, 얼굴 라인 약간 정의",
        "8주": "4-5kg 감량, 턱선 명확해짐",
        "12주": "5kg 이상 감량, 얼굴 라인 극적 개선"
    }
}

# SC006 (눈썹 문신) 예시
details = {
    "steps": {
        "상담": "원하는 눈썹 모양 디자인 및 색상 결정",
        "마킹": "눈썹 모양을 원하는 대로 표시",
        "마취": "표피 마취제 도포",
        "시술": "세미 바늘로 피부에 안료 삽입 (30-60분)",
        "터치업": "1-2주 후 터치업 (필수)"
    },
    "color_selection": {
        "밝은피부": "갈색/밝은 갈색",
        "중간피부": "갈색/초콜릿",
        "어두운피부": "초콜릿/검정색"
    },
    "clinic_selection": [
        "자격 있는 전문가 확인",
        "위생 관리 엄격한 곳",
        "사후 관리 체계 있는 곳",
        "후기/평판 확인"
    ]
}
```

---

## CSV 임포트 스크립트

```python
# scripts/import_self_care_items.py
import csv
import json
import ast
from sqlalchemy.orm import Session
from models import KBSelfCareItem

CATEGORY_MAPPING = {
    "기초 미용 관리": "CAT011",
    "데일리 메이크업": "CAT012",
    "기초 피부 관리": "CAT013",
    "신체 자기관리": "CAT014",
    "신체 자기관리 - 전신 개선": "CAT014",
    "반영구 미용": "CAT015",
    "반영구 미용 시술": "CAT015",
}

def parse_list_field(value: str) -> list:
    """Parse list-like string from CSV"""
    if not value or value.strip() == '':
        return None
    try:
        # Try to parse as Python literal (e.g., "['item1', 'item2']")
        parsed = ast.literal_eval(value)
        if isinstance(parsed, list):
            return parsed
    except:
        pass
    # Fallback: split by comma
    return [item.strip() for item in value.split(',')]

def parse_dict_field(value: str) -> dict:
    """Parse dict-like string from CSV"""
    if not value or value.strip() == '':
        return None
    try:
        # Try to parse as Python literal (e.g., "{'key': 'value'}")
        parsed = ast.literal_eval(value)
        if isinstance(parsed, dict):
            return parsed
    except:
        pass
    return None

def import_self_care_items(db: Session, csv_path: str):
    with open(csv_path, 'r', encoding='utf-8') as f:
        reader = csv.DictReader(f)
        items = []

        for row in reader:
            # Build details JSONB
            details = {}

            # Parse steps
            if row.get('상세_절차'):
                details['steps'] = parse_dict_field(row['상세_절차'])

            # Parse product selection guide
            if row.get('제품_선택_가이드'):
                details['product_selection_guide'] = parse_dict_field(row['제품_선택_가이드'])

            # Parse SPF guide
            if row.get('SPF_가이드'):
                details['spf_guide'] = parse_dict_field(row['SPF_가이드'])

            # Parse diet guide
            if row.get('식단_가이드'):
                details['diet_guide'] = parse_dict_field(row['식단_가이드'])

            # Parse exercise guide
            if row.get('운동_가이드'):
                details['exercise_guide'] = parse_dict_field(row['운동_가이드'])

            # Parse result criteria
            if row.get('결과_기준'):
                details['result_criteria'] = parse_dict_field(row['결과_기준'])

            # Parse color selection
            if row.get('색상_선택'):
                details['color_selection'] = parse_dict_field(row['색상_선택'])

            # Parse clinic selection
            if row.get('시술소_선택'):
                details['clinic_selection'] = parse_list_field(row['시술소_선택'])

            item = KBSelfCareItem(
                id=row['항목_ID'],
                category_id=CATEGORY_MAPPING[row['카테고리']],
                name_ko=row['항목_이름'],
                name_en=row['영문_이름'] or None,
                goal=row['목표'] or None,
                effects=parse_list_field(row['효과']) or [],
                target_areas=parse_list_field(row['영향_부위']) or [],
                method=row['방법'] or None,
                effect_onset=row['효과_발현'] or None,
                effect_duration=row['효과_지속'] or None,
                downtime=row['다운타임'] or None,
                recovery_period=row['회복기간'] or None,
                common_side_effects=parse_list_field(row['흔한_부작용']),
                serious_side_effects=parse_list_field(row['심각_부작용']),
                precautions=parse_list_field(row['주의사항']),
                contraindications=parse_list_field(row['금기사항']),
                cost_range=row['비용'] or None,
                repeat_cycle=row['반복_주기'] or None,
                monthly_frequency=row.get('월_시술횟수') or None,
                cumulative_effect=row.get('효과_누적') or None,
                skin_tone_suitability=row.get('피부톤_적합성') or None,
                age_restriction=row.get('나이_제약') or None,
                pregnancy_safe=row.get('임신중_안전성') or None,
                male_applicable=row.get('남성적용') or None,
                pros=parse_list_field(row.get('장점')),
                cons=parse_list_field(row.get('단점')),
                optimal_user=row.get('최적_사용자') or None,
                additional_tips=parse_list_field(row.get('추가_팁')),
                product_recommendations=row.get('제품_추천') or None,
                duration=row.get('기간') or None,
                duration_period=row.get('지속_기간') or None,
                monthly_activity=row.get('월_활동') or None,
                english_alias=row.get('영문_이명') or row.get('영문_이름') or None,  # CSV에 영문_이명이 없으면 영문_이름 사용
                procedure_type=row.get('시술_유형') or None,
                session_count=row.get('횟수') or None,
                details=details if details else None,
            )
            items.append(item)

        db.bulk_save_objects(items)
        db.commit()

        print(f"✅ Imported {len(items)} self-care items")

# 실행
if __name__ == "__main__":
    from database import SessionLocal
    db = SessionLocal()
    try:
        import_self_care_items(db, "자기관리_데이터셋.csv")
    finally:
        db.close()
```

---

## 사용 예시

### 조회

```python
# 모든 자기관리 항목 조회
items = db.query(KBSelfCareItem).all()

# 카테고리별 조회
items = db.query(KBSelfCareItem).filter(
    KBSelfCareItem.category_id == "CAT011"
).all()

# 이름으로 검색
items = db.query(KBSelfCareItem).filter(
    KBSelfCareItem.name_ko.like('%눈썹%')
).all()
```

### JSONB 쿼리

```python
# steps가 있는 항목 조회
items = db.query(KBSelfCareItem).filter(
    KBSelfCareItem.details['steps'].astext.isnot(None)
).all()

# diet_guide가 있는 항목 조회
items = db.query(KBSelfCareItem).filter(
    KBSelfCareItem.details.has_key('diet_guide')
).all()
```

---

## 참고

- 스키마 문서: `docs/technical/schema_kb.md`
- 관련 모델: `kb-category.md`, `report-knowledge-item.md`
- CSV 파일: `자기관리_데이터셋.csv`
