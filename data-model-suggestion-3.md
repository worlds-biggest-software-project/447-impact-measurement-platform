# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

## Overview

This model uses PostgreSQL as a single database engine but strategically combines normalised relational tables for stable, well-understood entities with JSONB columns for flexible, variable, and evolving data structures. The core insight is that an impact measurement platform has two fundamentally different data profiles:

1. **Stable domain entities** -- organisations, programmes, participants, enrolments, indicators -- whose structure is well-defined and unlikely to change. These benefit from relational normalisation with foreign keys, constraints, and indexes.

2. **Variable and evolving data** -- form definitions, survey responses, theory of change configurations, framework metric structures, qualitative analysis results, and report configurations -- whose structure varies by programme, organisation, and over time. These benefit from schema-flexible JSONB storage.

PostgreSQL's JSONB support is mature enough to make this a single-technology solution. GIN indexes on JSONB columns provide fast querying. Partial indexes can target specific JSON structures. And the ability to mix SQL joins with JSON operators means a single query can traverse both relational and document data.

This hybrid approach directly addresses the key weakness of the fully normalised model (Suggestion 1): the rigidity that forces schema changes whenever a new question type, indicator category, or assessment format is introduced. In the nonprofit impact measurement domain, where every programme may have unique assessment instruments and every funder requires different reporting formats, this flexibility is operationally critical.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Database | PostgreSQL 16+ | Single engine for both relational and document storage; JSONB with GIN indexes |
| JSONB Validation | pg_jsonschema extension or application-layer | Validate JSONB documents against JSON Schema at write time |
| Migration | Alembic or Flyway | Version-controlled migrations for relational portions |
| ORM | Prisma (with Json fields) or SQLAlchemy | Both support JSONB natively |
| Search | PostgreSQL tsvector + JSONB text extraction | Full-text search across both relational and JSONB content |
| Caching | Redis | Cache computed dashboard data and report snapshots |

---

## Complete Schema Definition

### Tier 1: Normalised Relational Tables (Stable Entities)

```sql
-- ============================================================
-- ORGANISATIONS AND USERS
-- These entities have fixed, well-understood structures.
-- ============================================================

CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    -- Settings stored as JSONB for flexibility (branding, defaults, etc.)
    settings JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example settings structure:
    -- {
    --   "branding": { "logo_url": "...", "primary_color": "#...", "report_footer": "..." },
    --   "defaults": { "timezone": "UTC", "currency": "USD", "language": "en" },
    --   "privacy": {
    --     "gdpr_data_controller": true,
    --     "data_retention_months": 84,
    --     "pseudonymisation_enabled": true,
    --     "encryption_key_provider": "aws_kms"
    --   },
    --   "integrations": {
    --     "allowed_types": ["salesforce", "kobotoolbox", "csv"]
    --   }
    -- }
    country_code CHAR(2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);

CREATE INDEX idx_org_slug ON organisations(slug);
-- GIN index on settings for querying organisation configurations
CREATE INDEX idx_org_settings ON organisations USING GIN (settings);

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
    -- User preferences as JSONB (notification settings, dashboard layout, etc.)
    preferences JSONB NOT NULL DEFAULT '{}'::JSONB,
    is_active BOOLEAN DEFAULT TRUE,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(organisation_id, email)
);

CREATE INDEX idx_users_org ON users(organisation_id);
CREATE INDEX idx_users_email ON users(email);

-- ============================================================
-- PARTICIPANTS AND CONSENT
-- Core identity tracking with encrypted PII.
-- ============================================================

CREATE TABLE participants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    pseudonym_id VARCHAR(100) NOT NULL,
    -- PII stored encrypted
    pii_encrypted BYTEA,  -- encrypted JSON blob containing name, dob, email, phone
    -- Demographic data for outcome disaggregation (not PII)
    demographics JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example demographics structure:
    -- {
    --   "gender": "female",
    --   "age_group": "25-34",
    --   "ethnicity": "Hispanic/Latino",
    --   "primary_language": "es",
    --   "geographic_region": "Northeast",
    --   "postcode_area": "SW1",
    --   "disability_status": "none_disclosed",
    --   "household_size": 4,
    --   "income_bracket": "below_30k",
    --   "education_level": "high_school",
    --   "employment_status": "part_time",
    --   "custom_fields": {
    --     "veteran_status": true,
    --     "housing_type": "rental"
    --   }
    -- }
    -- Deduplication hashes
    dedup_hashes JSONB,
    -- Example: { "name_phonetic": "abc123", "dob": "def456", "location": "ghi789" }
    dedup_cluster_id UUID,
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
CREATE INDEX idx_participants_status ON participants(organisation_id, status);
-- GIN index on demographics for disaggregated outcome queries
CREATE INDEX idx_participants_demographics ON participants USING GIN (demographics);
-- Partial GIN index on dedup hashes
CREATE INDEX idx_participants_dedup ON participants USING GIN (dedup_hashes)
    WHERE dedup_hashes IS NOT NULL;

CREATE TABLE participant_consents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    participant_id UUID NOT NULL REFERENCES participants(id) ON DELETE CASCADE,
    consent_type VARCHAR(100) NOT NULL,
    granted BOOLEAN NOT NULL,
    -- Consent details as JSONB (method, document, legal basis, etc.)
    details JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "method": "digital_signature",
    --   "document_url": "https://...",
    --   "legal_basis": "consent",
    --   "parent_guardian": false,
    --   "witness": "Jane Smith"
    -- }
    granted_at TIMESTAMPTZ,
    withdrawn_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    recorded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_consents_participant ON participant_consents(participant_id);

-- ============================================================
-- PROGRAMMES AND ENROLMENTS
-- Stable relational structure with JSONB for variable config.
-- ============================================================

CREATE TABLE programmes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    programme_type VARCHAR(100),
    status VARCHAR(50) DEFAULT 'draft' CHECK (status IN (
        'draft', 'active', 'paused', 'completed', 'archived'
    )),
    start_date DATE,
    end_date DATE,
    -- Programme configuration stored as JSONB
    config JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example config:
    -- {
    --   "target_participant_count": 200,
    --   "budget": { "amount": 150000, "currency": "USD" },
    --   "measurement_schedule": {
    --     "baseline": { "timing": "enrollment", "window_days": 14 },
    --     "midpoint": { "timing": "month_6", "window_days": 30 },
    --     "endline": { "timing": "exit", "window_days": 14 },
    --     "followups": [
    --       { "name": "3_month_followup", "months_after_exit": 3, "window_days": 30 },
    --       { "name": "6_month_followup", "months_after_exit": 6, "window_days": 30 }
    --     ]
    --   },
    --   "eligibility_criteria": [
    --     { "field": "age", "operator": ">=", "value": 18 },
    --     { "field": "geographic_region", "operator": "in", "value": ["Northeast", "Midwest"] }
    --   ],
    --   "tags": ["workforce", "youth", "urban"]
    -- }
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_programmes_org ON programmes(organisation_id);
CREATE INDEX idx_programmes_status ON programmes(status);
CREATE INDEX idx_programmes_config ON programmes USING GIN (config);

CREATE TABLE cohorts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    start_date DATE,
    end_date DATE,
    is_control_group BOOLEAN DEFAULT FALSE,
    metadata JSONB NOT NULL DEFAULT '{}'::JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_cohorts_programme ON cohorts(programme_id);

CREATE TABLE programme_enrolments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    participant_id UUID NOT NULL REFERENCES participants(id),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    cohort_id UUID REFERENCES cohorts(id),
    enrolment_date DATE NOT NULL,
    exit_date DATE,
    exit_reason VARCHAR(100),
    status VARCHAR(50) DEFAULT 'enrolled' CHECK (status IN (
        'enrolled', 'active', 'on_hold', 'completed',
        'withdrawn', 'lost_to_followup'
    )),
    -- Enrolment-specific data as JSONB (intake answers, referral info, etc.)
    intake_data JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "referral_source": "community_partner",
    --   "referral_partner": "Local Youth Center",
    --   "intake_assessment_score": 45,
    --   "presenting_needs": ["employment", "housing", "mental_health"],
    --   "case_worker_id": "uuid-here",
    --   "priority_level": "high"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_enrolments_participant ON programme_enrolments(participant_id);
CREATE INDEX idx_enrolments_programme ON programme_enrolments(programme_id);
CREATE INDEX idx_enrolments_status ON programme_enrolments(programme_id, status);
CREATE INDEX idx_enrolments_intake ON programme_enrolments USING GIN (intake_data);
```

