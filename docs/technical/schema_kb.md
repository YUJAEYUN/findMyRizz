# Knowledge Base 데이터베이스 스키마

## 테이블 구조

```
kb_categories (15 rows)
├── kb_procedures (26 rows)
└── kb_self_care_items (6 rows)
```

**정규화 수준:**

- 제1정규화: 모든 속성이 원자값
- 제2정규화: category → item_type 함수 종속성 분리
- 타입별 테이블 분리: NULL 최소화, RDBMS 최대 활용

---

## 테이블 1: kb_categories

### 스키마

| 컬럼        | 타입         | 제약             | 설명                 |
| ----------- | ------------ | ---------------- | -------------------- |
| id          | VARCHAR(20)  | PK               | CAT001, CAT002...    |
| name        | VARCHAR(50)  | NOT NULL, UNIQUE | 카테고리명           |
| item_type   | ENUM         | NOT NULL         | PROCEDURE, SELF_CARE |
| description | VARCHAR(200) | NULL             | 설명                 |
| created_at  | TIMESTAMP    | NOT NULL         | 생성일시             |
| updated_at  | TIMESTAMP    | NOT NULL         | 수정일시             |

### 인덱스

```sql
CREATE INDEX idx_kb_categories_name ON kb_categories(name);
CREATE INDEX idx_kb_categories_type ON kb_categories(item_type);
```

### 초기 데이터 (15 rows)

```sql
-- 시술정보 (10 rows)
INSERT INTO kb_categories (id, name, item_type) VALUES
('CAT001', '신경독소 주사', 'PROCEDURE'),
('CAT002', '충전제 주사', 'PROCEDURE'),
('CAT003', '레이저 시술', 'PROCEDURE'),
('CAT004', '고주파 리프팅', 'PROCEDURE'),
('CAT005', '초음파 리프팅', 'PROCEDURE'),
('CAT006', '광선 치료', 'PROCEDURE'),
('CAT007', '미세침 시술', 'PROCEDURE'),
('CAT008', '박피술', 'PROCEDURE'),
('CAT009', '실리프팅', 'PROCEDURE'),
('CAT010', '제모 시술', 'PROCEDURE');

-- 자기관리 (5 rows)
INSERT INTO kb_categories (id, name, item_type) VALUES
('CAT011', '기초 미용 관리', 'SELF_CARE'),
('CAT012', '데일리 메이크업', 'SELF_CARE'),
('CAT013', '기초 피부 관리', 'SELF_CARE'),
('CAT014', '신체 자기관리', 'SELF_CARE'),
('CAT015', '반영구 미용', 'SELF_CARE');
```

---

## 테이블 2: kb_procedures

### 스키마 (28 columns)

| 컬럼                  | 타입          | 제약         | CSV 컬럼        |
| --------------------- | ------------- | ------------ | --------------- |
| id                    | VARCHAR(20)   | PK           | 절차\_ID        |
| category_id           | VARCHAR(20)   | FK, NOT NULL | 카테고리 (매핑) |
| name_ko               | VARCHAR(100)  | NOT NULL     | 시술\_이름      |
| name_en               | VARCHAR(200)  | NULL         | 영문\_이명      |
| principle             | TEXT          | NULL         | 작용\_원리      |
| effect_onset          | VARCHAR(100)  | NULL         | 효과*발현*시간  |
| main_effects          | TEXT[]        | NOT NULL     | 주요\_효과      |
| target_areas          | VARCHAR(50)[] | NOT NULL     | 사용\_부위      |
| procedure_type        | VARCHAR(100)  | NULL         | 시술\_유형      |
| procedure_method      | TEXT          | NULL         | 시술\_방법      |
| recovery_period       | VARCHAR(100)  | NULL         | 회복\_기간      |
| effect_duration       | VARCHAR(100)  | NULL         | 효과\_지속기간  |
| downtime              | VARCHAR(50)   | NULL         | 다운타임        |
| common_side_effects   | TEXT[]        | NULL         | 흔한\_부작용    |
| rare_side_effects     | TEXT[]        | NULL         | 드운\_부작용    |
| precautions           | TEXT[]        | NULL         | 주의사항        |
| contraindications     | TEXT[]        | NULL         | 금기사항        |
| cost_range            | VARCHAR(100)  | NULL         | 비용\_범위      |
| repeat_cycle          | VARCHAR(100)  | NULL         | 권장*반복*주기  |
| session_count         | VARCHAR(50)   | NULL         | 시술\_횟수      |
| cumulative_effect     | VARCHAR(20)   | NULL         | 효과\_누적      |
| skin_tone_restriction | VARCHAR(50)   | NULL         | 피부톤\_제약    |
| age_restriction       | VARCHAR(50)   | NULL         | 나이\_제약      |
| pregnancy_safe        | VARCHAR(50)   | NULL         | 임신*중*안전성  |
| fda_approved          | VARCHAR(20)   | NULL         | FDA\_승인       |
| pros                  | TEXT[]        | NULL         | 장점            |
| cons                  | TEXT[]        | NULL         | 단점            |
| treatment_target      | TEXT          | NULL         | 치료\_대상      |
| created_at            | TIMESTAMP     | NOT NULL     | -               |
| updated_at            | TIMESTAMP     | NOT NULL     | -               |

