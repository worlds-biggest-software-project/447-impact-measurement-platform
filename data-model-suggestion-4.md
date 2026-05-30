# Data Model Suggestion 4: Graph Database + Time-Series Hybrid (Neo4j + TimescaleDB)

## Overview

This model uses a two-engine architecture specifically chosen for the two defining characteristics of the Impact Measurement Platform's domain:

1. **Causal pathway modeling** -- The theory of change, with its directed chains from inputs through activities and outputs to outcomes and impact, is fundamentally a graph. Causal attribution -- determining whether outcome changes are attributable to programme activities versus external factors -- requires traversing and analysing these graph relationships. A graph database (Neo4j) represents this natively.

2. **Longitudinal measurement** -- Outcome measurements, attendance records, and survey responses are time-stamped observations that accumulate over months and years. Distance-travelled calculations, trend analysis, anomaly detection, and cohort comparisons are time-series operations. A purpose-built time-series engine (TimescaleDB, a PostgreSQL extension) handles these far more efficiently than a general-purpose relational database.

The key insight is that no single database engine excels at both graph traversal and time-series aggregation. Rather than forcing one engine to do both poorly, this model uses each engine for what it does best, with a synchronisation layer keeping them consistent.

Neo4j handles: organisations, programmes, theories of change, participant-programme relationships, framework alignment mappings, and causal attribution queries.

TimescaleDB handles: outcome measurements, form submissions, attendance records, anomaly detection, and all time-series aggregation and reporting queries.

A coordination layer (PostgreSQL tables in TimescaleDB) holds the shared reference data (participant identities, indicator definitions, consent records) that both engines need.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Graph Database | Neo4j 5.x Community or Enterprise | Mature graph database with Cypher query language; strong for causal pathway traversal |
| Time-Series Database | TimescaleDB 2.x (PostgreSQL extension) | Time-series on PostgreSQL; hypertables with automatic partitioning; continuous aggregates |
| Coordination Store | PostgreSQL (same instance as TimescaleDB) | Shared reference data; participant identity; consent management |
| Sync Layer | Debezium CDC or application-level dual-write | Keep graph and time-series stores consistent |
| Caching | Redis | Cache dashboard data and frequently accessed graph query results |
| Search | PostgreSQL full-text search + Neo4j full-text indexes | Qualitative data search in both engines |

---

## Architecture Overview

```
                         ┌─────────────────────────────────┐
                         │         API / Application       │
                         │                                 │
                         │  Commands ──►  Command Handlers │
                         │  Queries  ──►  Query Router     │
                         └──────┬──────────────┬───────────┘
                                │              │
                    ┌───────────┴──┐    ┌──────┴───────────┐
                    │              │    │                   │
                    ▼              ▼    ▼                   ▼
          ┌──────────────┐  ┌──────────────────┐  ┌──────────────┐
          │   Neo4j      │  │  PostgreSQL /    │  │   Redis      │
          │   (Graph)    │  │  TimescaleDB     │  │   (Cache)    │
          │              │  │  (Time-Series +  │  │              │
          │  - ToC       │  │   Coordination)  │  │  - Dashboard │
          │  - Causal    │  │                  │  │  - Sessions  │
          │    pathways  │  │  - Outcomes      │  │  - Graph     │
          │  - Framework │  │  - Submissions   │  │    query     │
          │    alignment │  │  - Attendance    │  │    cache     │
          │  - Programme │  │  - Participants  │  │              │
          │    structure │  │  - Indicators    │  │              │
          │              │  │  - Consent       │  │              │
          │              │  │  - Audit log     │  │              │
          └──────────────┘  └──────────────────┘  └──────────────┘
                 │                    │
                 └────────┬───────────┘
                          │
                  ┌───────┴────────┐
                  │  Sync Layer    │
                  │  (Debezium /   │
                  │   dual-write)  │
                  └────────────────┘
```

---

## Neo4j Graph Schema (Cypher)

### Node Types and Relationships