### Tier 2: JSONB-Heavy Tables (Variable/Evolving Structures)

```sql
-- ============================================================
-- THEORY OF CHANGE
-- The entire ToC graph is stored as a JSONB document because:
-- 1. Each programme's ToC has a unique structure
-- 2. ToC editing is a single-user operation (no concurrent writes)
-- 3. The ToC is typically loaded and saved as a whole document
-- 4. Visual layout data (positions, colors) is inherently unstructured
-- ============================================================

CREATE TABLE theories_of_change (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id UUID NOT NULL REFERENCES programmes(id) ON DELETE CASCADE,
    version INTEGER NOT NULL DEFAULT 1,
    name VARCHAR(255) NOT NULL,
    is_current BOOLEAN DEFAULT TRUE,
    -- The entire ToC graph as a JSONB document
    graph JSONB NOT NULL,
    -- Example structure:
    -- {
    --   "narrative": "Our workforce development programme...",
    --   "nodes": [
    --     {
    --       "id": "node-1",
    --       "type": "input",
    --       "title": "Trained facilitators",
    --       "description": "15 certified career coaches",
    --       "position": { "x": 100, "y": 200 },
    --       "style": { "color": "#4A90D9", "icon": "people" },
    --       "indicators": ["ind-uuid-1", "ind-uuid-2"],
    --       "metadata": { "responsible_team": "Training" }
    --     },
    --     {
    --       "id": "node-2",
    --       "type": "activity",
    --       "title": "Weekly career workshops",
    --       "description": "2-hour sessions covering resume, interview, networking",
    --       "position": { "x": 300, "y": 200 },
    --       "indicators": ["ind-uuid-3"],
    --       "dosage": { "frequency": "weekly", "duration_hours": 2, "total_sessions": 12 }
    --     },
    --     {
    --       "id": "node-3",
    --       "type": "outcome",
    --       "title": "Increased employability skills",
    --       "description": "Measurable improvement in job-readiness assessment",
    --       "position": { "x": 500, "y": 200 },
    --       "indicators": ["ind-uuid-4", "ind-uuid-5"],
    --       "timeframe": "3_months"
    --     },
    --     {
    --       "id": "node-4",
    --       "type": "impact",
    --       "title": "Sustained employment",
    --       "description": "Participants maintain employment for 6+ months",
    --       "position": { "x": 700, "y": 200 },
    --       "indicators": ["ind-uuid-6"],
    --       "timeframe": "12_months"
    --     }
    --   ],
    --   "edges": [
    --     {
    --       "id": "edge-1",
    --       "from": "node-1",
    --       "to": "node-2",
    --       "label": "Delivers",
    --       "assumption": "Facilitators maintain consistent quality",
    --       "evidence_strength": "moderate"
    --     },
    --     {
    --       "id": "edge-2",
    --       "from": "node-2",
    --       "to": "node-3",
    --       "label": "Produces",
    --       "assumption": "Participants attend at least 8 of 12 sessions",
    --       "evidence_strength": "strong"
    --     },
    --     {
    --       "id": "edge-3",
    --       "from": "node-3",
    --       "to": "node-4",
    --       "label": "Leads to",
    --       "assumption": "Local job market has sufficient openings",
    --       "evidence_strength": "weak"
    --     }
    --   ],
    --   "assumptions": [
    --     {
    --       "id": "assumption-1",
    --       "edge_id": "edge-2",
    --       "description": "Participants attend at least 8 of 12 sessions",
    --       "status": "supported",
    --       "evidence": "Average attendance is 9.2 sessions (Q3 2025 data)"
    --     }
    --   ]
    -- }
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(programme_id, version)
);

CREATE INDEX idx_toc_programme ON theories_of_change(programme_id);
-- GIN index for searching within ToC graph structures
CREATE INDEX idx_toc_graph ON theories_of_change USING GIN (graph);

-- ============================================================
-- INDICATORS
-- Hybrid: relational core with JSONB for variable configuration.
-- ============================================================

CREATE TABLE indicators (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    code VARCHAR(50),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    measurement_type VARCHAR(50) NOT NULL CHECK (measurement_type IN (
        'numeric', 'percentage', 'scale', 'boolean', 'categorical', 'text', 'composite'
    )),
    direction VARCHAR(20) DEFAULT 'increase' CHECK (direction IN (
        'increase', 'decrease', 'maintain', 'target'
    )),
    is_active BOOLEAN DEFAULT TRUE,
    -- Variable configuration as JSONB
    config JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example config for a scale indicator:
    -- {
    --   "unit": "score",
    --   "scale": { "min": 1, "max": 10, "labels": { "1": "Not at all", "10": "Completely" } },
    --   "collection_frequency": "quarterly",
    --   "data_source": "participant_survey",
    --   "thresholds": {
    --     "clinically_significant_change": 2,
    --     "target": 7
    --   },
    --   "disaggregation_dimensions": ["gender", "age_group", "ethnicity"],
    --   "statistical_method": {
    --     "change_calculation": "reliable_change_index",
    --     "rci_se_measurement": 0.8
    --   }
    -- }
    --
    -- Example config for a composite indicator:
    -- {
    --   "composite_formula": "average",
    --   "component_indicators": ["ind-uuid-1", "ind-uuid-2", "ind-uuid-3"],
    --   "weights": [0.4, 0.3, 0.3]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_indicators_org ON indicators(organisation_id);
CREATE INDEX idx_indicators_code ON indicators(organisation_id, code);
CREATE INDEX idx_indicators_config ON indicators USING GIN (config);

-- ============================================================
-- FORM DEFINITIONS
-- Entirely JSONB-based form structure. This is the biggest win
-- of the hybrid approach: forms vary enormously between programmes
-- and organisations, making a normalised question/choice/section
-- schema extremely rigid.
-- ============================================================

CREATE TABLE form_definitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    programme_id UUID REFERENCES programmes(id),
    name VARCHAR(255) NOT NULL,
    form_type VARCHAR(50) NOT NULL CHECK (form_type IN (
        'baseline_assessment', 'midpoint_assessment', 'endline_assessment',
        'followup_survey', 'intake_form', 'exit_form',
        'feedback_360', 'case_note', 'custom'
    )),
    version INTEGER NOT NULL DEFAULT 1,
    status VARCHAR(50) DEFAULT 'draft' CHECK (status IN (
        'draft', 'published', 'archived'
    )),
    -- The entire form definition as JSONB
    -- This mirrors the XLSForm/ODK approach where forms are defined
    -- as structured documents rather than relational entities.
    definition JSONB NOT NULL,
    -- Example structure:
    -- {
    --   "title": "Baseline Employability Assessment",
    --   "description": "Administered at programme intake",
    --   "estimated_duration_minutes": 20,
    --   "sections": [
    --     {
    --       "id": "sec-1",
    --       "title": "Demographics",
    --       "description": "Background information",
    --       "questions": [
    --         {
    --           "id": "q-1",
    --           "type": "single_choice",
    --           "text": "What is your current employment status?",
    --           "help_text": "Select the option that best describes...",
    --           "required": true,
    --           "indicator_id": "ind-uuid-1",
    --           "choices": [
    --             { "value": "employed_full", "label": "Employed full-time" },
    --             { "value": "employed_part", "label": "Employed part-time" },
    --             { "value": "unemployed_seeking", "label": "Unemployed, seeking work" },
    --             { "value": "unemployed_not_seeking", "label": "Unemployed, not seeking work" },
    --             { "value": "student", "label": "Student" },
    --             { "value": "retired", "label": "Retired" }
    --           ]
    --         },
    --         {
    --           "id": "q-2",
    --           "type": "number",
    --           "text": "How many months have you been in your current situation?",
    --           "required": false,
    --           "validation": { "min": 0, "max": 600 },
    --           "display_condition": { "question": "q-1", "operator": "!=", "value": "retired" }
    --         },
    --         {
    --           "id": "q-3",
    --           "type": "likert_scale",
    --           "text": "How confident do you feel about your job interview skills?",
    --           "required": true,
    --           "indicator_id": "ind-uuid-2",
    --           "scale": {
    --             "min": 1, "max": 5,
    --             "labels": {
    --               "1": "Not at all confident",
    --               "3": "Somewhat confident",
    --               "5": "Very confident"
    --             }
    --           }
    --         }
    --       ]
    --     },
    --     {
    --       "id": "sec-2",
    --       "title": "Skills Self-Assessment",
    --       "description": "Rate your current skill levels",
    --       "repeatable": false,
    --       "questions": [
    --         {
    --           "id": "q-4",
    --           "type": "matrix",
    --           "text": "Rate your confidence in each area:",
    --           "rows": [
    --             { "id": "r-1", "label": "Resume writing", "indicator_id": "ind-uuid-3" },
    --             { "id": "r-2", "label": "Networking", "indicator_id": "ind-uuid-4" },
    --             { "id": "r-3", "label": "Time management", "indicator_id": "ind-uuid-5" }
    --           ],
    --           "columns": [
    --             { "value": 1, "label": "Poor" },
    --             { "value": 2, "label": "Fair" },
    --             { "value": 3, "label": "Good" },
    --             { "value": 4, "label": "Very Good" },
    --             { "value": 5, "label": "Excellent" }
    --           ]
    --         },
    --         {
    --           "id": "q-5",
    --           "type": "textarea",
    --           "text": "What are your main goals for this programme?",
    --           "required": true,
    --           "max_length": 2000
    --         }
    --       ]
    --     }
    --   ],
    --   "scoring": {
    --     "method": "sum",
    --     "scored_questions": ["q-3", "q-4"],
    --     "max_score": 25
    --   },
    --   "submission_rules": {
    --     "allow_save_draft": true,
    --     "require_gps": false,
    --     "auto_submit_on_complete": false
    --   }
    -- }
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_forms_org ON form_definitions(organisation_id);
CREATE INDEX idx_forms_programme ON form_definitions(programme_id);
CREATE INDEX idx_forms_type ON form_definitions(form_type);
-- GIN index for searching within form definitions (finding forms that use specific indicators)
CREATE INDEX idx_forms_definition ON form_definitions USING GIN (definition);

-- ============================================================
-- FORM SUBMISSIONS
-- Relational metadata with JSONB answers. This is the critical
-- design choice: the submission record is relational (who, when,
-- where, status) but the actual answers are a JSONB document
-- that mirrors the form definition structure.
-- ============================================================

CREATE TABLE form_submissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_id UUID NOT NULL REFERENCES form_definitions(id),
    enrolment_id UUID NOT NULL REFERENCES programme_enrolments(id),
    -- Relational fields for core querying
    submitted_by UUID REFERENCES users(id),
    submission_type VARCHAR(50) CHECK (submission_type IN (
        'self_report', 'staff_administered', 'peer_assessment',
        'supervisor_assessment', 'automated'
    )),
    status VARCHAR(50) DEFAULT 'draft' CHECK (status IN (
        'draft', 'submitted', 'validated', 'flagged', 'rejected'
    )),
    collection_method VARCHAR(50),
    submitted_at TIMESTAMPTZ,
    validated_at TIMESTAMPTZ,
    validated_by UUID REFERENCES users(id),
    -- Answers stored as JSONB document
    answers JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example answers structure:
    -- {
    --   "q-1": { "value": "unemployed_seeking", "type": "choice" },
    --   "q-2": { "value": 8, "type": "number" },
    --   "q-3": { "value": 2, "type": "scale" },
    --   "q-4": {
    --     "type": "matrix",
    --     "values": {
    --       "r-1": 3,
    --       "r-2": 2,
    --       "r-3": 4
    --     }
    --   },
    --   "q-5": {
    --     "value": "I want to find a stable job in healthcare...",
    --     "type": "text"
    --   },
    --   "_metadata": {
    --     "total_score": 11,
    --     "completion_time_seconds": 840,
    --     "device_id": "mobile-abc123",
    --     "gps": { "lat": 40.7128, "lng": -74.0060, "accuracy_m": 15 }
    --   }
    -- }
    -- Offline sync metadata
    sync_metadata JSONB,
    -- { "device_id": "...", "local_timestamp": "...", "sync_status": "synced" }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_submissions_form ON form_submissions(form_id);
CREATE INDEX idx_submissions_enrolment ON form_submissions(enrolment_id);
CREATE INDEX idx_submissions_date ON form_submissions(submitted_at);
CREATE INDEX idx_submissions_status ON form_submissions(status);
-- GIN index on answers for querying specific answer values across submissions
CREATE INDEX idx_submissions_answers ON form_submissions USING GIN (answers);
```

