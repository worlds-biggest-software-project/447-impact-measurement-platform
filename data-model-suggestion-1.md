# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

## Overview

This model uses a fully normalized relational schema in PostgreSQL, designed around the core domain concepts of the Impact Measurement Platform: organisations, programmes, theories of change, participants, data collection instruments, outcome indicators, and reporting frameworks. Every entity has a dedicated table with foreign key relationships enforcing referential integrity. The schema follows Third Normal Form (3NF) throughout, minimising redundancy and ensuring data consistency across what may be years-long longitudinal tracking.

The normalized approach is particularly well-suited to the evidence continuity problem the platform aims to solve. By maintaining strict referential integrity between baseline assessments, participation records, and follow-up surveys through persistent participant identifiers, the relational model guarantees that no evidence link can be broken by application-level bugs or inconsistent writes.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Primary Database | PostgreSQL 16+ | Mature, open-source, excellent for complex joins across normalised tables; supports row-level security for multi-tenant isolation |
| Connection Pooling | PgBouncer | Essential for nonprofit deployments where connection limits matter |
| Migrations | Flyway or Alembic | Version-controlled schema migrations for self-hosted deployments |
| Full-Text Search | PostgreSQL tsvector/tsquery | Built-in full-text search for qualitative data without external dependency |
| Backup | pg_dump + WAL archiving | Point-in-time recovery for compliance and data protection |
| ORM | Prisma or SQLAlchemy | Type-safe query building; Prisma for Node.js stack, SQLAlchemy for Python |

---

## Complete Schema Definition

### Core Organisation and Multi-Tenancy

```sql
-- Organisations are the top-level tenant boundary.
-- Every subsequent table references an organisation directly or transitively.
CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    logo_url VARCHAR(500),
    website VARCHAR(500),
    primary_contact_email VARCHAR(255),
    country_code CHAR(2),
    timezone VARCHAR(50) DEFAULT 'UTC',
    gdpr_data_controller BOOLEAN DEFAULT FALSE,
    data_retention_policy_months INTEGER DEFAULT 84,  -- 7 years default
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ  -- soft delete for audit trail
);

CREATE INDEX idx_organisations_slug ON organisations(slug);

-- Users belong to organisations with role-based access.
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    role VARCHAR(50) NOT NULL CHECK (role IN (
        'org_admin', 'programme_manager', 'evaluator',
        'data_collector', 'report_viewer', 'api_client'
    )),
    is_active BOOLEAN DEFAULT TRUE,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(organisation_id, email)
);

CREATE INDEX idx_users_org ON users(organisation_id);
CREATE INDEX idx_users_email ON users(email);
```

### Programme and Theory of Change