```cypher
// ============================================================
// ORGANISATIONS AND USERS
// ============================================================

// Organisation node
CREATE CONSTRAINT org_id IF NOT EXISTS FOR (o:Organisation) REQUIRE o.id IS UNIQUE;

// Example organisation node:
// (:Organisation {
//     id: "uuid",
//     name: "Youth Employment Alliance",
//     slug: "yea",
//     country_code: "US",
//     timezone: "America/New_York",
//     created_at: datetime()
// })

// User node
CREATE CONSTRAINT user_id IF NOT EXISTS FOR (u:User) REQUIRE u.id IS UNIQUE;

// (:User {
//     id: "uuid",
//     email: "jane@yea.org",
//     first_name: "Jane",
//     last_name: "Smith",
//     role: "programme_manager"
// })

// Relationship: User belongs to Organisation
// (u:User)-[:BELONGS_TO]->(o:Organisation)

// ============================================================
// PROGRAMMES AND COHORTS
// ============================================================

CREATE CONSTRAINT programme_id IF NOT EXISTS FOR (p:Programme) REQUIRE p.id IS UNIQUE;

// (:Programme {
//     id: "uuid",
//     name: "Career Pathways 2026",
//     programme_type: "workforce_development",
//     status: "active",
//     start_date: date("2026-01-15"),
//     end_date: date("2026-12-31"),
//     target_participant_count: 200
// })

// (p:Programme)-[:OPERATED_BY]->(o:Organisation)
// (u:User)-[:MANAGES]->(p:Programme)

CREATE CONSTRAINT cohort_id IF NOT EXISTS FOR (c:Cohort) REQUIRE c.id IS UNIQUE;

// (:Cohort {
//     id: "uuid",
//     name: "Spring 2026 Cohort",
//     start_date: date("2026-03-01"),
//     end_date: date("2026-06-30"),
//     is_control_group: false
// })

// (c:Cohort)-[:PART_OF]->(p:Programme)

// ============================================================
// THEORY OF CHANGE (The core graph model advantage)
// ============================================================

// ToC Container
CREATE CONSTRAINT toc_id IF NOT EXISTS FOR (t:TheoryOfChange) REQUIRE t.id IS UNIQUE;

// (:TheoryOfChange {
//     id: "uuid",
//     version: 1,
//     name: "Career Pathways ToC v1",
//     narrative: "Our theory is that providing...",
//     is_current: true
// })

// (t:TheoryOfChange)-[:MODELS]->(p:Programme)

// ToC Nodes -- the distinct stages in the causal chain
CREATE CONSTRAINT toc_node_id IF NOT EXISTS FOR (n:TocNode) REQUIRE n.id IS UNIQUE;

// Input nodes
// (:TocNode:Input {
//     id: "node-1",
//     title: "Trained career coaches",
//     description: "15 certified coaches with industry experience",
//     position_x: 100, position_y: 200,
//     dosage: null
// })

// Activity nodes
// (:TocNode:Activity {
//     id: "node-2",
//     title: "Weekly career workshops",
//     description: "2-hour sessions covering resume, interview, networking",
//     position_x: 300, position_y: 200,
//     dosage_frequency: "weekly",
//     dosage_hours: 2,
//     dosage_total_sessions: 12
// })

// Output nodes
// (:TocNode:Output {
//     id: "node-3",
//     title: "Workshop completion certificates",
//     description: "Participants completing 8+ sessions receive certification",
//     position_x: 500, position_y: 100
// })

// Outcome nodes
// (:TocNode:Outcome {
//     id: "node-4",
//     title: "Increased employability skills",
//     description: "Measurable improvement in job-readiness scores",
//     position_x: 500, position_y: 300,
//     timeframe: "3_months"
// })

// Impact nodes
// (:TocNode:Impact {
//     id: "node-5",
//     title: "Sustained employment",
//     description: "Participants maintain employment for 6+ months",
//     position_x: 700, position_y: 200,
//     timeframe: "12_months"
// })

// Causal edges connecting ToC nodes
// These edges ARE the theory of change -- the causal hypotheses
// (n1:TocNode)-[:LEADS_TO {
//     label: "Produces",
//     assumption: "Participants attend at least 8 of 12 sessions",
//     evidence_strength: "strong",
//     assumption_status: "supported",
//     evidence_notes: "Average attendance 9.2 sessions (Q3 data)"
// }]->(n2:TocNode)

// Nodes belong to a specific ToC version
// (n:TocNode)-[:IN_TOC]->(t:TheoryOfChange)

// ============================================================
// INDICATORS AND THEIR GRAPH RELATIONSHIPS
// ============================================================

CREATE CONSTRAINT indicator_id IF NOT EXISTS FOR (i:Indicator) REQUIRE i.id IS UNIQUE;

// (:Indicator {
//     id: "uuid",
//     code: "EMP-01",
//     name: "Employment Status",
//     measurement_type: "categorical",
//     direction: "increase",
//     collection_frequency: "quarterly"
// })

// Indicators measure specific ToC nodes (outcomes/outputs)
// (i:Indicator)-[:MEASURES]->(n:TocNode:Outcome)

// Indicators belong to an organisation
// (i:Indicator)-[:DEFINED_BY]->(o:Organisation)

// ============================================================
// FRAMEWORK ALIGNMENT (Natural graph relationships)
// ============================================================

CREATE CONSTRAINT framework_id IF NOT EXISTS FOR (f:Framework) REQUIRE f.id IS UNIQUE;

// (:Framework {
//     id: "uuid",
//     name: "UN Sustainable Development Goals",
//     version: "2030 Agenda",
//     framework_type: "sdg"
// })

// Framework hierarchy as graph nodes
CREATE CONSTRAINT fw_metric_id IF NOT EXISTS FOR (m:FrameworkMetric) REQUIRE m.id IS UNIQUE;

// SDG Goal level
// (:FrameworkMetric:SDGGoal {
//     id: "sdg-1",
//     code: "SDG-1",
//     name: "No Poverty"
// })

// SDG Target level
// (:FrameworkMetric:SDGTarget {
//     id: "sdg-1-1",
//     code: "1.1",
//     name: "Eradicate extreme poverty"
// })

// SDG Indicator level
// (:FrameworkMetric:SDGIndicator {
//     id: "sdg-1-1-1",
//     code: "1.1.1",
//     name: "Population below international poverty line"
// })

// Framework hierarchy
// (m:FrameworkMetric)-[:PART_OF]->(f:Framework)
// (t:SDGTarget)-[:UNDER]->(g:SDGGoal)
// (i:SDGIndicator)-[:UNDER]->(t:SDGTarget)

// IRIS+ hierarchy
// (:FrameworkMetric:IRISCategory { code: "PI", name: "Product Impact" })
// (:FrameworkMetric:IRISTheme { code: "PI-EMPLOYMENT", name: "Employment Generation" })
// (:FrameworkMetric:IRISMetric { code: "PI1234", name: "Full-time Employees" })

// (theme)-[:UNDER]->(category)
// (metric)-[:UNDER]->(theme)

// Alignment mappings between internal indicators and framework metrics
// (i:Indicator)-[:ALIGNS_WITH {
//     mapping_confidence: "exact",
//     mapping_notes: "Direct measure of employment status",
//     mapped_by: "user-uuid",
//     mapped_at: datetime()
// }]->(m:FrameworkMetric)

// ============================================================
// PARTICIPANTS AND ENROLMENTS (Graph relationships)
// ============================================================

CREATE CONSTRAINT participant_id IF NOT EXISTS FOR (p:Participant) REQUIRE p.id IS UNIQUE;

// (:Participant {
//     id: "uuid",
//     pseudonym_id: "P-2026-00042",
//     status: "active",
//     created_at: datetime()
// })
// Note: PII and demographics stored in PostgreSQL only (not in graph)

// (p:Participant)-[:REGISTERED_WITH]->(o:Organisation)

// Enrolment as a relationship with properties
// (p:Participant)-[:ENROLLED_IN {
//     enrolment_id: "uuid",
//     enrolment_date: date("2026-03-01"),
//     status: "active",
//     exit_date: null,
//     exit_reason: null
// }]->(prog:Programme)

// Cohort membership
// (p:Participant)-[:MEMBER_OF]->(c:Cohort)

// ============================================================
// FORM DEFINITIONS (Graph node for relationships, detail in PG)
// ============================================================

CREATE CONSTRAINT form_id IF NOT EXISTS FOR (f:FormDefinition) REQUIRE f.id IS UNIQUE;

// (:FormDefinition {
//     id: "uuid",
//     name: "Baseline Employability Assessment",
//     form_type: "baseline_assessment",
//     version: 1,
//     status: "published"
// })
// Note: Full form structure (sections, questions, choices) stored in PostgreSQL

// (f:FormDefinition)-[:USED_BY]->(p:Programme)
// (f:FormDefinition)-[:COLLECTS_DATA_FOR]->(i:Indicator)
// (f:FormDefinition)-[:CREATED_BY]->(u:User)
```

