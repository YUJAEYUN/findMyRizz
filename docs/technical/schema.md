## 데이터베이스 스키마

### Knowledge Base (3개 테이블)

```sql
-- 카테고리 마스터 (15 rows)
CREATE TABLE kb_categories (
    id VARCHAR(20) PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    item_type VARCHAR(20) NOT NULL,
    description VARCHAR(200),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX ix_kb_categories_item_type ON kb_categories(item_type);

-- 시술정보 (26 rows, 28 columns, 100% RDBMS)
CREATE TABLE kb_procedures (
    id VARCHAR(20) PRIMARY KEY,
    category_id VARCHAR(20) NOT NULL REFERENCES kb_categories(id),
    name_ko VARCHAR(100) NOT NULL,
    name_en VARCHAR(200),
    principle TEXT,
    effect_onset VARCHAR(100),
    main_effects TEXT[],
    target_areas VARCHAR(50)[],
    procedure_type VARCHAR(100),
    procedure_method TEXT,
    recovery_period VARCHAR(100),
    effect_duration VARCHAR(100),
    downtime VARCHAR(100),
    common_side_effects TEXT[],
    rare_side_effects TEXT[],
    precautions TEXT[],
    contraindications TEXT[],
    cost_range VARCHAR(100),
    repeat_cycle VARCHAR(100),
    session_count VARCHAR(100),
    cumulative_effect VARCHAR(50),
    skin_tone_restriction VARCHAR(100),
    age_restriction VARCHAR(100),
    pregnancy_safe VARCHAR(50),
    fda_approved VARCHAR(50),
    pros TEXT[],
    cons TEXT[],
    target_users TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX ix_kb_procedures_category_id ON kb_procedures(category_id);
CREATE INDEX ix_kb_procedures_name_ko ON kb_procedures(name_ko);
CREATE INDEX ix_kb_procedures_procedure_type ON kb_procedures(procedure_type);

-- 자기관리 (6 rows, 38 columns, RDBMS + JSONB)
CREATE TABLE kb_self_care_items (
    id VARCHAR(20) PRIMARY KEY,
    category_id VARCHAR(20) NOT NULL REFERENCES kb_categories(id),
    name_ko VARCHAR(100) NOT NULL,
    name_en VARCHAR(200),
    goal TEXT,
    effects TEXT[],
    target_areas VARCHAR(50)[],
    method TEXT,
    effect_onset VARCHAR(100),
    effect_duration VARCHAR(100),
    downtime VARCHAR(100),
    recovery_period VARCHAR(100),
    common_side_effects TEXT[],
    serious_side_effects TEXT[],
    precautions TEXT[],
    contraindications TEXT[],
    cost_range VARCHAR(100),
    repeat_cycle VARCHAR(100),
    monthly_frequency VARCHAR(100),
    cumulative_effect VARCHAR(50),
    skin_tone_suitability VARCHAR(100),
    age_restriction VARCHAR(100),
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
    session_count VARCHAR(100),
    details JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX ix_kb_self_care_items_category_id ON kb_self_care_items(category_id);
CREATE INDEX ix_kb_self_care_items_name_ko ON kb_self_care_items(name_ko);
CREATE INDEX ix_kb_self_care_items_details ON kb_self_care_items USING GIN(details);
```

---

### Job 관련 (5개 테이블)

```sql

CREATE TABLE jobs (
    job_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_phone_number VARCHAR(20),  -- 결제 성공 후 입력
    status VARCHAR(50) NOT NULL DEFAULT 'pending_payment',
    -- Status: pending_payment → pending_phone → pending_upload → processing → completed/failed/expired
    ai_analysis_text TEXT,
    error_message TEXT,
    merchant_uid VARCHAR(100) UNIQUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '24 hours',
    deleted_at TIMESTAMPTZ
);

CREATE INDEX idx_jobs_status ON jobs(status);
CREATE INDEX idx_jobs_expires_at ON jobs(expires_at);
CREATE INDEX idx_jobs_phone_hash ON jobs USING HASH (user_phone_number);
CREATE INDEX idx_jobs_deleted_at ON jobs(deleted_at) WHERE deleted_at IS NOT NULL;

CREATE TABLE payments (
    payment_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID NOT NULL REFERENCES jobs(job_id),
    pg_transaction_id TEXT,
    imp_uid VARCHAR(100),
    amount INT NOT NULL,
    method VARCHAR(50),
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    paid_at TIMESTAMPTZ
);

CREATE INDEX idx_payments_job_id ON payments(job_id);
CREATE INDEX idx_payments_imp_uid ON payments(imp_uid);

CREATE TABLE job_files (
    file_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID NOT NULL REFERENCES jobs(job_id) ON DELETE CASCADE,
    file_type VARCHAR(50) NOT NULL,
    s3_key TEXT NOT NULL,
    s3_bucket VARCHAR(100) NOT NULL,
    file_size_bytes BIGINT,
    mime_type VARCHAR(100),
    crop_data JSONB,
    display_order INT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_job_files_job_id ON job_files(job_id);
CREATE INDEX idx_job_files_type ON job_files(file_type);

CREATE TABLE sms_logs (
    sms_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID NOT NULL REFERENCES jobs(job_id) ON DELETE CASCADE,
    phone_number VARCHAR(20) NOT NULL,
    message_type VARCHAR(50) NOT NULL,
    message_content TEXT NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    coolsms_group_id VARCHAR(100),
    coolsms_message_id VARCHAR(100),
    error_message TEXT,
    sent_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_sms_logs_job_id ON sms_logs(job_id);
CREATE INDEX idx_sms_logs_status ON sms_logs(status);

CREATE TABLE phone_verification_attempts (
    attempt_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID NOT NULL REFERENCES jobs(job_id) ON DELETE CASCADE,
    phone_number VARCHAR(20) NOT NULL,
    ip_address INET NOT NULL,
    success BOOLEAN NOT NULL DEFAULT FALSE,
    attempted_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_phone_verification_job_id ON phone_verification_attempts(job_id);
CREATE INDEX idx_phone_verification_ip ON phone_verification_attempts(ip_address, attempted_at);
```

---

### Report 관련 (3개 테이블)

```sql
CREATE TABLE reports (
    report_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID NOT NULL REFERENCES jobs(job_id) ON DELETE CASCADE,
    analysis_summary TEXT,
    improvement_areas JSONB,
    specific_observations JSONB,
    confidence_score NUMERIC(3, 2),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_reports_job_id ON reports(job_id);

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

CREATE TABLE satisfaction_surveys (
    survey_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID REFERENCES jobs(job_id) ON DELETE SET NULL,
    survey_data JSONB NOT NULL,
    feedback_text TEXT,
    review VARCHAR(250),
    submitted_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_satisfaction_surveys_gin ON satisfaction_surveys USING GIN (survey_data);
CREATE INDEX idx_satisfaction_surveys_job_id ON satisfaction_surveys(job_id);
```

---

## 참고 문서

- **Knowledge Base 상세 스키마**: `docs/technical/schema_kb.md`
- **모델 구현 문서**: `docs/implementation/02-database/models/`
  - `kb-category.md` - 카테고리 마스터 + 시드 데이터
  - `kb-procedure.md` - 시술정보 모델 + CSV 임포트
  - `kb-self-care-item.md` - 자기관리 모델 + CSV 임포트
  - `report-knowledge-item.md` - Polymorphic 관계 모델
- **CSV 데이터**:
  - `시술정보_데이터셋.csv` (26 rows)
  - `자기관리_데이터셋.csv` (5 rows)