```sql
-- Programmes are the primary unit of impact measurement.
CREATE TABLE programmes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    programme_type VARCHAR(100) CHECK (programme_type IN (
        'workforce_development', 'housing', 'youth_mentoring',
        'health', 'education', 'food_security', 'financial_inclusion',
        'environmental', 'community_development', 'other'
    )),
    status VARCHAR(50) DEFAULT 'draft' CHECK (status IN (
        'draft', 'active', 'paused', 'completed', 'archived'
    )),
    start_date DATE,
    end_date DATE,
    target_participant_count INTEGER,
    budget_amount NUMERIC(15,2),
    budget_currency CHAR(3) DEFAULT 'USD',
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_programmes_org ON programmes(organisation_id);
CREATE INDEX idx_programmes_status ON programmes(status);

-- Theory of Change is the causal model for a programme.
-- Each ToC has a hierarchical chain: inputs -> activities -> outputs -> outcomes -> impact.
CREATE TABLE theories_of_change (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id UUID NOT NULL REFERENCES programmes(id) ON DELETE CASCADE,
    version INTEGER NOT NULL DEFAULT 1,
    name VARCHAR(255) NOT NULL,
    narrative TEXT,  -- prose description of the causal theory
    is_current BOOLEAN DEFAULT TRUE,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(programme_id, version)
);

CREATE INDEX idx_toc_programme ON theories_of_change(programme_id);

-- Individual nodes in the theory of change chain.
CREATE TABLE toc_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    toc_id UUID NOT NULL REFERENCES theories_of_change(id) ON DELETE CASCADE,
    node_type VARCHAR(50) NOT NULL CHECK (node_type IN (
        'input', 'activity', 'output', 'outcome', 'impact'
    )),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,
    -- Visual positioning for the ToC builder UI
    position_x REAL,
    position_y REAL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_toc_nodes_toc ON toc_nodes(toc_id);
CREATE INDEX idx_toc_nodes_type ON toc_nodes(toc_id, node_type);

-- Directed edges connecting ToC nodes (causal pathways).
CREATE TABLE toc_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    toc_id UUID NOT NULL REFERENCES theories_of_change(id) ON DELETE CASCADE,
    from_node_id UUID NOT NULL REFERENCES toc_nodes(id) ON DELETE CASCADE,
    to_node_id UUID NOT NULL REFERENCES toc_nodes(id) ON DELETE CASCADE,
    label VARCHAR(255),
    -- Assumptions that must hold for this causal link
    assumption TEXT,
    evidence_strength VARCHAR(20) CHECK (evidence_strength IN (
        'strong', 'moderate', 'weak', 'untested'
    )),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(from_node_id, to_node_id)
);

CREATE INDEX idx_toc_edges_toc ON toc_edges(toc_id);
CREATE INDEX idx_toc_edges_from ON toc_edges(from_node_id);
CREATE INDEX idx_toc_edges_to ON toc_edges(to_node_id);

-- Assumptions associated with the theory of change.
CREATE TABLE toc_assumptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    toc_id UUID NOT NULL REFERENCES theories_of_change(id) ON DELETE CASCADE,
    edge_id UUID REFERENCES toc_edges(id) ON DELETE SET NULL,
    description TEXT NOT NULL,
    status VARCHAR(50) DEFAULT 'untested' CHECK (status IN (
        'untested', 'supported', 'partially_supported', 'refuted'
    )),
    evidence_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Indicators and Framework Alignment

```sql
-- Outcome indicators define what is measured.
CREATE TABLE indicators (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    code VARCHAR(50),  -- internal code like "EMP-01"
    name VARCHAR(255) NOT NULL,
    description TEXT,
    measurement_type VARCHAR(50) NOT NULL CHECK (measurement_type IN (
        'numeric', 'percentage', 'scale', 'boolean', 'categorical', 'text'
    )),
    unit_of_measure VARCHAR(100),  -- e.g. "hours", "dollars", "score (1-10)"
    direction VARCHAR(20) DEFAULT 'increase' CHECK (direction IN (
        'increase', 'decrease', 'maintain', 'target'
    )),
    data_source VARCHAR(100),  -- e.g. "participant_survey", "admin_records"
    collection_frequency VARCHAR(50) CHECK (collection_frequency IN (
        'one_time', 'weekly', 'monthly', 'quarterly', 'semi_annual',
        'annual', 'baseline_endline', 'custom'
    )),
    is_core BOOLEAN DEFAULT FALSE,  -- organisation-wide core indicator
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_indicators_org ON indicators(organisation_id);
CREATE INDEX idx_indicators_code ON indicators(organisation_id, code);

-- Link indicators to ToC nodes (which outcomes does this indicator measure?).
CREATE TABLE toc_node_indicators (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    toc_node_id UUID NOT NULL REFERENCES toc_nodes(id) ON DELETE CASCADE,
    indicator_id UUID NOT NULL REFERENCES indicators(id) ON DELETE CASCADE,
    target_value NUMERIC,
    target_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(toc_node_id, indicator_id)
);

-- External frameworks (IRIS+, SDGs, funder-specific).
CREATE TABLE frameworks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    version VARCHAR(50),
    description TEXT,
    source_url VARCHAR(500),
    framework_type VARCHAR(50) CHECK (framework_type IN (
        'iris_plus', 'sdg', 'funder_template', 'government',
        'custom', 'sroi', 'social_value'
    )),
    is_system BOOLEAN DEFAULT FALSE,  -- pre-loaded vs user-created
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Individual metrics/goals within a framework (e.g. SDG Goal 1, IRIS PI1234).
CREATE TABLE framework_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id UUID NOT NULL REFERENCES frameworks(id) ON DELETE CASCADE,
    external_code VARCHAR(100) NOT NULL,  -- e.g. "PI1234", "SDG-1.1"
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(255),  -- hierarchical category within the framework
    parent_metric_id UUID REFERENCES framework_metrics(id),
    sort_order INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_framework_metrics_fw ON framework_metrics(framework_id);
CREATE INDEX idx_framework_metrics_code ON framework_metrics(framework_id, external_code);

-- Mapping between internal indicators and external framework metrics.
CREATE TABLE indicator_framework_mappings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    indicator_id UUID NOT NULL REFERENCES indicators(id) ON DELETE CASCADE,
    framework_metric_id UUID NOT NULL REFERENCES framework_metrics(id) ON DELETE CASCADE,
    mapping_confidence VARCHAR(20) DEFAULT 'exact' CHECK (mapping_confidence IN (
        'exact', 'approximate', 'partial', 'derived'
    )),
    mapping_notes TEXT,
    mapped_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(indicator_id, framework_metric_id)
);