### Power of Graph Queries for Impact Measurement

```cypher
// ============================================================
// QUERY 1: Full causal chain from input to impact
// "Show me the complete theory of change pathway"
// ============================================================

MATCH path = (input:TocNode:Input)-[:LEADS_TO*]->(impact:TocNode:Impact)
WHERE input.id IN [nodeId]
RETURN path, 
       [n IN nodes(path) | n.title] AS pathway_titles,
       [r IN relationships(path) | r.assumption] AS assumptions,
       [r IN relationships(path) | r.evidence_strength] AS evidence_strengths;

// ============================================================
// QUERY 2: Which indicators measure a specific outcome in the ToC?
// ============================================================

MATCH (impact:TocNode:Impact {title: "Sustained employment"})
      <-[:LEADS_TO*]-(outcome:TocNode:Outcome)
      <-[:MEASURES]-(indicator:Indicator)
RETURN outcome.title AS outcome,
       indicator.code AS indicator_code,
       indicator.name AS indicator_name;

// ============================================================
// QUERY 3: Framework alignment -- how does this programme map to SDGs?
// ============================================================

MATCH (prog:Programme {id: $programme_id})
      <-[:MODELS]-(toc:TheoryOfChange {is_current: true})
      <-[:IN_TOC]-(node:TocNode)
      <-[:MEASURES]-(indicator:Indicator)
      -[:ALIGNS_WITH]->(metric:FrameworkMetric)
      -[:UNDER*0..3]->(goal:FrameworkMetric:SDGGoal)
RETURN goal.code AS sdg_goal,
       goal.name AS sdg_name,
       COLLECT(DISTINCT indicator.name) AS contributing_indicators,
       COLLECT(DISTINCT node.title) AS related_outcomes;

// ============================================================
// QUERY 4: Cross-programme indicator reuse
// "Which indicators are used across multiple programmes?"
// ============================================================

MATCH (i:Indicator)-[:MEASURES]->(n:TocNode)-[:IN_TOC]->(toc:TheoryOfChange)-[:MODELS]->(p:Programme)
WITH i, COLLECT(DISTINCT p.name) AS programmes, COUNT(DISTINCT p) AS programme_count
WHERE programme_count > 1
RETURN i.code, i.name, programme_count, programmes
ORDER BY programme_count DESC;

// ============================================================
// QUERY 5: Causal attribution support
// "What programme activities could have influenced this outcome?"
// ============================================================

MATCH (outcome:TocNode:Outcome {id: $outcome_node_id})
      <-[:LEADS_TO*]-(activity:TocNode:Activity)
MATCH path = shortestPath((activity)-[:LEADS_TO*]->(outcome))
RETURN activity.title AS activity,
       length(path) AS causal_distance,
       [r IN relationships(path) | r.assumption] AS causal_assumptions,
       [r IN relationships(path) | r.evidence_strength] AS evidence_strengths
ORDER BY causal_distance;

// ============================================================
// QUERY 6: Identify weak links in the theory of change
// "Which causal assumptions are untested or weakly supported?"
// ============================================================

MATCH (n1:TocNode)-[r:LEADS_TO]->(n2:TocNode)
WHERE r.evidence_strength IN ['weak', 'untested']
MATCH (n1)-[:IN_TOC]->(toc:TheoryOfChange)-[:MODELS]->(prog:Programme)
RETURN prog.name AS programme,
       n1.title AS from_stage,
       n2.title AS to_stage,
       r.assumption AS assumption,
       r.evidence_strength AS strength,
       r.assumption_status AS status;

// ============================================================
// QUERY 7: Participant journey across programmes
// "Show all programmes and cohorts a participant has been part of"
// ============================================================

MATCH (p:Participant {pseudonym_id: $pseudonym})
      -[e:ENROLLED_IN]->(prog:Programme)
      -[:OPERATED_BY]->(org:Organisation)
OPTIONAL MATCH (p)-[:MEMBER_OF]->(c:Cohort)-[:PART_OF]->(prog)
RETURN prog.name AS programme,
       e.enrolment_date AS enrolled,
       e.exit_date AS exited,
       e.status AS status,
       c.name AS cohort,
       org.name AS organisation
ORDER BY e.enrolment_date;
```