### 외래키

```sql
ALTER TABLE kb_procedures
ADD CONSTRAINT fk_kb_procedures_category
FOREIGN KEY (category_id) REFERENCES kb_categories(id);
```

### 인덱스

```sql
CREATE INDEX idx_kb_procedures_category_id ON kb_procedures(category_id);
CREATE INDEX idx_kb_procedures_name_ko ON kb_procedures(name_ko);
CREATE INDEX idx_kb_procedures_procedure_type ON kb_procedures(procedure_type);
```

---

## 테이블 3: kb_self_care_items

### 스키마 (35 columns + JSONB)

| 컬럼                    | 타입          | 제약         | CSV 컬럼        |
| ----------------------- | ------------- | ------------ | --------------- |
| id                      | VARCHAR(20)   | PK           | 항목\_ID        |
| category_id             | VARCHAR(20)   | FK, NOT NULL | 카테고리 (매핑) |
| name_ko                 | VARCHAR(100)  | NOT NULL     | 항목\_이름      |
| name_en                 | VARCHAR(200)  | NULL         | 영문\_이름      |
| goal                    | TEXT          | NULL         | 목표            |
| effects                 | TEXT[]        | NOT NULL     | 효과            |
| target_areas            | VARCHAR(50)[] | NOT NULL     | 영향\_부위      |
| method                  | TEXT          | NULL         | 방법            |
| effect_onset            | VARCHAR(100)  | NULL         | 효과\_발현      |
| effect_duration         | VARCHAR(100)  | NULL         | 효과\_지속      |
| downtime                | VARCHAR(50)   | NULL         | 다운타임        |
| recovery_period         | VARCHAR(100)  | NULL         | 회복기간        |
| common_side_effects     | TEXT[]        | NULL         | 흔한\_부작용    |
| serious_side_effects    | TEXT[]        | NULL         | 심각\_부작용    |
| precautions             | TEXT[]        | NULL         | 주의사항        |
| contraindications       | TEXT[]        | NULL         | 금기사항        |
| cost_range              | VARCHAR(100)  | NULL         | 비용            |
| repeat_cycle            | VARCHAR(100)  | NULL         | 반복\_주기      |
| monthly_frequency       | VARCHAR(50)   | NULL         | 월\_시술횟수    |
| cumulative_effect       | VARCHAR(20)   | NULL         | 효과\_누적      |
| skin_tone_suitability   | VARCHAR(100)  | NULL         | 피부톤\_적합성  |
| age_restriction         | VARCHAR(50)   | NULL         | 나이\_제약      |
| pregnancy_safe          | VARCHAR(50)   | NULL         | 임신중\_안전성  |
| male_applicable         | VARCHAR(50)   | NULL         | 남성적용        |
| pros                    | TEXT[]        | NULL         | 장점            |
| cons                    | TEXT[]        | NULL         | 단점            |
| optimal_user            | TEXT          | NULL         | 최적\_사용자    |
| additional_tips         | TEXT[]        | NULL         | 추가\_팁        |
| product_recommendations | TEXT          | NULL         | 제품\_추천      |
| duration                | VARCHAR(100)  | NULL         | 기간            |
| duration_period         | VARCHAR(100)  | NULL         | 지속\_기간      |
| monthly_activity        | VARCHAR(100)  | NULL         | 월\_활동        |
| english_alias           | VARCHAR(200)  | NULL         | 영문\_이명      |
| procedure_type          | VARCHAR(100)  | NULL         | 시술\_유형      |
| session_count           | VARCHAR(50)   | NULL         | 횟수            |
| details                 | JSONB         | NULL         | (비정형 데이터) |
| created_at              | TIMESTAMP     | NOT NULL     | -               |
| updated_at              | TIMESTAMP     | NOT NULL     | -               |