CREATE INDEX idx_indicator_fw_map_indicator ON indicator_framework_mappings(indicator_id);
CREATE INDEX idx_indicator_fw_map_metric ON indicator_framework_mappings(framework_metric_id);
```

### Participant Identity and Consent

```sql
-- Participants are the individuals whose outcomes are tracked.
-- This is the persistent identity that solves the evidence continuity problem.
CREATE TABLE participants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    -- Pseudonymised identifier for GDPR compliance
    pseudonym_id VARCHAR(100) NOT NULL,
    -- PII fields are stored separately and can be encrypted at rest
    first_name_encrypted BYTEA,
    last_name_encrypted BYTEA,
    date_of_birth_encrypted BYTEA,
    email_encrypted BYTEA,
    phone_encrypted BYTEA,
    -- Non-PII demographic fields for outcome disaggregation
    gender VARCHAR(50),
    ethnicity VARCHAR(100),
    primary_language VARCHAR(50),
    geographic_region VARCHAR(255),
    postcode_area VARCHAR(10),  -- truncated for privacy
    -- Deduplication support
    name_phonetic_hash VARCHAR(100),  -- for probabilistic matching
    dob_hash VARCHAR(100),
    dedup_cluster_id UUID,  -- links deduplicated records
    -- Status tracking
    status VARCHAR(50) DEFAULT 'active' CHECK (status IN (
        'active', 'inactive', 'withdrawn', 'deceased', 'merged'
    )),
    merged_into_id UUID REFERENCES participants(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(organisation_id, pseudonym_id)
);

CREATE INDEX idx_participants_org ON participants(organisation_id);
CREATE INDEX idx_participants_pseudonym ON participants(pseudonym_id);
CREATE INDEX idx_participants_dedup ON participants(name_phonetic_hash, dob_hash);

-- Consent records for GDPR/privacy compliance.
CREATE TABLE participant_consents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    participant_id UUID NOT NULL REFERENCES participants(id) ON DELETE CASCADE,
    consent_type VARCHAR(100) NOT NULL CHECK (consent_type IN (
        'data_collection', 'data_sharing_anonymised',
        'data_sharing_identified', 'contact_followup',
        'research_use', 'photo_media', 'third_party_sharing'
    )),
    granted BOOLEAN NOT NULL,
    granted_at TIMESTAMPTZ,
    withdrawn_at TIMESTAMPTZ,
    consent_method VARCHAR(50) CHECK (consent_method IN (
        'written_form', 'digital_signature', 'verbal_recorded',
        'online_checkbox', 'parent_guardian'
    )),
    consent_document_url VARCHAR(500),
    expires_at TIMESTAMPTZ,
    recorded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_consents_participant ON participant_consents(participant_id);
CREATE INDEX idx_consents_type ON participant_consents(participant_id, consent_type);

-- Data Subject Access Request tracking (GDPR Article 15).
CREATE TABLE data_access_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    participant_id UUID REFERENCES participants(id),
    request_type VARCHAR(50) NOT NULL CHECK (request_type IN (
        'access', 'rectification', 'erasure', 'portability', 'restriction'
    )),
    status VARCHAR(50) DEFAULT 'received' CHECK (status IN (
        'received', 'in_progress', 'completed', 'denied'
    )),
    requested_at TIMESTAMPTZ NOT NULL,
    due_by TIMESTAMPTZ NOT NULL,  -- 30-day GDPR deadline
    completed_at TIMESTAMPTZ,
    handled_by UUID REFERENCES users(id),
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Programme Enrolment and Cohorts