### Tier 2 Continued: Outcome Measurement

```sql
-- ============================================================
-- OUTCOME RECORDS
-- Hybrid: relational keys for joining and aggregation, JSONB
-- for variable measurement data and source tracing.
-- ============================================================

CREATE TABLE outcome_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Relational keys for fast joining and aggregation
    enrolment_id UUID NOT NULL REFERENCES programme_enrolments(id),
    indicator_id UUID NOT NULL REFERENCES indicators(id),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    -- Core measurement fields (relational for aggregation queries)
    measurement_period VARCHAR(50) NOT NULL,
    measurement_date DATE NOT NULL,
    value_numeric NUMERIC,  -- primary numeric value (extracted for aggregation)
    -- Full measurement data as JSONB (for values that don't fit a single numeric)
    measurement_data JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example for a simple numeric:
    -- { "value": 7.5, "unit": "score", "out_of": 10 }
    --
    -- Example for a categorical:
    -- { "value": "employed_full", "label": "Employed full-time", "is_positive": true }
    --
    -- Example for a composite:
    -- {
    --   "composite_value": 72.5,
    --   "components": {
    --     "resume_skills": { "value": 8, "weight": 0.4 },
    --     "interview_skills": { "value": 6, "weight": 0.3 },
    --     "networking": { "value": 7, "weight": 0.3 }
    --   }
    -- }
    --
    -- Example for distance-travelled context:
    -- {
    --   "value": 7,
    --   "baseline_value": 3,
    --   "change": 4,
    --   "percentage_change": 133.3,
    --   "standardised_change": 1.8,
    --   "reliable_change_index": 2.4,
    --   "is_reliable_change": true,
    --   "direction": "improved"
    -- }
    -- Source tracing
    source JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- {
    --   "type": "form_submission",
    --   "form_submission_id": "uuid",
    --   "question_id": "q-3",
    --   "form_name": "Baseline Employability Assessment",
    --   "collected_by": "user-uuid",
    --   "collection_method": "mobile_online"
    -- }
    is_validated BOOLEAN DEFAULT FALSE,
    validation JSONB,
    -- { "validated_by": "user-uuid", "validated_at": "...", "notes": "..." }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Core relational indexes for aggregation queries
CREATE INDEX idx_outcomes_enrolment ON outcome_records(enrolment_id);
CREATE INDEX idx_outcomes_indicator ON outcome_records(indicator_id);
CREATE INDEX idx_outcomes_programme ON outcome_records(programme_id);
CREATE INDEX idx_outcomes_org ON outcome_records(organisation_id);
CREATE INDEX idx_outcomes_date ON outcome_records(measurement_date);
CREATE INDEX idx_outcomes_period ON outcome_records(measurement_period);
-- Composite index for distance-travelled: baseline vs endline for same person/indicator
CREATE INDEX idx_outcomes_distance ON outcome_records(
    enrolment_id, indicator_id, measurement_period
);
-- Composite index for programme-level aggregation
CREATE INDEX idx_outcomes_programme_agg ON outcome_records(
    programme_id, indicator_id, measurement_period, value_numeric
);
-- GIN index on measurement data for complex querying
CREATE INDEX idx_outcomes_measurement ON outcome_records USING GIN (measurement_data);

-- ============================================================
-- PROGRAMME OUTCOME SUMMARIES
-- Pre-computed aggregates, refreshed periodically or on demand.
-- ============================================================

CREATE TABLE programme_outcome_summaries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    indicator_id UUID NOT NULL REFERENCES indicators(id),
    cohort_id UUID REFERENCES cohorts(id),
    reporting_period_start DATE NOT NULL,
    reporting_period_end DATE NOT NULL,
    -- Aggregate statistics as JSONB for flexibility
    summary JSONB NOT NULL,
    -- {
    --   "participant_count": 150,
    --   "measured_count": 142,
    --   "response_rate": 94.7,
    --   "baseline": {
    --     "mean": 3.2, "median": 3.0, "std_dev": 1.1,
    --     "min": 1, "max": 7, "count": 142
    --   },
    --   "endline": {
    --     "mean": 6.8, "median": 7.0, "std_dev": 1.4,
    --     "min": 2, "max": 10, "count": 138
    --   },
    --   "change": {
    --     "mean_change": 3.6, "median_change": 4.0,
    --     "std_dev_change": 1.3, "effect_size_cohens_d": 2.88
    --   },
    --   "outcomes": {
    --     "improved": { "count": 118, "percentage": 85.5 },
    --     "maintained": { "count": 12, "percentage": 8.7 },
    --     "declined": { "count": 8, "percentage": 5.8 }
    --   },
    --   "reliable_change": {
    --     "reliably_improved": { "count": 95, "percentage": 68.8 },
    --     "no_reliable_change": { "count": 39, "percentage": 28.3 },
    --     "reliably_declined": { "count": 4, "percentage": 2.9 }
    --   },
    --   "disaggregation": {
    --     "by_gender": {
    --       "female": { "count": 82, "mean_change": 3.8, "improved_pct": 87.8 },
    --       "male": { "count": 56, "mean_change": 3.3, "improved_pct": 82.1 }
    --     },
    --     "by_age_group": {
    --       "18-24": { "count": 45, "mean_change": 4.1, "improved_pct": 91.1 },
    --       "25-34": { "count": 53, "mean_change": 3.4, "improved_pct": 84.9 }
    --     }
    --   },
    --   "target": { "value": 7, "achieved": false, "gap": 0.2 }
    -- }
    calculated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(programme_id, indicator_id, cohort_id, reporting_period_start, reporting_period_end)
);

CREATE INDEX idx_prog_summary_programme ON programme_outcome_summaries(programme_id);
CREATE INDEX idx_prog_summary_indicator ON programme_outcome_summaries(indicator_id);
```