---

## TimescaleDB Schema (Time-Series + Coordination)

### Coordination Tables (Standard PostgreSQL)

```sql
-- ============================================================
-- SHARED REFERENCE DATA
-- These tables are the "source of truth" for entities that both
-- Neo4j and TimescaleDB need. Changes are synced to Neo4j.
-- ============================================================

-- Participant identity (PII and demographics live here, NOT in Neo4j)
CREATE TABLE participants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    pseudonym_id VARCHAR(100) NOT NULL,
    pii_encrypted BYTEA,
    demographics JSONB NOT NULL DEFAULT '{}'::JSONB,
    dedup_hashes JSONB,
    dedup_cluster_id UUID,
    status VARCHAR(50) DEFAULT 'active',
    merged_into_id UUID REFERENCES participants(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(organisation_id, pseudonym_id)
);

CREATE INDEX idx_participants_org ON participants(organisation_id);
CREATE INDEX idx_participants_demographics ON participants USING GIN (demographics);

-- Consent management (GDPR compliance stays in PostgreSQL)
CREATE TABLE participant_consents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    participant_id UUID NOT NULL REFERENCES participants(id) ON DELETE CASCADE,
    consent_type VARCHAR(100) NOT NULL,
    granted BOOLEAN NOT NULL,
    details JSONB NOT NULL DEFAULT '{}'::JSONB,
    granted_at TIMESTAMPTZ,
    withdrawn_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    recorded_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_consents_participant ON participant_consents(participant_id);

-- Indicator definitions (shared reference for both engines)
CREATE TABLE indicators (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    code VARCHAR(50),
    name VARCHAR(255) NOT NULL,
    measurement_type VARCHAR(50) NOT NULL,
    direction VARCHAR(20) DEFAULT 'increase',
    config JSONB NOT NULL DEFAULT '{}'::JSONB,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_indicators_org ON indicators(organisation_id);

-- Programme enrolments (relational data for time-series queries)
CREATE TABLE programme_enrolments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    participant_id UUID NOT NULL REFERENCES participants(id),
    programme_id UUID NOT NULL,  -- FK enforced at app level (programme lives in Neo4j)
    cohort_id UUID,
    enrolment_date DATE NOT NULL,
    exit_date DATE,
    exit_reason VARCHAR(100),
    status VARCHAR(50) DEFAULT 'enrolled',
    intake_data JSONB NOT NULL DEFAULT '{}'::JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_enrolments_participant ON programme_enrolments(participant_id);
CREATE INDEX idx_enrolments_programme ON programme_enrolments(programme_id);
CREATE INDEX idx_enrolments_status ON programme_enrolments(programme_id, status);

-- Form definitions (structure stored here, graph relationships in Neo4j)
CREATE TABLE form_definitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    programme_id UUID,
    name VARCHAR(255) NOT NULL,
    form_type VARCHAR(50) NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    status VARCHAR(50) DEFAULT 'draft',
    definition JSONB NOT NULL,
    created_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_forms_org ON form_definitions(organisation_id);
CREATE INDEX idx_forms_programme ON form_definitions(programme_id);
```

### Time-Series Hypertables