```sql
-- Enrolment links participants to programmes with temporal tracking.
CREATE TABLE programme_enrolments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    participant_id UUID NOT NULL REFERENCES participants(id),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    cohort_id UUID,  -- assigned below
    enrolment_date DATE NOT NULL,
    exit_date DATE,
    exit_reason VARCHAR(100) CHECK (exit_reason IN (
        'completed', 'withdrew', 'transferred', 'lost_to_followup',
        'ineligible', 'programme_ended', 'other'
    )),
    status VARCHAR(50) DEFAULT 'enrolled' CHECK (status IN (
        'enrolled', 'active', 'on_hold', 'completed',
        'withdrawn', 'lost_to_followup'
    )),
    referral_source VARCHAR(255),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_enrolments_participant ON programme_enrolments(participant_id);
CREATE INDEX idx_enrolments_programme ON programme_enrolments(programme_id);
CREATE INDEX idx_enrolments_status ON programme_enrolments(programme_id, status);

-- Cohorts group participants for comparative analysis.
CREATE TABLE cohorts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    start_date DATE,
    end_date DATE,
    is_control_group BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_cohorts_programme ON cohorts(programme_id);

-- Now add the FK for enrolments -> cohorts
ALTER TABLE programme_enrolments
    ADD CONSTRAINT fk_enrolment_cohort
    FOREIGN KEY (cohort_id) REFERENCES cohorts(id);

-- Activity attendance tracking (outputs).
CREATE TABLE activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    toc_node_id UUID REFERENCES toc_nodes(id),  -- link to ToC activity node
    name VARCHAR(255) NOT NULL,
    description TEXT,
    activity_type VARCHAR(100),
    scheduled_date DATE,
    scheduled_time TIME,
    duration_minutes INTEGER,
    location VARCHAR(255),
    facilitator_id UUID REFERENCES users(id),
    max_capacity INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_activities_programme ON activities(programme_id);
CREATE INDEX idx_activities_date ON activities(scheduled_date);

CREATE TABLE activity_attendance (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    activity_id UUID NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    enrolment_id UUID NOT NULL REFERENCES programme_enrolments(id),
    attended BOOLEAN NOT NULL DEFAULT TRUE,
    attendance_type VARCHAR(50) CHECK (attendance_type IN (
        'in_person', 'virtual', 'hybrid', 'excused_absence',
        'unexcused_absence'
    )),
    notes TEXT,
    recorded_by UUID REFERENCES users(id),
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(activity_id, enrolment_id)
);

CREATE INDEX idx_attendance_activity ON activity_attendance(activity_id);
CREATE INDEX idx_attendance_enrolment ON activity_attendance(enrolment_id);
```

### Data Collection: Forms, Surveys, and Assessments

```sql
-- Form definitions (survey instruments, assessment tools).
CREATE TABLE form_definitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    programme_id UUID REFERENCES programmes(id),  -- NULL = org-wide form
    name VARCHAR(255) NOT NULL,
    description TEXT,
    form_type VARCHAR(50) NOT NULL CHECK (form_type IN (
        'baseline_assessment', 'midpoint_assessment', 'endline_assessment',
        'followup_survey', 'intake_form', 'exit_form',
        'feedback_360', 'case_note', 'custom'
    )),
    version INTEGER NOT NULL DEFAULT 1,
    status VARCHAR(50) DEFAULT 'draft' CHECK (status IN (
        'draft', 'published', 'archived'
    )),
    is_template BOOLEAN DEFAULT FALSE,
    estimated_duration_minutes INTEGER,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_forms_org ON form_definitions(organisation_id);
CREATE INDEX idx_forms_programme ON form_definitions(programme_id);

-- Form sections group related questions.
CREATE TABLE form_sections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_id UUID NOT NULL REFERENCES form_definitions(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_repeatable BOOLEAN DEFAULT FALSE,
    display_condition TEXT,  -- skip logic expression
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_form_sections_form ON form_sections(form_id);

-- Individual questions/fields within a form.
CREATE TABLE form_questions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id UUID NOT NULL REFERENCES form_sections(id) ON DELETE CASCADE,
    indicator_id UUID REFERENCES indicators(id),  -- links question to indicator
    question_text TEXT NOT NULL,
    help_text TEXT,
    question_type VARCHAR(50) NOT NULL CHECK (question_type IN (
        'text', 'textarea', 'number', 'decimal', 'date',
        'single_choice', 'multiple_choice', 'likert_scale',
        'ranking', 'file_upload', 'geolocation', 'barcode',
        'signature', 'matrix', 'calculated'
    )),
    is_required BOOLEAN DEFAULT FALSE,
    sort_order INTEGER NOT NULL DEFAULT 0,
    validation_rules TEXT,  -- JSON validation rules
    display_condition TEXT,  -- skip logic
    -- For scale questions
    scale_min INTEGER,
    scale_max INTEGER,
    scale_min_label VARCHAR(100),
    scale_max_label VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_form_questions_section ON form_questions(section_id);
CREATE INDEX idx_form_questions_indicator ON form_questions(indicator_id);

-- Choice options for single/multiple choice questions.
CREATE TABLE form_question_choices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id UUID NOT NULL REFERENCES form_questions(id) ON DELETE CASCADE,
    label VARCHAR(255) NOT NULL,
    value VARCHAR(100) NOT NULL,
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_other BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_choices_question ON form_question_choices(question_id);

-- Submitted form responses (one row per form submission).
CREATE TABLE form_submissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_id UUID NOT NULL REFERENCES form_definitions(id),
    enrolment_id UUID NOT NULL REFERENCES programme_enrolments(id),
    submitted_by UUID REFERENCES users(id),
    submission_type VARCHAR(50) CHECK (submission_type IN (
        'self_report', 'staff_administered', 'peer_assessment',
        'supervisor_assessment', 'automated'
    )),
    status VARCHAR(50) DEFAULT 'draft' CHECK (status IN (
        'draft', 'submitted', 'validated', 'flagged', 'rejected'
    )),
    collection_method VARCHAR(50) CHECK (collection_method IN (
        'web', 'mobile_online', 'mobile_offline', 'paper_digitised',
        'phone', 'api_import'
    )),
    started_at TIMESTAMPTZ,
    submitted_at TIMESTAMPTZ,
    validated_at TIMESTAMPTZ,
    validated_by UUID REFERENCES users(id),
    device_id VARCHAR(100),  -- for mobile collection tracking
    gps_latitude DECIMAL(10,7),
    gps_longitude DECIMAL(10,7),
    sync_status VARCHAR(20) DEFAULT 'synced' CHECK (sync_status IN (
        'pending', 'syncing', 'synced', 'conflict'
    )),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_submissions_form ON form_submissions(form_id);
CREATE INDEX idx_submissions_enrolment ON form_submissions(enrolment_id);
CREATE INDEX idx_submissions_date ON form_submissions(submitted_at);

-- Individual answer values within a submission.
CREATE TABLE form_answers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    submission_id UUID NOT NULL REFERENCES form_submissions(id) ON DELETE CASCADE,
    question_id UUID NOT NULL REFERENCES form_questions(id),
    -- Typed value columns: only one is populated per answer
    value_text TEXT,
    value_numeric NUMERIC,
    value_date DATE,
    value_boolean BOOLEAN,
    value_choice_id UUID REFERENCES form_question_choices(id),
    -- For multi-select, stored as comma-separated choice IDs
    value_choice_ids TEXT,
    -- Raw value as entered (for audit)
    raw_value TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_answers_submission ON form_answers(submission_id);
CREATE INDEX idx_answers_question ON form_answers(question_id);
```