### Tier 2 Continued: Framework Alignment, Qualitative Analysis, and Reporting

```sql
-- ============================================================
-- FRAMEWORK ALIGNMENT
-- Frameworks and their metrics stored as JSONB documents
-- because they come from external sources with varying structures.
-- ============================================================

CREATE TABLE frameworks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    version VARCHAR(50),
    framework_type VARCHAR(50) CHECK (framework_type IN (
        'iris_plus', 'sdg', 'funder_template', 'government',
        'custom', 'sroi', 'social_value'
    )),
    is_system BOOLEAN DEFAULT FALSE,
    -- The full framework taxonomy as JSONB
    taxonomy JSONB NOT NULL,
    -- Example for SDGs:
    -- {
    --   "goals": [
    --     {
    --       "code": "SDG-1",
    --       "name": "No Poverty",
    --       "targets": [
    --         {
    --           "code": "SDG-1.1",
    --           "name": "Eradicate extreme poverty",
    --           "indicators": [
    --             { "code": "1.1.1", "name": "Population below poverty line (%)" }
    --           ]
    --         }
    --       ]
    --     }
    --   ]
    -- }
    --
    -- Example for IRIS+:
    -- {
    --   "categories": [
    --     {
    --       "code": "PI",
    --       "name": "Product Impact",
    --       "themes": [
    --         {
    --           "code": "PI-EMPLOYMENT",
    --           "name": "Employment Generation",
    --           "metrics": [
    --             { "code": "PI1234", "name": "Full-time Employees", "type": "numeric", "unit": "count" },
    --             { "code": "PI5678", "name": "Jobs Created", "type": "numeric", "unit": "count" }
    --           ]
    --         }
    --       ]
    --     }
    --   ]
    -- }
    source_url VARCHAR(500),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_frameworks_type ON frameworks(framework_type);
CREATE INDEX idx_frameworks_taxonomy ON frameworks USING GIN (taxonomy);

-- Mapping internal indicators to framework metrics
CREATE TABLE indicator_framework_mappings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    indicator_id UUID NOT NULL REFERENCES indicators(id) ON DELETE CASCADE,
    framework_id UUID NOT NULL REFERENCES frameworks(id) ON DELETE CASCADE,
    -- The path within the framework taxonomy to the mapped metric
    framework_metric_path TEXT NOT NULL,  -- e.g. "goals[0].targets[2].indicators[1]"
    framework_metric_code VARCHAR(100),   -- e.g. "SDG-1.1.1" or "PI1234"
    framework_metric_name VARCHAR(255),
    mapping_confidence VARCHAR(20) DEFAULT 'exact',
    mapping_notes TEXT,
    mapped_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(indicator_id, framework_id, framework_metric_code)
);

CREATE INDEX idx_fw_map_indicator ON indicator_framework_mappings(indicator_id);
CREATE INDEX idx_fw_map_framework ON indicator_framework_mappings(framework_id);

-- ============================================================
-- QUALITATIVE EVIDENCE AND AI ANALYSIS
-- Text content stored relationally; analysis results as JSONB.
-- ============================================================

CREATE TABLE qualitative_evidence (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    enrolment_id UUID REFERENCES programme_enrolments(id),
    programme_id UUID REFERENCES programmes(id),
    evidence_type VARCHAR(50) NOT NULL,
    content TEXT NOT NULL,
    language VARCHAR(10) DEFAULT 'en',
    word_count INTEGER,
    recorded_date DATE,
    author_id UUID REFERENCES users(id),
    -- Source tracing
    source JSONB,
    -- { "form_submission_id": "uuid", "question_id": "q-5" }
    -- AI analysis results stored as JSONB (evolves with model improvements)
    analysis JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- {
    --   "thematic_codes": [
    --     { "code": "EMPL-BARRIER", "label": "Employment Barriers",
    --       "confidence": 0.92, "assigned_by": "ai", "excerpt": "..." },
    --     { "code": "GOAL-HEALTHCARE", "label": "Healthcare Career Goal",
    --       "confidence": 0.87, "assigned_by": "ai", "excerpt": "..." }
    --   ],
    --   "sentiment": {
    --     "overall": "positive",
    --     "score": 0.65,
    --     "model_version": "gpt-4o-2025-05"
    --   },
    --   "key_phrases": ["stable job", "healthcare", "career change"],
    --   "summary": "Participant expresses strong motivation to transition...",
    --   "language_detected": "en",
    --   "analysed_at": "2026-01-15T10:30:00Z"
    -- }
    -- Full text search
    content_tsv TSVECTOR GENERATED ALWAYS AS (to_tsvector('english', content)) STORED,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_qual_org ON qualitative_evidence(organisation_id);
CREATE INDEX idx_qual_programme ON qualitative_evidence(programme_id);
CREATE INDEX idx_qual_enrolment ON qualitative_evidence(enrolment_id);
CREATE INDEX idx_qual_tsv ON qualitative_evidence USING GIN (content_tsv);
CREATE INDEX idx_qual_analysis ON qualitative_evidence USING GIN (analysis);

-- ============================================================
-- ACTIVITIES AND ATTENDANCE
-- Hybrid: relational for scheduling, JSONB for variable details.
-- ============================================================

CREATE TABLE activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    name VARCHAR(255) NOT NULL,
    scheduled_date DATE,
    scheduled_time TIME,
    -- Variable activity details as JSONB
    details JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- {
    --   "type": "workshop",
    --   "topic": "Resume Writing",
    --   "duration_minutes": 120,
    --   "location": { "name": "Room 204", "address": "123 Main St", "virtual_link": null },
    --   "facilitator_id": "user-uuid",
    --   "max_capacity": 25,
    --   "toc_node_id": "node-2",
    --   "materials": ["handout-resume-template.pdf"],
    --   "tags": ["employment", "skills"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_activities_programme ON activities(programme_id);
CREATE INDEX idx_activities_date ON activities(scheduled_date);
CREATE INDEX idx_activities_details ON activities USING GIN (details);

CREATE TABLE activity_attendance (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    activity_id UUID NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
    enrolment_id UUID NOT NULL REFERENCES programme_enrolments(id),
    attended BOOLEAN NOT NULL DEFAULT TRUE,
    attendance_type VARCHAR(50),
    notes TEXT,
    recorded_by UUID REFERENCES users(id),
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(activity_id, enrolment_id)
);

CREATE INDEX idx_attendance_activity ON activity_attendance(activity_id);
CREATE INDEX idx_attendance_enrolment ON activity_attendance(enrolment_id);

-- ============================================================
-- REPORTING
-- Report configurations and generated output metadata.
-- ============================================================

CREATE TABLE report_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    report_type VARCHAR(50),
    framework_id UUID REFERENCES frameworks(id),
    -- Template configuration as JSONB
    template JSONB NOT NULL,
    -- {
    --   "layout": "funder_standard",
    --   "sections": [
    --     { "type": "executive_summary", "auto_generate": true },
    --     { "type": "programme_overview", "fields": ["description", "dates", "participant_count"] },
    --     { "type": "outcome_table", "indicators": ["all"], "show_disaggregation": true },
    --     { "type": "qualitative_highlights", "max_quotes": 5, "theme_filter": null },
    --     { "type": "framework_alignment", "framework_id": "sdg-uuid", "show_mapping": true },
    --     { "type": "visualisations", "charts": [
    --       { "type": "bar", "title": "Outcomes by Indicator", "data_source": "outcome_summary" },
    --       { "type": "line", "title": "Attendance Over Time", "data_source": "attendance_trend" }
    --     ]},
    --     { "type": "appendix", "include_methodology": true }
    --   ],
    --   "branding": {
    --     "use_org_logo": true,
    --     "header_text": "Quarterly Impact Report",
    --     "footer_text": "Confidential - For funder use only"
    --   },
    --   "data_filters": {
    --     "cohort_ids": null,
    --     "date_range": "reporting_period"
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_report_templates_org ON report_templates(organisation_id);

CREATE TABLE generated_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id UUID NOT NULL REFERENCES report_templates(id),
    programme_id UUID REFERENCES programmes(id),
    generated_by UUID REFERENCES users(id),
    title VARCHAR(255) NOT NULL,
    reporting_period_start DATE,
    reporting_period_end DATE,
    format VARCHAR(20),
    file_url VARCHAR(500),
    file_size_bytes BIGINT,
    status VARCHAR(50) DEFAULT 'generating',
    -- Report data snapshot (for reproducibility)
    data_snapshot JSONB,
    -- { "generated_at": "...", "query_parameters": {...}, "data_version": "..." }
    generated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ
);

CREATE INDEX idx_reports_programme ON generated_reports(programme_id);

-- ============================================================
-- ANOMALY DETECTION
-- ============================================================

CREATE TABLE outcome_anomalies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id UUID NOT NULL REFERENCES programmes(id),
    indicator_id UUID NOT NULL REFERENCES indicators(id),
    cohort_id UUID REFERENCES cohorts(id),
    -- Anomaly details as JSONB
    anomaly JSONB NOT NULL,
    -- {
    --   "type": "unexpected_decline",
    --   "severity": "high",
    --   "description": "Cohort B employment outcomes dropped 15% below expected trajectory",
    --   "detected_value": 52.3,
    --   "expected_value": 67.8,
    --   "deviation_score": -2.3,
    --   "affected_participants": 23,
    --   "detection_model": "arima_forecast_v2",
    --   "suggested_actions": [
    --     "Review recent programme changes",
    --     "Check for external economic factors",
    --     "Interview affected cohort participants"
    --   ]
    -- }
    detected_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES users(id),
    resolution_notes TEXT
);

CREATE INDEX idx_anomalies_programme ON outcome_anomalies(programme_id);

-- ============================================================
-- INTEGRATIONS AND IMPORTS
-- ============================================================

CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    integration_type VARCHAR(50) NOT NULL,
    name VARCHAR(255) NOT NULL,
    config_encrypted BYTEA,
    is_active BOOLEAN DEFAULT TRUE,
    -- Sync state as JSONB
    sync_state JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- { "last_sync_at": "...", "last_status": "success", "records_synced": 45, "cursor": "..." }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE import_batches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    integration_id UUID REFERENCES integrations(id),
    source_type VARCHAR(50) NOT NULL,
    source_filename VARCHAR(255),
    status VARCHAR(50) DEFAULT 'pending',
    -- Import results as JSONB
    results JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- {
    --   "total_records": 500,
    --   "imported": 485,
    --   "skipped": 10,
    --   "errors": 5,
    --   "error_details": [
    --     { "row": 23, "field": "date_of_birth", "error": "Invalid date format", "value": "13/25/1990" }
    --   ],
    --   "field_mapping": { "Name": "first_name", "DOB": "date_of_birth" }
    -- }
    imported_by UUID REFERENCES users(id),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    organisation_id UUID NOT NULL,
    user_id UUID,
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(100) NOT NULL,
    entity_id UUID,
    changes JSONB,
    -- { "field": { "old": "value1", "new": "value2" } }
    request_metadata JSONB,
    -- { "ip": "...", "user_agent": "...", "endpoint": "/api/..." }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE audit_log_2026_01 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE audit_log_2026_02 PARTITION OF audit_log
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... continue for each month

CREATE INDEX idx_audit_org ON audit_log(organisation_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Key Design Decisions

### Where JSONB vs. Where Relational

The guiding principle: **If you need to JOIN on it, aggregate it, or enforce referential integrity across it, make it relational. If it varies per instance, evolves frequently, or is loaded/saved as a unit, make it JSONB.**

| Entity | Relational | JSONB | Rationale |
|--------|-----------|-------|-----------|
| Organisation identity | id, name, slug | settings, branding | Core identity is stable; settings change constantly |
| Participant identity | id, pseudonym, status | demographics, dedup_hashes | FK references need the id; demographic fields vary by org |
| Programme | id, name, status, dates | config, measurement_schedule | Status drives queries; config varies per programme |
| Enrolment | participant_id, programme_id, status | intake_data | FK chain is critical; intake questions vary |
| Theory of Change | programme_id, version | entire graph structure | Loaded as a single document in the visual editor |
| Indicator | id, code, name, measurement_type | config, thresholds, scale | Joins and aggregation use the id; config varies by type |
| Form definition | id, name, form_type, status | entire definition | Form structure is arbitrary per programme |
| Form submission | id, form_id, enrolment_id, status | answers | Submission metadata drives queries; answers mirror form structure |
| Outcome record | enrolment_id, indicator_id, date, value_numeric | measurement_data, source | Numeric value extracted for aggregation; rich context in JSONB |
| Framework | id, name, type | taxonomy | Framework structures vary enormously |
| Qualitative evidence | id, organisation_id, type | analysis results | AI analysis schema evolves with model improvements |

### JSONB Validation Strategy

Without constraints, JSONB columns can accumulate inconsistent data. The recommended approach:

1. **Application-layer validation**: JSON Schema validation in the API layer before writes
2. **PostgreSQL CHECK constraints** for critical structure:

```sql
-- Ensure outcome_records always have a value in measurement_data
ALTER TABLE outcome_records ADD CONSTRAINT chk_measurement_data
    CHECK (measurement_data ? 'value' OR measurement_data ? 'composite_value' OR measurement_data ? 'values');