```sql
-- ============================================================
-- OUTCOME MEASUREMENTS (Primary time-series data)
-- This is the core hypertable: every outcome measurement is a
-- time-stamped observation that TimescaleDB handles natively.
-- ============================================================

CREATE TABLE outcome_measurements (
    time TIMESTAMPTZ NOT NULL,  -- TimescaleDB requires a time column
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    programme_id UUID NOT NULL,
    enrolment_id UUID NOT NULL,
    participant_id UUID NOT NULL,
    indicator_id UUID NOT NULL,
    measurement_period VARCHAR(50) NOT NULL,
    -- Measurement values
    value_numeric NUMERIC,
    value_text TEXT,
    value_categorical VARCHAR(255),
    -- Rich measurement context
    measurement_data JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Source tracing
    form_submission_id UUID,
    data_source VARCHAR(100),
    -- Validation
    is_validated BOOLEAN DEFAULT FALSE,
    -- Unique constraint per measurement point
    UNIQUE(enrolment_id, indicator_id, measurement_period, time)
);

-- Convert to hypertable with monthly chunks
SELECT create_hypertable('outcome_measurements', 'time',
    chunk_time_interval => INTERVAL '1 month');

-- Indexes optimised for time-series access patterns
CREATE INDEX idx_outcomes_enrolment_time ON outcome_measurements(enrolment_id, time DESC);
CREATE INDEX idx_outcomes_indicator_time ON outcome_measurements(indicator_id, time DESC);
CREATE INDEX idx_outcomes_programme_time ON outcome_measurements(programme_id, time DESC);
CREATE INDEX idx_outcomes_org_time ON outcome_measurements(organisation_id, time DESC);
-- Composite for distance-travelled queries
CREATE INDEX idx_outcomes_distance ON outcome_measurements(
    enrolment_id, indicator_id, measurement_period
);

-- ============================================================
-- FORM SUBMISSIONS (Time-series of data collection events)
-- ============================================================

CREATE TABLE form_submissions (
    time TIMESTAMPTZ NOT NULL,  -- submission timestamp
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    form_id UUID NOT NULL,
    enrolment_id UUID NOT NULL,
    organisation_id UUID NOT NULL,
    programme_id UUID NOT NULL,
    submitted_by UUID,
    submission_type VARCHAR(50),
    status VARCHAR(50) DEFAULT 'draft',
    collection_method VARCHAR(50),
    answers JSONB NOT NULL DEFAULT '{}'::JSONB,
    sync_metadata JSONB,
    device_id VARCHAR(100),
    gps_latitude DECIMAL(10,7),
    gps_longitude DECIMAL(10,7)
);

SELECT create_hypertable('form_submissions', 'time',
    chunk_time_interval => INTERVAL '1 month');

CREATE INDEX idx_submissions_form_time ON form_submissions(form_id, time DESC);
CREATE INDEX idx_submissions_enrolment_time ON form_submissions(enrolment_id, time DESC);
CREATE INDEX idx_submissions_org_time ON form_submissions(organisation_id, time DESC);

-- ============================================================
-- ACTIVITY ATTENDANCE (Time-series of participation events)
-- ============================================================

CREATE TABLE activity_attendance (
    time TIMESTAMPTZ NOT NULL,  -- activity date/time
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    activity_id UUID NOT NULL,
    enrolment_id UUID NOT NULL,
    programme_id UUID NOT NULL,
    organisation_id UUID NOT NULL,
    attended BOOLEAN NOT NULL DEFAULT TRUE,
    attendance_type VARCHAR(50),
    activity_name VARCHAR(255),
    activity_type VARCHAR(100),
    notes TEXT,
    recorded_by UUID
);

SELECT create_hypertable('activity_attendance', 'time',
    chunk_time_interval => INTERVAL '1 month');

CREATE INDEX idx_attendance_enrolment_time ON activity_attendance(enrolment_id, time DESC);
CREATE INDEX idx_attendance_programme_time ON activity_attendance(programme_id, time DESC);

-- ============================================================
-- QUALITATIVE EVIDENCE (Time-stamped text data)
-- ============================================================

CREATE TABLE qualitative_evidence (
    time TIMESTAMPTZ NOT NULL,  -- recording date
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    programme_id UUID,
    enrolment_id UUID,
    evidence_type VARCHAR(50) NOT NULL,
    content TEXT NOT NULL,
    word_count INTEGER,
    language VARCHAR(10) DEFAULT 'en',
    source JSONB,
    analysis JSONB NOT NULL DEFAULT '{}'::JSONB,
    content_tsv TSVECTOR GENERATED ALWAYS AS (to_tsvector('english', content)) STORED
);

SELECT create_hypertable('qualitative_evidence', 'time',
    chunk_time_interval => INTERVAL '3 months');

CREATE INDEX idx_qual_programme_time ON qualitative_evidence(programme_id, time DESC);
CREATE INDEX idx_qual_tsv ON qualitative_evidence USING GIN (content_tsv);
CREATE INDEX idx_qual_analysis ON qualitative_evidence USING GIN (analysis);

-- ============================================================
-- OUTCOME ANOMALIES (Time-series of detected anomalies)
-- ============================================================

CREATE TABLE outcome_anomalies (
    time TIMESTAMPTZ NOT NULL,
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    programme_id UUID NOT NULL,
    indicator_id UUID NOT NULL,
    organisation_id UUID NOT NULL,
    cohort_id UUID,
    anomaly JSONB NOT NULL,
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID,
    resolution_notes TEXT
);

SELECT create_hypertable('outcome_anomalies', 'time',
    chunk_time_interval => INTERVAL '3 months');

CREATE INDEX idx_anomalies_programme_time ON outcome_anomalies(programme_id, time DESC);

-- ============================================================
-- AUDIT LOG (Time-series)
-- ============================================================

CREATE TABLE audit_log (
    time TIMESTAMPTZ NOT NULL,
    id BIGSERIAL,
    organisation_id UUID NOT NULL,
    user_id UUID,
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(100) NOT NULL,
    entity_id UUID,
    changes JSONB,
    request_metadata JSONB
);

SELECT create_hypertable('audit_log', 'time',
    chunk_time_interval => INTERVAL '1 month');

CREATE INDEX idx_audit_org_time ON audit_log(organisation_id, time DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id, time DESC);
```

### TimescaleDB Continuous Aggregates