### Outcome Measurement and Distance Travelled

```sql
-- Recorded outcome measurements linked to participants and indicators.
CREATE TABLE outcome_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrolment_id UUID NOT NULL REFERENCES programme_enrolments(id),
    indicator_id UUID NOT NULL REFERENCES indicators(id),
    measurement_period VARCHAR(50) NOT NULL CHECK (measurement_period IN (
        'baseline', 'month_1', 'month_3', 'month_6', 'midpoint',
        'month_12', 'endline', 'followup_3m', 'followup_6m',
        'followup_12m', 'custom'
    )),
    measurement_date DATE NOT NULL,
    value_numeric NUMERIC,
    value_text TEXT,
    value_categorical VARCHAR(255),
    -- Source traceability
    form_submission_id UUID REFERENCES form_submissions(id),
    form_answer_id UUID REFERENCES form_answers(id),
    data_source VARCHAR(100),  -- manual entry, imported, calculated
    -- Validation
    is_validated BOOLEAN DEFAULT FALSE,
    validated_by UUID REFERENCES users(id),
    validation_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outcomes_enrolment ON outcome_records(enrolment_id);
CREATE INDEX idx_outcomes_indicator ON outcome_records(indicator_id);
CREATE INDEX idx_outcomes_period ON outcome_records(measurement_period);
CREATE INDEX idx_outcomes_date ON outcome_records(measurement_date);
-- Composite index for distance-travelled queries (baseline vs endline)
CREATE INDEX idx_outcomes_distance ON outcome_records(enrolment_id, indicator_id, measurement_period);

-- Pre-computed distance travelled scores.
CREATE TABLE distance_travelled_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrolment_id UUID NOT NULL REFERENCES programme_enrolments(id),
    indicator_id UUID NOT NULL REFERENCES indicators(id),
    baseline_value NUMERIC,
    baseline_date DATE,
    current_value NUMERIC,
    current_date DATE,
    absolute_change NUMERIC,
    percentage_change NUMERIC,
    standardised_change NUMERIC,  -- Cohen's d or equivalent
    reliable_change_index NUMERIC,  -- RCI for statistical significance
    is_reliable_change BOOLEAN,
    direction VARCHAR(20) CHECK (direction IN ('improved', 'maintained', 'declined')),
    calculated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(enrolment_id, indicator_id)
);

CREATE INDEX idx_distance_enrolment ON distance_travelled_scores(enrolment_id);
CREATE INDEX idx_distance_indicator ON distance_travelled_scores(indicator_id);

-- Programme-level aggregated outcome summaries.
CREATE TABLE programme_outcome_summaries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    indicator_id UUID NOT NULL REFERENCES indicators(id),
    cohort_id UUID REFERENCES cohorts(id),  -- NULL = all cohorts
    reporting_period_start DATE NOT NULL,
    reporting_period_end DATE NOT NULL,
    participant_count INTEGER,
    measured_count INTEGER,
    -- Aggregate statistics
    mean_baseline NUMERIC,
    mean_endline NUMERIC,
    mean_change NUMERIC,
    median_change NUMERIC,
    std_dev_change NUMERIC,
    -- Outcome categories
    improved_count INTEGER,
    maintained_count INTEGER,
    declined_count INTEGER,
    improved_percentage NUMERIC(5,2),
    -- Target tracking
    target_value NUMERIC,
    target_achieved BOOLEAN,
    calculated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(programme_id, indicator_id, cohort_id, reporting_period_start, reporting_period_end)
);

CREATE INDEX idx_prog_outcomes_programme ON programme_outcome_summaries(programme_id);
```