-- Ensure form definitions have at least one section
ALTER TABLE form_definitions ADD CONSTRAINT chk_form_has_sections
    CHECK (jsonb_array_length(definition->'sections') > 0);
```

3. **pg_jsonschema extension** for full JSON Schema validation on critical tables (if available in deployment)

### Extracted Columns for Performance

The `value_numeric` column on `outcome_records` is a deliberate denormalisation. The full measurement is in `measurement_data` (JSONB), but the primary numeric value is extracted into a typed column because:

- Aggregate queries (`AVG`, `SUM`, `PERCENTILE_CONT`) on JSONB are slow
- Indexes on JSONB numeric paths are less efficient than B-tree indexes on numeric columns
- The `value_numeric` column enables the standard relational indexes used for distance-travelled and programme summary queries

---

## Example Queries

### Distance Travelled for a Participant

```sql
SELECT
    i.name AS indicator_name,
    baseline.value_numeric AS baseline_value,
    endline.value_numeric AS endline_value,
    endline.value_numeric - baseline.value_numeric AS absolute_change,
    CASE WHEN baseline.value_numeric > 0
         THEN ROUND(((endline.value_numeric - baseline.value_numeric) / baseline.value_numeric * 100), 1)
         ELSE NULL END AS percentage_change,
    endline.measurement_data->'standardised_change' AS cohens_d
