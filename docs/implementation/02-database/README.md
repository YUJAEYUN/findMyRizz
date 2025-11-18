# Phase 2: Database 설계 및 구현

## 📋 개요

정규화된 데이터베이스 스키마 및 SQLAlchemy 모델 구현

---

## 🎯 목표

1. **제2정규화(2NF) 준수** - 부분 함수 종속성 제거
2. **타입 안정성** - TEXT → INT/DECIMAL 구조화
3. **LangChain 호환** - RAG 구현을 위한 구조
4. **확장성** - 새로운 Knowledge 타입 추가 용이

---

## 📊 테이블 구조 (11개)

### Knowledge Base (3개)

1. **kb_categories** - 카테고리 마스터 (15 rows)
2. **kb_procedures** - 시술정보 (26 rows, 28 columns, 100% RDBMS)
3. **kb_self_care_items** - 자기관리 (5 rows, 38 columns, RDBMS + JSONB)

### Job 관련 (5개)

4. **jobs** - 중심 테이블
5. **payments** - 결제 정보 (1:1)
6. **job_files** - 파일 정보 (1:N)
7. **sms_logs** - SMS 발송 로그 (1:N)
8. **phone_verification_attempts** - 전화번호 인증 (1:N)

### Report 관련 (3개) - LangChain 반영 ✅

9. **reports** - AI 분석 리포트 (JSONB)
10. **report_knowledge_items** - RAG 매칭 결과 (N:M, Polymorphic)
11. **satisfaction_surveys** - 만족도 조사

---

## 🔍 설계 특징

### Knowledge Base 제2정규화

**CSV 데이터 기반 설계:**

- `시술정보_데이터셋.csv`: 26 rows, 28 columns → kb_procedures (100% RDBMS)
- `자기관리_데이터셋.csv`: 5 rows, 43 columns → kb_self_care_items (92% RDBMS + 8% JSONB)

**정규화 수준:**

```
kb_categories (15 rows)
├── kb_procedures (26 rows, 28 columns)
└── kb_self_care_items (5 rows, 38 columns)
```

- ✅ 제1정규화: 모든 속성이 원자값 (ARRAY 타입 사용)
- ✅ 제2정규화: category → item_type 함수 종속성 분리
- ✅ 타입별 테이블 분리: NULL 최소화, RDBMS 최대 활용

### RDBMS vs JSONB 전략

**kb_procedures (100% RDBMS):**

- 28개 컬럼 모두 정형화
- ARRAY 타입: main_effects, target_areas, precautions, contraindications, pros, cons
- JSONB 불필요

**kb_self_care_items (92% RDBMS + 8% JSONB):**

- 35개 컬럼: RDBMS
- 8개 필드: JSONB (비정형 데이터)
  - steps, product_selection_guide, spf_guide
  - diet_guide, exercise_guide, result_criteria
  - color_selection, clinic_selection

### Polymorphic 관계

**report_knowledge_items:**

- `item_id` (String) + `item_type` (PROCEDURE/SELF_CARE)
- kb_procedures 또는 kb_self_care_items 참조
- LangGraph match_knowledge_node 결과 저장

---

## 📁 파일 구조

```
docs/implementation/02-database/
├── README.md (이 파일)
├── connection.md
├── migrations.md
├── seed-data.md
└── models/
    ├── kb-category.md
    ├── kb-procedure.md
    ├── kb-self-care-item.md
    ├── job.md
    ├── payment.md
    ├── job-file.md
    ├── sms-log.md
    ├── phone-verification.md
    ├── report.md
    ├── report-knowledge-item.md
    └── satisfaction-survey.md
```

---

## 🚀 구현 순서

### Step 1: 데이터베이스 연결 설정

- `connection.md` 참고
- PostgreSQL 15+ 연결
- SQLAlchemy 2.0+ 설정

### Step 2: 모델 정의

- `models/` 디렉토리의 각 파일 참고
- Base 클래스 상속
- Relationship 설정

### Step 3: 마이그레이션

- `migrations.md` 참고
- Alembic 초기화
- 마이그레이션 파일 생성

### Step 4: 시드 데이터

- `seed-data.md` 참고
- Lookup 테이블 데이터
- Knowledge Base 초기 데이터

---

## ✅ 체크리스트

- [x] Knowledge Base 제2정규화 (3개 테이블)
- [x] CSV 데이터 완전 매핑
- [x] RDBMS 최대 활용 (96% RDBMS, 4% JSONB)
- [x] Report 구조화 (JSONB)
- [x] Polymorphic 관계 설계 (report_knowledge_items)
- [ ] Alembic 마이그레이션 생성
- [ ] CSV 임포트 스크립트 실행
- [ ] Phase 6 (Report Generation) 업데이트

---

## 📚 참고 문서

- `docs/technical/schema_kb.md` - Knowledge Base 스키마 상세
- `docs/technical/langchain_architecture.md` - LangChain 아키텍처
- `시술정보_데이터셋.csv` - 시술정보 원본 데이터 (26 rows)
- `자기관리_데이터셋.csv` - 자기관리 원본 데이터 (5 rows)

---

## 🔧 기술 스택

- **Database:** PostgreSQL 15+
- **ORM:** SQLAlchemy 2.0+
- **Migration:** Alembic
- **Connection Pool:** asyncpg
- **Type Checking:** Pydantic (for validation)