### Qualitative Data and AI Analysis

```sql
-- Qualitative evidence items (case notes, interview transcripts, open-ended responses).
CREATE TABLE qualitative_evidence (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    enrolment_id UUID REFERENCES programme_enrolments(id),
    programme_id UUID REFERENCES programmes(id),
    evidence_type VARCHAR(50) NOT NULL CHECK (evidence_type IN (
        'case_note', 'interview_transcript', 'open_ended_response',
        'focus_group', 'observation', 'story_of_change', 'other'
    )),
    content TEXT NOT NULL,
    author_id UUID REFERENCES users(id),
    form_submission_id UUID REFERENCES form_submissions(id),
    form_answer_id UUID REFERENCES form_answers(id),
    recorded_date DATE,
    word_count INTEGER,
    language VARCHAR(10) DEFAULT 'en',
    -- Full text search vector
    content_tsv TSVECTOR GENERATED ALWAYS AS (to_tsvector('english', content)) STORED,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_qual_org ON qualitative_evidence(organisation_id);
CREATE INDEX idx_qual_enrolment ON qualitative_evidence(enrolment_id);
CREATE INDEX idx_qual_programme ON qualitative_evidence(programme_id);
CREATE INDEX idx_qual_tsv ON qualitative_evidence USING GIN (content_tsv);

-- AI-generated thematic codes assigned to qualitative evidence.
CREATE TABLE thematic_codes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    code VARCHAR(100) NOT NULL,
    label VARCHAR(255) NOT NULL,
    description TEXT,
    parent_code_id UUID REFERENCES thematic_codes(id),
    source VARCHAR(50) CHECK (source IN ('manual', 'ai_generated', 'framework_derived')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(organisation_id, code)
);

CREATE TABLE qualitative_evidence_codes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    evidence_id UUID NOT NULL REFERENCES qualitative_evidence(id) ON DELETE CASCADE,
    code_id UUID NOT NULL REFERENCES thematic_codes(id),
    confidence_score NUMERIC(3,2),  -- 0.00-1.00 for AI-assigned codes
    assigned_by VARCHAR(50) CHECK (assigned_by IN ('human', 'ai')),
    assigned_by_user_id UUID REFERENCES users(id),
    excerpt TEXT,  -- the specific passage that was coded
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(evidence_id, code_id)
);

-- AI-generated sentiment analysis results.
CREATE TABLE sentiment_analysis (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    evidence_id UUID NOT NULL REFERENCES qualitative_evidence(id) ON DELETE CASCADE,
    overall_sentiment VARCHAR(20) CHECK (overall_sentiment IN (
        'very_positive', 'positive', 'neutral', 'negative', 'very_negative'
    )),
    sentiment_score NUMERIC(4,3),  -- -1.000 to 1.000
    model_version VARCHAR(100),
    analysed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Anomaly detection alerts on outcome trends.
CREATE TABLE outcome_anomalies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    indicator_id UUID NOT NULL REFERENCES indicators(id),
    cohort_id UUID REFERENCES cohorts(id),
    anomaly_type VARCHAR(50) CHECK (anomaly_type IN (
        'unexpected_decline', 'unexpected_improvement',
        'high_variance', 'missing_data_spike', 'outlier_cluster'
    )),
    severity VARCHAR(20) CHECK (severity IN ('low', 'medium', 'high', 'critical')),
    description TEXT,
    detected_value NUMERIC,
    expected_value NUMERIC,
    deviation_score NUMERIC,
    detected_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES users(id),
    resolution_notes TEXT
);

CREATE INDEX idx_anomalies_programme ON outcome_anomalies(programme_id);
```