### 외래키

```sql
ALTER TABLE kb_self_care_items
ADD CONSTRAINT fk_kb_self_care_items_category
FOREIGN KEY (category_id) REFERENCES kb_categories(id);
```

### 인덱스

```sql
CREATE INDEX idx_kb_self_care_items_category_id ON kb_self_care_items(category_id);
CREATE INDEX idx_kb_self_care_items_name_ko ON kb_self_care_items(name_ko);
```

### details JSONB 구조

```json
{
  "steps": {
    "단계1": "트리머로 긴 눈썹 정리 (방향: 眉毛 성장 방향)",
    "단계2": "핀셋으로 원하지 않는 모발 제거",
    "단계3": "면도기로 미세한 잔털 정리",
    "단계4": "아이브로우 펜슬로 완성 (선택사항)"
  },
  "product_selection_guide": {
    "건성피부": "수분/영양 썬크림",
    "지성피부": "매트 또는 논-코메도제닉 썬크림",
    "민감피부": "광물 썬크림 (산화아연/이산화티탄)"
  },
  "spf_guide": {
    "SPF30": "일상생활 (UV차단율 97%)",
    "SPF50": "야외활동/비치 (UV차단율 98%)",
    "SPF50+": "극강한 햇빛/수상활동"
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

## CSV → DB 매핑

### 시술정보\_데이터셋.csv → kb_procedures

| CSV 컬럼       | DB 컬럼               | 변환        |
| -------------- | --------------------- | ----------- |
| 절차\_ID       | id                    | 그대로      |
| 카테고리       | category_id           | 매핑 (하단) |
| 시술\_이름     | name_ko               | 그대로      |
| 영문\_이명     | name_en               | 그대로      |
| 작용\_원리     | principle             | 그대로      |
| 효과*발현*시간 | effect_onset          | 그대로      |
| 주요\_효과     | main_effects          | split(', ') |
| 사용\_부위     | target_areas          | split(', ') |
| 시술\_유형     | procedure_type        | 그대로      |
| 시술\_방법     | procedure_method      | 그대로      |
| 회복\_기간     | recovery_period       | 그대로      |
| 효과\_지속기간 | effect_duration       | 그대로      |
| 다운타임       | downtime              | 그대로      |
| 흔한\_부작용   | common_side_effects   | split(', ') |
| 드운\_부작용   | rare_side_effects     | split(', ') |
| 주의사항       | precautions           | split(', ') |
| 금기사항       | contraindications     | split(', ') |
| 비용\_범위     | cost_range            | 그대로      |
| 권장*반복*주기 | repeat_cycle          | 그대로      |
| 시술\_횟수     | session_count         | 그대로      |
| 효과\_누적     | cumulative_effect     | 그대로      |
| 피부톤\_제약   | skin_tone_restriction | 그대로      |
| 나이\_제약     | age_restriction       | 그대로      |
| 임신*중*안전성 | pregnancy_safe        | 그대로      |
| FDA\_승인      | fda_approved          | 그대로      |
| 장점           | pros                  | split(', ') |
| 단점           | cons                  | split(', ') |
| 치료\_대상     | treatment_target      | 그대로      |

### 자기관리\_데이터셋.csv → kb_self_care_items

| CSV 컬럼         | DB 컬럼                 | 변환               |
| ---------------- | ----------------------- | ------------------ |
| 항목\_ID         | id                      | 그대로             |
| 카테고리         | category_id             | 매핑 (하단)        |
| 항목\_이름       | name_ko                 | 그대로             |
| 영문\_이름       | name_en                 | 그대로             |
| 목표             | goal                    | 그대로             |
| 효과             | effects                 | split(', ')        |
| 영향\_부위       | target_areas            | split(', ')        |
| 방법             | method                  | 그대로             |
| 효과\_발현       | effect_onset            | 그대로             |
| 효과\_지속       | effect_duration         | 그대로             |
| 다운타임         | downtime                | 그대로             |
| 회복기간         | recovery_period         | 그대로             |
| 흔한\_부작용     | common_side_effects     | split(', ')        |
| 심각\_부작용     | serious_side_effects    | split(', ')        |
| 주의사항         | precautions             | parse list         |
| 금기사항         | contraindications       | parse list         |
| 비용             | cost_range              | 그대로             |
| 반복\_주기       | repeat_cycle            | 그대로             |
| 월\_시술횟수     | monthly_frequency       | 그대로             |
| 효과\_누적       | cumulative_effect       | 그대로             |
| 피부톤\_적합성   | skin_tone_suitability   | 그대로             |
| 나이\_제약       | age_restriction         | 그대로             |
| 임신중\_안전성   | pregnancy_safe          | 그대로             |
| 남성적용         | male_applicable         | 그대로             |
| 장점             | pros                    | parse list         |
| 단점             | cons                    | parse list         |
| 최적\_사용자     | optimal_user            | 그대로             |
| 추가\_팁         | additional_tips         | parse list         |
| 제품\_추천       | product_recommendations | 그대로             |
| 기간             | duration                | 그대로             |
| 지속\_기간       | duration_period         | 그대로             |
| 월\_활동         | monthly_activity        | 그대로             |
| 영문\_이명       | english_alias           | 그대로             |
| 시술\_유형       | procedure_type          | 그대로             |
| 횟수             | session_count           | 그대로             |
| 상세\_절차       | details.steps           | parse dict → JSONB |
| 제품*선택*가이드 | details.product_sel...  | parse dict → JSONB |
| SPF\_가이드      | details.spf_guide       | parse dict → JSONB |
| 식단\_가이드     | details.diet_guide      | parse dict → JSONB |
| 운동\_가이드     | details.exercise_guide  | parse dict → JSONB |
| 결과\_기준       | details.result_criteria | parse dict → JSONB |
| 색상\_선택       | details.color_selection | parse dict → JSONB |
| 시술소\_선택     | details.clinic_sel...   | parse list → JSONB |

### 카테고리 매핑

```python
CATEGORY_MAPPING = {
    # 시술정보
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

    # 자기관리
    "기초 미용 관리": "CAT011",
    "데일리 메이크업": "CAT012",
    "기초 피부 관리": "CAT013",
    "신체 자기관리": "CAT014",
    "신체 자기관리 - 전신 개선": "CAT014",
    "반영구 미용": "CAT015",
    "반영구 미용 시술": "CAT015",
}
```

---

## 데이터 임포트 순서

1. **kb_categories 테이블 생성 및 시드 데이터 삽입** (15 rows)
2. **kb_procedures 테이블 생성**
3. **kb_self_care_items 테이블 생성**
4. **시술정보\_데이터셋.csv → kb_procedures 임포트** (26 rows)
5. **자기관리\_데이터셋.csv → kb_self_care_items 임포트** (6 rows)
6. **검증**:
   - `SELECT COUNT(*) FROM kb_procedures` → 26 rows
   - `SELECT COUNT(*) FROM kb_self_care_items` → 6 rows

---

## 참고

- 구현 문서: `docs/implementation/02-database/models/kb-category.md`
- 구현 문서: `docs/implementation/02-database/models/kb-procedure.md`
- 구현 문서: `docs/implementation/02-database/models/kb-self-care-item.md`
- 데이터셋: `시술정보_데이터셋.csv`, `자기관리_데이터셋.csv`