```sql
-- ============================================================
-- CONTINUOUS AGGREGATES
-- These materialised views update automatically as new data
-- arrives, providing real-time dashboard data without manual
-- refresh schedules.
-- ============================================================

-- Programme-level outcome summary (updated continuously)
CREATE MATERIALIZED VIEW programme_outcome_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    programme_id,
    indicator_id,
    organisation_id,
    COUNT(*) AS measurement_count,
    AVG(value_numeric) AS mean_value,
    MIN(value_numeric) AS min_value,
    MAX(value_numeric) AS max_value,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY value_numeric) AS median_value,
    STDDEV(value_numeric) AS std_dev
FROM outcome_measurements
WHERE value_numeric IS NOT NULL
GROUP BY bucket, programme_id, indicator_id, organisation_id;

-- Refresh policy: update every hour, covering the last 2 hours
SELECT add_continuous_aggregate_policy('programme_outcome_daily',
    start_offset => INTERVAL '2 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- Monthly programme summary for reporting
CREATE MATERIALIZED VIEW programme_outcome_monthly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', time) AS bucket,
    programme_id,
    indicator_id,
    organisation_id,
    measurement_period,
    COUNT(*) AS measurement_count,
    AVG(value_numeric) AS mean_value,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY value_numeric) AS median_value,
    STDDEV(value_numeric) AS std_dev,
    COUNT(CASE WHEN is_validated THEN 1 END) AS validated_count
FROM outcome_measurements
WHERE value_numeric IS NOT NULL
GROUP BY bucket, programme_id, indicator_id, organisation_id, measurement_period;

SELECT add_continuous_aggregate_policy('programme_outcome_monthly',
    start_offset => INTERVAL '2 months',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');

-- Attendance trends
CREATE MATERIALIZED VIEW attendance_weekly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 week', time) AS bucket,
    programme_id,
    organisation_id,
    COUNT(*) AS total_records,
    COUNT(CASE WHEN attended THEN 1 END) AS attended_count,
    COUNT(CASE WHEN NOT attended THEN 1 END) AS absent_count,
    ROUND(
        COUNT(CASE WHEN attended THEN 1 END)::NUMERIC /
        NULLIF(COUNT(*), 0) * 100, 1
    ) AS attendance_rate
FROM activity_attendance
GROUP BY bucket, programme_id, organisation_id;

SELECT add_continuous_aggregate_policy('attendance_weekly',
    start_offset => INTERVAL '2 weeks',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');

-- Data collection activity tracking
CREATE MATERIALIZED VIEW submissions_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    programme_id,
    organisation_id,
    form_id,
    status,
    collection_method,
    COUNT(*) AS submission_count
FROM form_submissions
GROUP BY bucket, programme_id, organisation_id, form_id, status, collection_method;

SELECT add_continuous_aggregate_policy('submissions_daily',
    start_offset => INTERVAL '2 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

### Time-Series Analytical Queries

```sql
-- ============================================================
-- DISTANCE TRAVELLED (Baseline vs Endline comparison)
-- ============================================================

WITH baseline AS (
    SELECT enrolment_id, indicator_id, value_numeric,
           time AS baseline_time
    FROM outcome_measurements
    WHERE measurement_period = 'baseline'
        AND programme_id = $1
),
endline AS (
    SELECT enrolment_id, indicator_id, value_numeric,
           time AS endline_time
    FROM outcome_measurements
    WHERE measurement_period = 'endline'
        AND programme_id = $1
)
SELECT
    b.enrolment_id,
    i.name AS indicator_name,
    b.value_numeric AS baseline_value,
    e.value_numeric AS endline_value,
    e.value_numeric - b.value_numeric AS absolute_change,
    CASE WHEN b.value_numeric > 0
         THEN ROUND(((e.value_numeric - b.value_numeric) / b.value_numeric * 100), 1)
         ELSE NULL END AS pct_change,
    e.endline_time - b.baseline_time AS time_in_programme
FROM baseline b
JOIN endline e ON b.enrolment_id = e.enrolment_id
    AND b.indicator_id = e.indicator_id
JOIN indicators i ON i.id = b.indicator_id;

-- ============================================================
-- OUTCOME TREND ANALYSIS (Time-series specific)
-- Uses TimescaleDB time_bucket for efficient time-range grouping
-- ============================================================

SELECT
    time_bucket('1 month', time) AS month,
    indicator_id,
    COUNT(*) AS measurements,
    AVG(value_numeric) AS mean_value,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY value_numeric) AS median,
    STDDEV(value_numeric) AS std_dev
FROM outcome_measurements
WHERE programme_id = $1
    AND indicator_id = $2
    AND time >= NOW() - INTERVAL '24 months'
GROUP BY month, indicator_id
ORDER BY month;

-- ============================================================
-- ANOMALY DETECTION (Statistical deviation from trend)
-- ============================================================

WITH recent AS (
    SELECT
        time_bucket('1 week', time) AS week,
        programme_id,
        indicator_id,
        AVG(value_numeric) AS weekly_mean
    FROM outcome_measurements
    WHERE programme_id = $1
        AND time >= NOW() - INTERVAL '12 months'
    GROUP BY week, programme_id, indicator_id
),
stats AS (
    SELECT
        programme_id,
        indicator_id,
        AVG(weekly_mean) AS overall_mean,
        STDDEV(weekly_mean) AS overall_std
    FROM recent
    GROUP BY programme_id, indicator_id
)
SELECT
    r.week,
    r.indicator_id,
    r.weekly_mean,
    s.overall_mean,
    (r.weekly_mean - s.overall_mean) / NULLIF(s.overall_std, 0) AS z_score
FROM recent r
JOIN stats s ON r.programme_id = s.programme_id
    AND r.indicator_id = s.indicator_id
WHERE ABS((r.weekly_mean - s.overall_mean) / NULLIF(s.overall_std, 0)) > 2
ORDER BY r.week DESC;

-- ============================================================
-- PARTICIPANT JOURNEY TIMELINE (Time-ordered events)
-- ============================================================