### Reporting

```sql
-- Report definitions and generated reports.
CREATE TABLE report_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    report_type VARCHAR(50) CHECK (report_type IN (
        'funder_report', 'annual_impact', 'programme_summary',
        'outcome_dashboard', 'participant_progress', 'custom'
    )),
    framework_id UUID REFERENCES frameworks(id),
    template_content TEXT,  -- template markup
    is_system BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE generated_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id UUID NOT NULL REFERENCES report_templates(id),
    programme_id UUID REFERENCES programmes(id),
    generated_by UUID REFERENCES users(id),
    title VARCHAR(255) NOT NULL,
    reporting_period_start DATE,
    reporting_period_end DATE,
    format VARCHAR(20) CHECK (format IN ('pdf', 'docx', 'xlsx', 'html')),
    file_url VARCHAR(500),
    file_size_bytes BIGINT,
    status VARCHAR(50) DEFAULT 'generating' CHECK (status IN (
        'generating', 'completed', 'failed', 'expired'
    )),
    generated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ
);

CREATE INDEX idx_reports_programme ON generated_reports(programme_id);
CREATE INDEX idx_reports_date ON generated_reports(generated_at);
```

### Integration and Data Import

```sql
-- External system integrations.
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    integration_type VARCHAR(50) NOT NULL CHECK (integration_type IN (
        'salesforce', 'apricot', 'eto', 'kobotoolbox', 'odk',
        'csv_import', 'api', 'webhook'
    )),
    name VARCHAR(255) NOT NULL,
    config_encrypted BYTEA,  -- encrypted connection configuration
    is_active BOOLEAN DEFAULT TRUE,
    last_sync_at TIMESTAMPTZ,
    last_sync_status VARCHAR(50),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Import batch tracking for audit trail.
CREATE TABLE import_batches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    integration_id UUID REFERENCES integrations(id),
    source_type VARCHAR(50) NOT NULL,
    source_filename VARCHAR(255),
    status VARCHAR(50) DEFAULT 'pending' CHECK (status IN (
        'pending', 'validating', 'importing', 'completed',
        'completed_with_errors', 'failed', 'rolled_back'
    )),
    total_records INTEGER,
    imported_records INTEGER,
    skipped_records INTEGER,
    error_records INTEGER,
    error_log TEXT,
    imported_by UUID REFERENCES users(id),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_imports_org ON import_batches(organisation_id);
```

### Audit Trail

```sql
-- Comprehensive audit log for compliance.
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    user_id UUID REFERENCES users(id),
    action VARCHAR(50) NOT NULL CHECK (action IN (
        'create', 'read', 'update', 'delete', 'export',
        'login', 'logout', 'consent_change', 'data_access_request'
    )),
    entity_type VARCHAR(100) NOT NULL,
    entity_id UUID,
    old_values TEXT,  -- JSON of previous values
    new_values TEXT,  -- JSON of new values
    ip_address INET,
    user_agent VARCHAR(500),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_date ON audit_log(created_at);
-- Partition audit log by month for performance
-- (In production, use PostgreSQL declarative partitioning)
```

---

## Key Design Decisions

### Evidence Continuity Through Foreign Keys

The central design principle is the chain: `participants` -> `programme_enrolments` -> `form_submissions` -> `form_answers` -> `outcome_records`. Every outcome measurement traces back through validated foreign keys to the specific participant, the specific programme enrolment, the specific form submission, and the specific question answered. This chain cannot be broken without violating database constraints, which is exactly the guarantee the platform promises.

### Pseudonymisation Architecture

Participant PII (names, dates of birth, contact details) is stored in encrypted bytea columns within the `participants` table. The `pseudonym_id` serves as the working identifier throughout the system. Encryption keys are managed externally (e.g., AWS KMS, HashiCorp Vault). In the event of a GDPR erasure request, PII columns can be nullified while preserving the pseudonymised outcome data for aggregate reporting.

### Multi-Tenancy via Organisation Scoping