FROM outcome_records baseline
JOIN outcome_records endline ON baseline.enrolment_id = endline.enrolment_id
    AND baseline.indicator_id = endline.indicator_id
JOIN indicators i ON i.id = baseline.indicator_id
WHERE baseline.enrolment_id = $1
    AND baseline.measurement_period = 'baseline'
    AND endline.measurement_period = 'endline';
```

### Programme Outcomes Disaggregated by Demographic

```sql
SELECT
    p.demographics->>'gender' AS gender,
    i.name AS indicator,
    COUNT(*) AS participant_count,
    AVG(o.value_numeric) AS mean_value,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY o.value_numeric) AS median_value
FROM outcome_records o
JOIN programme_enrolments pe ON pe.id = o.enrolment_id
JOIN participants p ON p.id = pe.participant_id
JOIN indicators i ON i.id = o.indicator_id
WHERE o.programme_id = $1
    AND o.measurement_period = 'endline'
GROUP BY p.demographics->>'gender', i.name;
```

### Find Forms Using a Specific Indicator

```sql
SELECT fd.id, fd.name, fd.form_type
FROM form_definitions fd
WHERE fd.definition @> '{"sections": [{"questions": [{"indicator_id": "target-indicator-uuid"}]}]}'::JSONB;
```

---

## Pros and Cons

### Pros

1. **Single technology stack**: Everything runs on PostgreSQL. No additional database engines to operate, monitor, or back up. This dramatically simplifies deployment for resource-constrained nonprofits and self-hosted environments.

2. **Form flexibility without schema changes**: New assessment instruments, question types, and scoring methods can be deployed by writing new JSONB structures -- no database migration required. This is critical for a platform where every programme may have unique data collection instruments.

3. **Relational integrity where it matters**: The FK chain from participant through enrolment to outcome is fully enforced. Evidence continuity is guaranteed by the database, not by application code.

4. **Efficient aggregation**: Extracted numeric values (`value_numeric` on outcome_records) enable fast SQL aggregation while the full measurement context lives in JSONB. Best of both worlds.

5. **Demographics flexibility**: Different organisations track different demographic dimensions. JSONB demographics with GIN indexes allow any combination without schema changes, while still supporting disaggregated outcome queries via JSON operators.

6. **Framework import simplicity**: External frameworks (IRIS+, SDGs) can be imported as-is into JSONB taxonomy columns without transforming their hierarchical structure into relational tables.

7. **AI analysis evolution**: As AI models improve, the structure of thematic coding, sentiment analysis, and clustering results will change. JSONB `analysis` columns absorb these changes without migrations.

8. **Theory of Change as a document**: The ToC graph -- nodes, edges, positions, assumptions -- is naturally a document that the visual editor loads and saves as a unit. JSONB storage eliminates the impedance mismatch of decomposing it into relational tables.

### Cons

1. **Weaker data integrity on JSONB columns**: Foreign key relationships cannot be enforced within JSONB. If an `indicator_id` referenced in a form definition's JSONB is deleted, the reference becomes dangling. Application-layer validation must compensate.

2. **Query complexity**: Queries that mix relational joins with JSONB operators are harder to write, read, and optimise than pure relational queries. Developers need proficiency with PostgreSQL's JSON functions (`->`, `->>`, `@>`, `jsonb_array_elements`).

3. **Schema documentation burden**: With a normalised schema, the DDL IS the documentation. With JSONB columns, the structure must be separately documented (via JSON Schema, API docs, or code comments). Without discipline, JSONB columns become opaque blobs.

4. **GIN index performance characteristics**: GIN indexes on large JSONB columns consume significant storage and slow down writes. For tables with high write throughput (form_submissions, outcome_records), GIN indexes must be sized and monitored carefully.

5. **Partial indexing limitations**: While PostgreSQL supports indexing specific JSONB paths, the optimizer's ability to use these indexes depends on query structure. Subtle query variations may bypass JSONB indexes and fall back to sequential scans.

6. **Reporting query performance**: Complex reports that aggregate across JSONB fields (e.g., disaggregation by demographic dimensions stored in JSONB) are slower than equivalent queries on normalised columns. The extracted `value_numeric` pattern mitigates this for the most common case but does not cover all scenarios.

7. **Data migration complexity**: Migrating data between schema versions when JSONB structures change requires JSON transformation scripts rather than simple ALTER TABLE statements. This is manageable but requires more testing.

---

## Migration and Scaling Considerations

### Initial Deployment

The hybrid model deploys identically to a standard PostgreSQL application. A single managed instance (AWS RDS, Supabase, Neon, or self-hosted) serves both relational and JSONB data. No additional infrastructure is required.

### Monitoring JSONB Performance

Key metrics to track:
- GIN index sizes relative to table sizes (GIN indexes can be 2-5x the data size)
- Query plans for JSONB-involving queries (watch for sequential scans on large tables)
- Write latency on tables with GIN indexes (may need to defer index updates during bulk imports)

### Growth Path (10K-100K Participants)

- **Materialised views** for programme outcome summaries, refreshed on schedule
- **Partial GIN indexes** -- index only the JSONB paths actually queried:
  ```sql
  CREATE INDEX idx_outcomes_direction ON outcome_records ((measurement_data->>'direction'))
      WHERE measurement_data ? 'direction';
  ```
- **Table partitioning** on outcome_records (by year) and audit_log (by month)
- **Read replicas** for reporting workloads

### Large Scale (100K+ Participants)

- **TOAST compression tuning** for large JSONB columns (PostgreSQL compresses JSONB by default but tuning storage parameters can improve read performance)
- **Citus sharding** by organisation_id, which works with JSONB columns
- **Extract-and-materialize pattern**: For the most frequently queried JSONB paths, add materialised columns that are auto-populated by triggers:
  ```sql
  ALTER TABLE outcome_records ADD COLUMN direction VARCHAR(20)
      GENERATED ALWAYS AS (measurement_data->>'direction') STORED;
  ```
- **Archive JSONB data**: Move historical form submissions and their JSONB answers to cold storage, keeping summary outcome records in the primary database

### Migrating from Incumbent Systems

The JSONB approach simplifies data import because source system data does not need to be decomposed into a normalised structure. A Salesforce export with variable custom fields can be imported into JSONB columns with minimal transformation. The import pipeline:

1. Map source system fields to the known relational columns (participant pseudonym, enrolment dates, programme name)
2. Store remaining fields in the appropriate JSONB column (demographics, intake_data, measurement_data)
3. Validate JSONB content against JSON Schema
4. Generate outcome_records with both value_numeric (extracted) and measurement_data (full context)