WITH events AS (
    -- Attendance events
    SELECT time, 'attendance' AS event_type,
           activity_name AS description,
           CASE WHEN attended THEN 'attended' ELSE 'absent' END AS status,
           NULL::NUMERIC AS value
    FROM activity_attendance
    WHERE enrolment_id = $1

    UNION ALL

    -- Form submissions
    SELECT time, 'form_submission' AS event_type,
           (SELECT name FROM form_definitions WHERE id = fs.form_id) AS description,
           status,
           NULL::NUMERIC AS value
    FROM form_submissions fs
    WHERE enrolment_id = $1

    UNION ALL

    -- Outcome measurements
    SELECT time, 'outcome_measurement' AS event_type,
           (SELECT name FROM indicators WHERE id = om.indicator_id) AS description,
           measurement_period AS status,
           value_numeric AS value
    FROM outcome_measurements om
    WHERE enrolment_id = $1
)
SELECT * FROM events ORDER BY time;

-- ============================================================
-- DATA RETENTION AND COMPRESSION
-- TimescaleDB native compression for historical data
-- ============================================================

-- Enable compression on outcome measurements (compress chunks older than 6 months)
ALTER TABLE outcome_measurements SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'programme_id, indicator_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('outcome_measurements', INTERVAL '6 months');

-- Compression on form submissions
ALTER TABLE form_submissions SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'programme_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('form_submissions', INTERVAL '6 months');

-- Compression on attendance
ALTER TABLE activity_attendance SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'programme_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('activity_attendance', INTERVAL '6 months');

-- Data retention policy for audit logs (keep 7 years per GDPR)
SELECT add_retention_policy('audit_log', INTERVAL '7 years');
```

---

## Synchronisation Strategy

### Dual-Write Coordination

```python
# Application-level dual-write with transactional outbox pattern

class ImpactMeasurementService:
    def __init__(self, neo4j_driver, pg_connection):
        self.neo4j = neo4j_driver
        self.pg = pg_connection

    async def create_programme(self, programme_data):
        programme_id = uuid4()
        
        # 1. Write to PostgreSQL (coordination record)
        # This is inside a transaction with the outbox
        async with self.pg.transaction():
            # Write any PG-side data
            await self.pg.execute("""
                INSERT INTO sync_outbox (entity_type, entity_id, action, data)
                VALUES ('Programme', $1, 'create', $2)
            """, programme_id, json.dumps(programme_data))
        
        # 2. Background worker reads outbox and writes to Neo4j
        # This ensures eventual consistency without distributed transactions
    
    async def record_outcome(self, measurement_data):
        # Outcomes go directly to TimescaleDB (no Neo4j involvement)
        await self.pg.execute("""
            INSERT INTO outcome_measurements (time, id, organisation_id, ...)
            VALUES ($1, $2, $3, ...)
        """, measurement_data.values())
    
    async def update_toc(self, toc_data):
        # Theory of Change goes to Neo4j (graph data)
        async with self.neo4j.session() as session:
            await session.run("""
                MATCH (toc:TheoryOfChange {id: $toc_id})
                SET toc.updated_at = datetime()
                // ... update nodes and edges
            """, toc_id=toc_data.id)

# Outbox processor (runs as background worker)
class OutboxProcessor:
    async def process(self):
        pending = await self.pg.fetch("""
            SELECT * FROM sync_outbox
            WHERE processed_at IS NULL
            ORDER BY created_at
            LIMIT 100
        """)
        
        for record in pending:
            try:
                await self.sync_to_neo4j(record)
                await self.pg.execute("""
                    UPDATE sync_outbox SET processed_at = NOW()
                    WHERE id = $1
                """, record.id)
            except Exception as e:
                await self.pg.execute("""
                    UPDATE sync_outbox
                    SET retry_count = retry_count + 1,
                        last_error = $2
                    WHERE id = $1
                """, record.id, str(e))