Rather than separate schemas per tenant, the model uses `organisation_id` foreign keys with row-level security (RLS) policies in PostgreSQL. This simplifies deployment for self-hosted scenarios while providing strong isolation:

```sql
ALTER TABLE programmes ENABLE ROW LEVEL SECURITY;
CREATE POLICY programmes_org_policy ON programmes
    USING (organisation_id = current_setting('app.current_org_id')::UUID);
```

### Versioned Theories of Change

The `theories_of_change` table supports versioning so that when a programme's causal model evolves based on evidence, the historical version is preserved. Outcome measurements remain linked to the ToC version under which they were collected.

---

## Pros and Cons

### Pros

1. **Complete referential integrity**: Every evidence link is enforced by the database engine, not application code. The chain from participant through enrolment to outcome is unbreakable.

2. **Mature ecosystem**: PostgreSQL has decades of production reliability, extensive documentation, and wide availability of managed hosting (AWS RDS, Supabase, Neon). Nonprofits with limited DevOps capacity can rely on managed services.

3. **Strong querying for complex analysis**: Normalised tables excel at the complex joins needed for longitudinal analysis -- comparing baseline to endline across cohorts, disaggregating outcomes by demographics, rolling up indicators to framework metrics.

4. **GDPR compliance built in**: Row-level security, column-level encryption, and the pseudonymisation architecture directly support the platform's privacy requirements.

5. **Audit trail without duplication**: Because data is normalised, the audit log only needs to track changes to individual fields rather than entire denormalised documents.

6. **Framework alignment is first-class**: The `frameworks` -> `framework_metrics` -> `indicator_framework_mappings` chain makes it straightforward to map any indicator to IRIS+, SDGs, or funder-specific templates.

### Cons

1. **Schema rigidity for variable data**: Different programmes measure different outcomes with different instruments. Adding new question types, indicator categories, or assessment scales requires schema changes or relies on generic columns (like `value_text` catch-alls).

2. **Complex migrations**: With 30+ interrelated tables, schema migrations become operationally risky, especially for self-hosted deployments where organisations may be running different versions.

3. **Performance at scale with many joins**: Generating an impact report that spans participants, enrolments, submissions, answers, outcomes, and framework mappings requires 6+ table joins. At scale (100K+ participants, millions of outcome records), these queries need careful index tuning and possibly materialised views.

4. **Form definition rigidity**: The `form_definitions` -> `form_sections` -> `form_questions` -> `form_question_choices` structure works well for standard surveys but struggles with highly dynamic forms (conditional sections, repeating groups, calculated fields with complex logic).

5. **Offline sync complexity**: The normalised model requires careful conflict resolution when mobile devices sync after offline data collection. Each table's foreign key constraints must be satisfied during sync, requiring specific insertion ordering.

6. **Qualitative data storage**: Storing large text blobs (interview transcripts, case notes) in relational tables is functional but not optimal. Full-text search via tsvector works but is less capable than dedicated search engines for the AI analysis use cases.

---

## Migration and Scaling Considerations

### Initial Deployment

For organisations with fewer than 10,000 participants and a handful of programmes, a single PostgreSQL instance (even a modest 2-core, 4GB managed instance) will handle the load comfortably. The schema is designed to work from day one without sharding or partitioning.

### Growth Path (10K-100K participants)

- **Materialised views** for programme outcome summaries, refreshed on a schedule rather than computed on every report request
- **Table partitioning** on `audit_log` (by month), `form_submissions` (by year), and `outcome_records` (by year) using PostgreSQL declarative partitioning
- **Read replicas** for reporting queries, keeping the primary instance responsive for data collection
- **Connection pooling** via PgBouncer becomes essential

### Large Scale (100K+ participants)

- **Citus extension** for horizontal sharding by `organisation_id`, distributing tenant data across multiple nodes while preserving the relational model
- **pg_partman** for automated partition management on time-series-like tables
- **Separate reporting database** with ETL pipeline (dbt or similar) for complex cross-programme analytics
- **Archive strategy**: Move completed programme data older than the retention period to cold storage (S3 + Parquet) while maintaining summary records

### Data Migration from Incumbents

The normalised schema maps cleanly to CSV imports from Salesforce, Apricot, and ETO because each table represents a single entity type. The `import_batches` table provides full audit trails for data migration, and the generic structure of `form_definitions` allows mapping diverse source system forms into the unified schema.

### Schema Evolution

Using Flyway or Alembic for version-controlled migrations ensures that self-hosted deployments can upgrade incrementally. Each migration script is tested against a copy of production data before deployment. Backward-compatible changes (adding nullable columns, new tables) are preferred over breaking changes.