```

### Sync Outbox Table

```sql
CREATE TABLE sync_outbox (
    id BIGSERIAL PRIMARY KEY,
    entity_type VARCHAR(100) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,  -- create, update, delete
    data JSONB NOT NULL,
    target_store VARCHAR(50) NOT NULL DEFAULT 'neo4j',
    processed_at TIMESTAMPTZ,
    retry_count INTEGER DEFAULT 0,
    last_error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outbox_pending ON sync_outbox(created_at)
    WHERE processed_at IS NULL;
```

---

## Pros and Cons

### Pros

1. **Theory of Change is a natural graph**: The ToC -- with its nodes (inputs, activities, outputs, outcomes, impact), edges (causal pathways), and properties (assumptions, evidence strength) -- maps perfectly to a graph database. Querying "what activities lead to this outcome?" or "trace the full causal chain" is a simple graph traversal, not a multi-table JOIN with recursive CTEs.

2. **Framework alignment is graph-native**: The hierarchical taxonomies of IRIS+ and SDGs (goal -> target -> indicator) and their cross-references to internal indicators form a natural graph. Finding all SDG goals a programme contributes to is a single Cypher query, not a series of self-joins on a framework_metrics table.

3. **Time-series performance**: TimescaleDB's hypertables with automatic chunking, continuous aggregates, and native compression provide orders-of-magnitude better performance for the longitudinal outcome queries that are the platform's core value proposition. Time-bucketed trend analysis, anomaly detection, and cohort comparison are first-class operations.

4. **Automatic data lifecycle**: TimescaleDB's compression policies automatically compress historical chunks (90%+ compression ratios typical), and retention policies handle GDPR-compliant data deletion by time period. No manual partition management needed.

5. **Continuous aggregates eliminate batch computation**: Programme outcome summaries, attendance rates, and submission counts update automatically as data arrives. No need for scheduled batch jobs to refresh materialised views.

6. **Causal attribution queries**: The graph model enables sophisticated causal pathway analysis that would be extremely complex in a relational model. "What is the shortest causal path from this activity to this outcome?" and "Which assumptions along this path are weakest?" are native graph operations.

7. **Cross-programme analysis**: The graph naturally represents relationships between programmes, shared indicators, common framework alignments, and participant journeys across programmes. These cross-cutting queries are Neo4j's strength.

### Cons

1. **Operational complexity**: Running two database engines (Neo4j + PostgreSQL/TimescaleDB) doubles the infrastructure burden. Backups, monitoring, upgrades, and disaster recovery must be coordinated across both systems. This is significant for resource-constrained nonprofits.

2. **Synchronisation challenges**: Keeping the graph and time-series stores consistent requires a sync layer (outbox pattern or CDC). This introduces eventual consistency and the risk of sync failures. Debugging data inconsistencies across two stores is harder than debugging a single database.

3. **Skill requirements**: Developers need proficiency in both Cypher (Neo4j's query language) and SQL/TimescaleDB. Finding contributors with both skills narrows the talent pool, especially in the nonprofit technology space.

4. **Cost**: Neo4j Enterprise has significant licensing costs. Neo4j Community Edition is free but lacks features like clustering, role-based access control, and advanced monitoring. For a platform targeting nonprofits, cost is a critical factor.

5. **Transaction boundaries**: There is no distributed transaction across Neo4j and PostgreSQL. Operations that must update both stores (e.g., creating a programme with its ToC and initial indicators) cannot be atomic. The outbox pattern provides eventual consistency but not strict consistency.

6. **Participant PII split**: Participant identity data must live in PostgreSQL (for GDPR compliance features, encryption, consent management) while participant relationship data lives in Neo4j. This split means some queries require hitting both stores and joining at the application layer.

7. **Self-hosted deployment complexity**: The platform targets self-hosted deployment for data sovereignty. Requiring organisations to operate both Neo4j and PostgreSQL/TimescaleDB raises the deployment bar significantly compared to a single-database architecture.

8. **Testing complexity**: Integration tests must verify behaviour across both stores, including sync scenarios, partial failures, and recovery. This increases the testing surface area substantially.

---

## Migration and Scaling Considerations

### Initial Deployment (Simplified)

For initial deployments, a simplified architecture is possible:

- **TimescaleDB only** for the first release, with the graph data stored as JSONB in PostgreSQL (similar to Suggestion 3)
- **Neo4j added later** when ToC complexity or cross-programme analysis demands justify the additional infrastructure
- The coordination tables are designed to support this phased approach -- they work standalone and gain additional graph capabilities when Neo4j is introduced

### Growth Path

**0-10K participants:**
- Single PostgreSQL/TimescaleDB instance with Neo4j Community Edition
- All data on a single server (4-core, 16GB RAM is sufficient)
- Outbox processor runs in the application process

**10K-50K participants:**
- Dedicated Neo4j instance (separate server, 8GB heap)
- TimescaleDB with compression enabled for data older than 6 months
- Background outbox processor as a separate service
- Redis caching for dashboard queries

**50K-200K participants:**
- Neo4j clustered deployment (read replicas for graph queries)
- TimescaleDB multi-node for distributed hypertables
- Dedicated continuous aggregate refresh policies tuned for query patterns
- CDC (Debezium) replaces outbox for more reliable sync

**200K+ participants:**
- Neo4j Enterprise with causal clustering
- TimescaleDB with dedicated read replicas for reporting
- Graph query result caching with intelligent invalidation
- Consider Apache AGE (graph on PostgreSQL) as a Neo4j alternative to reduce infrastructure

### Migration from Incumbent Systems

1. **Phase 1**: Import participant and enrolment data into PostgreSQL coordination tables
2. **Phase 2**: Import historical outcome measurements into TimescaleDB hypertables
3. **Phase 3**: Build programme structures and theories of change in Neo4j
4. **Phase 4**: Establish framework alignments in the graph
5. **Phase 5**: Run continuous aggregates against imported historical data to pre-populate dashboards

### GDPR Considerations

- All PII stays in PostgreSQL with encryption -- never stored in Neo4j
- Neo4j stores only pseudonymised identifiers and relationship data
- TimescaleDB retention policies handle time-based data deletion
- Erasure requests processed in PostgreSQL; Neo4j nodes anonymised via sync layer
- Compressed TimescaleDB chunks with expired PII can be dropped entirely

### Alternative: Apache AGE (Graph on PostgreSQL)

If the operational complexity of Neo4j is prohibitive, Apache AGE provides graph query capabilities as a PostgreSQL extension. This would allow a single-database deployment while retaining graph query semantics for the theory of change:

```sql
-- Apache AGE example (graph queries in PostgreSQL)
SELECT * FROM cypher('impact_graph', $$
    MATCH (a:Activity)-[:LEADS_TO*]->(o:Outcome)
    WHERE a.programme_id = 'uuid-here'
    RETURN a.title, o.title
$$) AS (activity agtype, outcome agtype);
```

This approach sacrifices some of Neo4j's performance and ecosystem advantages but eliminates the dual-database operational burden. It could serve as a stepping stone, with migration to Neo4j if graph query complexity grows.
