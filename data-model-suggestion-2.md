# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Overview

This model applies Event Sourcing and Command Query Responsibility Segregation (CQRS) to the Impact Measurement Platform. Instead of storing the current state of participants, outcomes, and programme data in mutable rows, every state change is captured as an immutable event in an append-only event store. Read models (projections) are materialised from these events to serve queries efficiently.

This architecture is a natural fit for impact measurement because the domain fundamentally cares about change over time. The entire purpose of the platform is to answer "what changed for this participant between baseline and endline?" -- which is precisely what an event log captures natively. Every assessment, every attendance record, every outcome measurement is already a discrete event in the real world. Event sourcing stores them as such, preserving the full history without the lossy updates of a traditional CRUD system.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Event Store | EventStoreDB or PostgreSQL with append-only tables | EventStoreDB is purpose-built; PostgreSQL option reduces infrastructure for smaller deployments |
| Message Broker | Apache Kafka or NATS JetStream | Durable event streaming for projections and integrations |
| Read Models | PostgreSQL + Redis | PostgreSQL for complex query projections; Redis for dashboard caching |
| API Layer | Node.js/TypeScript or Python FastAPI | Command handlers and query endpoints |
| Projection Engine | Custom workers or Marten (if .NET) | Event subscription and read model updates |
| Offline Sync | CRDTs + Event merge | Conflict-free offline event collection on mobile |

---

## Event Store Schema

### Core Event Store Tables (PostgreSQL Implementation)

```sql
-- The central event store: every state change in the system is recorded here.
-- This table is APPEND-ONLY. Events are never updated or deleted.
CREATE TABLE event_store (
    -- Global sequential position for ordering
    global_position BIGSERIAL NOT NULL,
    -- Stream identification
    stream_id VARCHAR(255) NOT NULL,       -- e.g. "participant-a1b2c3d4"
    stream_type VARCHAR(100) NOT NULL,     -- e.g. "Participant", "ProgrammeEnrolment"
    -- Event metadata
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type VARCHAR(200) NOT NULL,      -- e.g. "ParticipantEnrolled"
    event_version INTEGER NOT NULL,        -- schema version of this event type
    -- Event data
    data JSONB NOT NULL,                   -- the event payload
    metadata JSONB NOT NULL DEFAULT '{}',  -- correlation IDs, causation, user info
    -- Optimistic concurrency
    stream_position INTEGER NOT NULL,      -- position within the stream
    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Organisation scoping for multi-tenancy
    organisation_id UUID NOT NULL,
    -- Ensure ordering within a stream
    UNIQUE(stream_id, stream_position)
);

-- Primary access pattern: read all events for a stream in order
CREATE INDEX idx_events_stream ON event_store(stream_id, stream_position);
-- Access pattern: read all events of a type (for projections)
CREATE INDEX idx_events_type ON event_store(event_type, global_position);
-- Access pattern: read all events after a position (catch-up subscriptions)
CREATE INDEX idx_events_position ON event_store(global_position);
-- Access pattern: tenant-scoped queries
CREATE INDEX idx_events_org ON event_store(organisation_id, global_position);
-- Access pattern: time-based queries
CREATE INDEX idx_events_created ON event_store(created_at);

-- Snapshot store for performance: periodically save aggregate state
-- to avoid replaying entire event history on every load.
CREATE TABLE snapshots (
    stream_id VARCHAR(255) NOT NULL,
    stream_type VARCHAR(100) NOT NULL,
    stream_position INTEGER NOT NULL,  -- position at which snapshot was taken
    data JSONB NOT NULL,               -- serialised aggregate state
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (stream_id)
);

-- Subscription checkpoints: track where each projection has read to.
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(200) PRIMARY KEY,
    last_global_position BIGINT NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Dead letter queue for events that failed projection processing.
CREATE TABLE dead_letter_events (
    id BIGSERIAL PRIMARY KEY,
    event_id UUID NOT NULL,
    projection_name VARCHAR(200) NOT NULL,
    error_message TEXT,
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 5,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_retry_at TIMESTAMPTZ
);
```

---

## Event Type Definitions

### Organisation and User Events

```
OrganisationCreated
  { organisation_id, name, slug, country_code, timezone }

OrganisationUpdated
  { organisation_id, changes: { field: { old, new } } }

UserRegistered
  { user_id, organisation_id, email, role, first_name, last_name }

UserRoleChanged
  { user_id, old_role, new_role, changed_by }

UserDeactivated
  { user_id, reason, deactivated_by }
```

### Programme Lifecycle Events

```
ProgrammeCreated
  { programme_id, organisation_id, name, programme_type, created_by }

ProgrammeActivated
  { programme_id, start_date, target_participant_count }

ProgrammePaused
  { programme_id, reason, paused_by }

ProgrammeCompleted
  { programme_id, end_date, final_participant_count }

ProgrammeArchived
  { programme_id, archived_by }

CohortCreated
  { cohort_id, programme_id, name, start_date, end_date, is_control_group }
```

### Theory of Change Events

```
TheoryOfChangeCreated
  { toc_id, programme_id, version, name, narrative, created_by }

TocNodeAdded
  { toc_id, node_id, node_type, title, description, position_x, position_y }

TocNodeUpdated
  { toc_id, node_id, changes: { field: { old, new } } }

TocNodeRemoved
  { toc_id, node_id, reason }

TocEdgeCreated
  { toc_id, edge_id, from_node_id, to_node_id, label, assumption }

TocEdgeRemoved
  { toc_id, edge_id }

TocAssumptionStatusChanged
  { toc_id, assumption_id, old_status, new_status, evidence_notes }

TheoryOfChangeVersioned
  { toc_id, old_version, new_version, reason }
```

### Participant Identity Events

```
ParticipantRegistered
  { participant_id, organisation_id, pseudonym_id, demographics: {
      gender, ethnicity, primary_language, geographic_region } }

ParticipantPiiRecorded
  { participant_id, pii_fields: [field_names], encryption_key_id }
  -- PII stored separately; event metadata marks this as sensitive

ParticipantDemographicsUpdated
  { participant_id, changes: { field: { old, new } } }

ParticipantDeduplicationMatched
  { participant_id, matched_participant_id, match_confidence,
    match_factors: [name_similarity, dob_match, location_proximity] }

ParticipantsMerged
  { surviving_id, merged_id, merge_reason, merged_by }

ParticipantWithdrawn
  { participant_id, reason, withdrawn_at }

ParticipantDeceased
  { participant_id, reported_by }
```

### Consent Events

```
ConsentGranted
  { participant_id, consent_type, method, document_url, granted_at }

ConsentWithdrawn
  { participant_id, consent_type, withdrawn_at, reason }

ConsentExpired
  { participant_id, consent_type, expired_at }

DataAccessRequestReceived
  { request_id, participant_id, request_type, requested_at, due_by }

DataAccessRequestCompleted
  { request_id, completed_at, handled_by }
```

### Enrolment and Attendance Events

```
ParticipantEnrolled
  { enrolment_id, participant_id, programme_id, cohort_id,
    enrolment_date, referral_source }

ParticipantStatusChanged
  { enrolment_id, old_status, new_status, reason }

ParticipantExited
  { enrolment_id, exit_date, exit_reason }

ParticipantReenrolled
  { enrolment_id, new_enrolment_id, programme_id, reenrolment_date }

ActivityScheduled
  { activity_id, programme_id, name, activity_type, scheduled_date,
    duration_minutes, location, facilitator_id }

ActivityAttendanceRecorded
  { attendance_id, activity_id, enrolment_id, attended,
    attendance_type, recorded_by }

ActivityCancelled
  { activity_id, reason, cancelled_by }
```

### Data Collection Events

```
FormDefinitionCreated
  { form_id, organisation_id, programme_id, name, form_type, version }

FormDefinitionPublished
  { form_id, published_by }

FormSectionAdded
  { form_id, section_id, title, sort_order }

FormQuestionAdded
  { form_id, section_id, question_id, question_text, question_type,
    indicator_id, is_required, validation_rules }

FormSubmissionStarted
  { submission_id, form_id, enrolment_id, started_by,
    collection_method, device_id }

FormAnswerRecorded
  { submission_id, answer_id, question_id, value, value_type }
  -- value_type: "numeric", "text", "date", "boolean", "choice"

FormSubmissionCompleted
  { submission_id, submitted_at, gps_latitude, gps_longitude }

FormSubmissionValidated
  { submission_id, validated_by, validation_notes }

FormSubmissionFlagged
  { submission_id, flagged_by, reason }

FormSubmissionRejected
  { submission_id, rejected_by, reason }

-- Offline-specific events
OfflineSubmissionQueued
  { submission_id, device_id, queued_at, local_timestamp }

OfflineSubmissionSynced
  { submission_id, device_id, synced_at, conflict_resolved: boolean }
```

### Outcome Measurement Events

```
OutcomeMeasured
  { outcome_id, enrolment_id, indicator_id, measurement_period,
    measurement_date, value_numeric, value_text, value_categorical,
    form_submission_id, data_source }

OutcomeValidated
  { outcome_id, validated_by, validation_notes }

OutcomeCorrected
  { outcome_id, old_value, new_value, correction_reason, corrected_by }
  -- The original measurement event is preserved; this records the correction

DistanceTravelledCalculated
  { enrolment_id, indicator_id, baseline_value, baseline_date,
    current_value, current_date, absolute_change, percentage_change,
    standardised_change, reliable_change_index, is_reliable_change,
    direction }

ProgrammeOutcomeSummaryComputed
  { programme_id, indicator_id, cohort_id, reporting_period_start,
    reporting_period_end, participant_count, measured_count,
    mean_baseline, mean_endline, mean_change, median_change,
    improved_count, maintained_count, declined_count }

OutcomeAnomalyDetected
  { programme_id, indicator_id, cohort_id, anomaly_type, severity,
    description, detected_value, expected_value, deviation_score }

OutcomeAnomalyAcknowledged
  { anomaly_id, acknowledged_by, resolution_notes }
```

### Qualitative Analysis Events

```
QualitativeEvidenceRecorded
  { evidence_id, organisation_id, enrolment_id, programme_id,
    evidence_type, content_hash, word_count, language }
  -- Actual content stored in a separate content store, not in the event

ThematicCodeAssigned
  { evidence_id, code_id, code_label, confidence_score,
    assigned_by: "human" | "ai", excerpt }

ThematicCodeRemoved
  { evidence_id, code_id, removed_by, reason }

SentimentAnalysisCompleted
  { evidence_id, overall_sentiment, sentiment_score, model_version }

QualitativeClusterIdentified
  { cluster_id, programme_id, theme_label, evidence_ids: [],
    cluster_size, model_version }
```

### Framework Alignment Events

```
FrameworkImported
  { framework_id, name, version, framework_type, metric_count }

IndicatorDefined
  { indicator_id, organisation_id, code, name, measurement_type,
    unit_of_measure, direction, collection_frequency }

IndicatorMappedToFramework
  { indicator_id, framework_metric_id, mapping_confidence, mapped_by }

IndicatorUnmappedFromFramework
  { indicator_id, framework_metric_id, reason }
```

### Reporting Events

```
ReportRequested
  { report_id, template_id, programme_id, requested_by,
    reporting_period_start, reporting_period_end, format }

ReportGenerated
  { report_id, file_url, file_size_bytes, generation_duration_ms }

ReportFailed
  { report_id, error_message, failed_at }

ReportExported
  { report_id, exported_by, export_format }
```

---

## Read Model Projections

### Projection 1: Current Participant State

```sql
-- Materialised from: ParticipantRegistered, ParticipantDemographicsUpdated,
--   ParticipantsMerged, ParticipantWithdrawn, ConsentGranted, ConsentWithdrawn
CREATE TABLE rm_participants (
    id UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    pseudonym_id VARCHAR(100) NOT NULL,
    gender VARCHAR(50),
    ethnicity VARCHAR(100),
    primary_language VARCHAR(50),
    geographic_region VARCHAR(255),
    status VARCHAR(50),
    merged_into_id UUID,
    active_consent_types TEXT[],  -- array of currently active consent types
    programme_count INTEGER DEFAULT 0,
    first_enrolled_at TIMESTAMPTZ,
    last_activity_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ
);

CREATE INDEX idx_rm_participants_org ON rm_participants(organisation_id);
CREATE INDEX idx_rm_participants_status ON rm_participants(organisation_id, status);
```

### Projection 2: Programme Dashboard

```sql
-- Materialised from: ProgrammeCreated, ProgrammeActivated, ParticipantEnrolled,
--   ParticipantExited, ActivityScheduled, ActivityAttendanceRecorded,
--   OutcomeMeasured, ProgrammeOutcomeSummaryComputed
CREATE TABLE rm_programme_dashboard (
    programme_id UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    name VARCHAR(255),
    programme_type VARCHAR(100),
    status VARCHAR(50),
    start_date DATE,
    end_date DATE,
    -- Enrolment counts
    total_enrolled INTEGER DEFAULT 0,
    currently_active INTEGER DEFAULT 0,
    completed INTEGER DEFAULT 0,
    withdrawn INTEGER DEFAULT 0,
    lost_to_followup INTEGER DEFAULT 0,
    -- Activity stats
    total_activities INTEGER DEFAULT 0,
    total_attendance_records INTEGER DEFAULT 0,
    average_attendance_rate NUMERIC(5,2),
    -- Outcome stats
    indicators_tracked INTEGER DEFAULT 0,
    outcome_measurements_count INTEGER DEFAULT 0,
    participants_with_baseline INTEGER DEFAULT 0,
    participants_with_endline INTEGER DEFAULT 0,
    overall_improvement_rate NUMERIC(5,2),
    -- Alerts
    active_anomalies INTEGER DEFAULT 0,
    updated_at TIMESTAMPTZ
);

CREATE INDEX idx_rm_prog_org ON rm_programme_dashboard(organisation_id);
```

### Projection 3: Enrolment Timeline

```sql
-- Materialised from: ParticipantEnrolled, ParticipantStatusChanged,
--   ParticipantExited, ActivityAttendanceRecorded, FormSubmissionCompleted,
--   OutcomeMeasured
CREATE TABLE rm_enrolment_timeline (
    enrolment_id UUID PRIMARY KEY,
    participant_id UUID NOT NULL,
    programme_id UUID NOT NULL,
    organisation_id UUID NOT NULL,
    cohort_id UUID,
    cohort_name VARCHAR(255),
    enrolment_date DATE,
    exit_date DATE,
    exit_reason VARCHAR(100),
    current_status VARCHAR(50),
    -- Timeline summary
    days_enrolled INTEGER,
    activities_attended INTEGER DEFAULT 0,
    activities_missed INTEGER DEFAULT 0,
    attendance_rate NUMERIC(5,2),
    forms_submitted INTEGER DEFAULT 0,
    has_baseline BOOLEAN DEFAULT FALSE,
    has_midpoint BOOLEAN DEFAULT FALSE,
    has_endline BOOLEAN DEFAULT FALSE,
    -- Latest outcome summary
    indicators_measured INTEGER DEFAULT 0,
    indicators_improved INTEGER DEFAULT 0,
    indicators_maintained INTEGER DEFAULT 0,
    indicators_declined INTEGER DEFAULT 0,
    updated_at TIMESTAMPTZ
);

CREATE INDEX idx_rm_timeline_participant ON rm_enrolment_timeline(participant_id);
CREATE INDEX idx_rm_timeline_programme ON rm_enrolment_timeline(programme_id);
CREATE INDEX idx_rm_timeline_org ON rm_enrolment_timeline(organisation_id);
```

### Projection 4: Outcome Longitudinal View

```sql
-- Materialised from: OutcomeMeasured, OutcomeCorrected, OutcomeValidated,
--   DistanceTravelledCalculated
-- This is the key read model for the evidence continuity use case.
CREATE TABLE rm_outcome_longitudinal (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrolment_id UUID NOT NULL,
    participant_id UUID NOT NULL,
    programme_id UUID NOT NULL,
    organisation_id UUID NOT NULL,
    indicator_id UUID NOT NULL,
    indicator_name VARCHAR(255),
    indicator_code VARCHAR(50),
    measurement_period VARCHAR(50),
    measurement_date DATE,
    value_numeric NUMERIC,
    value_text TEXT,
    value_categorical VARCHAR(255),
    is_validated BOOLEAN DEFAULT FALSE,
    is_corrected BOOLEAN DEFAULT FALSE,
    -- Pre-computed distance from baseline
    baseline_value NUMERIC,
    absolute_change_from_baseline NUMERIC,
    percentage_change_from_baseline NUMERIC,
    -- Source tracing
    form_submission_id UUID,
    data_source VARCHAR(100),
    event_id UUID,  -- back-reference to the originating event
    created_at TIMESTAMPTZ
);

CREATE INDEX idx_rm_outcome_enrolment ON rm_outcome_longitudinal(enrolment_id);
CREATE INDEX idx_rm_outcome_programme ON rm_outcome_longitudinal(programme_id, indicator_id);
CREATE INDEX idx_rm_outcome_indicator ON rm_outcome_longitudinal(indicator_id, measurement_date);
CREATE INDEX idx_rm_outcome_org ON rm_outcome_longitudinal(organisation_id);
-- Composite for distance-travelled queries
CREATE INDEX idx_rm_outcome_distance ON rm_outcome_longitudinal(
    enrolment_id, indicator_id, measurement_period
);
```

### Projection 5: Framework Alignment Report

```sql
-- Materialised from: IndicatorDefined, IndicatorMappedToFramework,
--   ProgrammeOutcomeSummaryComputed
CREATE TABLE rm_framework_alignment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    framework_id UUID NOT NULL,
    framework_name VARCHAR(255),
    framework_metric_id UUID NOT NULL,
    framework_metric_code VARCHAR(100),
    framework_metric_name VARCHAR(255),
    indicator_id UUID NOT NULL,
    indicator_code VARCHAR(50),
    indicator_name VARCHAR(255),
    mapping_confidence VARCHAR(20),
    -- Latest programme-level data
    programme_id UUID,
    programme_name VARCHAR(255),
    latest_value NUMERIC,
    latest_measurement_date DATE,
    target_value NUMERIC,
    target_achieved BOOLEAN,
    updated_at TIMESTAMPTZ
);

CREATE INDEX idx_rm_fw_org ON rm_framework_alignment(organisation_id);
CREATE INDEX idx_rm_fw_framework ON rm_framework_alignment(framework_id);
```

### Projection 6: Qualitative Evidence Explorer

```sql
-- Materialised from: QualitativeEvidenceRecorded, ThematicCodeAssigned,
--   SentimentAnalysisCompleted
CREATE TABLE rm_qualitative_explorer (
    evidence_id UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    programme_id UUID,
    programme_name VARCHAR(255),
    participant_pseudonym VARCHAR(100),
    evidence_type VARCHAR(50),
    content_preview VARCHAR(500),  -- first 500 chars for listing views
    word_count INTEGER,
    language VARCHAR(10),
    recorded_date DATE,
    -- Thematic codes (denormalised for fast filtering)
    thematic_codes TEXT[],  -- array of code labels
    primary_theme VARCHAR(255),
    -- Sentiment
    overall_sentiment VARCHAR(20),
    sentiment_score NUMERIC(4,3),
    -- Search
    content_tsv TSVECTOR,
    updated_at TIMESTAMPTZ
);

CREATE INDEX idx_rm_qual_org ON rm_qualitative_explorer(organisation_id);
CREATE INDEX idx_rm_qual_programme ON rm_qualitative_explorer(programme_id);
CREATE INDEX idx_rm_qual_theme ON rm_qualitative_explorer USING GIN (thematic_codes);
CREATE INDEX idx_rm_qual_sentiment ON rm_qualitative_explorer(overall_sentiment);
CREATE INDEX idx_rm_qual_tsv ON rm_qualitative_explorer USING GIN (content_tsv);
```

### Projection 7: Audit and Compliance View

```sql
-- Materialised from: ALL events (filtered by type for compliance-relevant actions)
CREATE TABLE rm_audit_trail (
    id BIGSERIAL PRIMARY KEY,
    organisation_id UUID NOT NULL,
    event_id UUID NOT NULL,
    event_type VARCHAR(200) NOT NULL,
    stream_type VARCHAR(100),
    stream_id VARCHAR(255),
    actor_user_id UUID,
    actor_email VARCHAR(255),
    action_summary TEXT,  -- human-readable description
    entity_type VARCHAR(100),
    entity_id UUID,
    -- Sensitive data flag
    involves_pii BOOLEAN DEFAULT FALSE,
    involves_consent BOOLEAN DEFAULT FALSE,
    -- Time
    occurred_at TIMESTAMPTZ NOT NULL,
    -- Searchable
    action_tsv TSVECTOR
);

CREATE INDEX idx_rm_audit_org ON rm_audit_trail(organisation_id, occurred_at);
CREATE INDEX idx_rm_audit_entity ON rm_audit_trail(entity_type, entity_id);
CREATE INDEX idx_rm_audit_user ON rm_audit_trail(actor_user_id);
CREATE INDEX idx_rm_audit_pii ON rm_audit_trail(organisation_id, involves_pii)
    WHERE involves_pii = TRUE;
```

---

## Command Handlers (Write Side)

### Example: Enrol Participant Command

```python
# Pseudocode for the command handler pattern

class EnrolParticipantCommand:
    participant_id: UUID
    programme_id: UUID
    cohort_id: Optional[UUID]
    enrolment_date: date
    referral_source: Optional[str]
    enrolled_by: UUID

class ProgrammeEnrolmentAggregate:
    """Aggregate root for programme enrolments."""
    
    def __init__(self, stream_id: str):
        self.stream_id = stream_id
        self.enrolments = {}
        self.status = None
    
    def enrol_participant(self, cmd: EnrolParticipantCommand):
        # Business rule validation
        if cmd.participant_id in self.enrolments:
            existing = self.enrolments[cmd.participant_id]
            if existing.status == 'active':
                raise DomainError("Participant already enrolled and active")
        
        # Emit event
        return ParticipantEnrolled(
            enrolment_id=uuid4(),
            participant_id=cmd.participant_id,
            programme_id=cmd.programme_id,
            cohort_id=cmd.cohort_id,
            enrolment_date=cmd.enrolment_date,
            referral_source=cmd.referral_source
        )
    
    def apply(self, event):
        """Rebuild state from events."""
        if isinstance(event, ParticipantEnrolled):
            self.enrolments[event.participant_id] = EnrolmentState(
                enrolment_id=event.enrolment_id,
                status='enrolled',
                enrolment_date=event.enrolment_date
            )
        elif isinstance(event, ParticipantExited):
            self.enrolments[event.participant_id].status = 'exited'
```

### Example: Record Outcome Measurement Command

```python
class RecordOutcomeCommand:
    enrolment_id: UUID
    indicator_id: UUID
    measurement_period: str
    measurement_date: date
    value_numeric: Optional[Decimal]
    value_text: Optional[str]
    form_submission_id: Optional[UUID]
    recorded_by: UUID

class OutcomeAggregate:
    """Aggregate root for a participant's outcome measurements."""
    
    def __init__(self, stream_id: str):
        self.stream_id = stream_id
        self.measurements = {}  # keyed by (indicator_id, measurement_period)
    
    def record_outcome(self, cmd: RecordOutcomeCommand):
        key = (cmd.indicator_id, cmd.measurement_period)
        
        if key in self.measurements:
            # This is a correction, not a new measurement
            old = self.measurements[key]
            return OutcomeCorrected(
                outcome_id=old.outcome_id,
                old_value=old.value,
                new_value=cmd.value_numeric or cmd.value_text,
                correction_reason="Updated measurement",
                corrected_by=cmd.recorded_by
            )
        
        return OutcomeMeasured(
            outcome_id=uuid4(),
            enrolment_id=cmd.enrolment_id,
            indicator_id=cmd.indicator_id,
            measurement_period=cmd.measurement_period,
            measurement_date=cmd.measurement_date,
            value_numeric=cmd.value_numeric,
            value_text=cmd.value_text,
            form_submission_id=cmd.form_submission_id,
            data_source="form_submission" if cmd.form_submission_id else "manual_entry"
        )
```

---

## Event Processing Pipeline

```
                    ┌─────────────────────┐
                    │   Command Handlers  │
                    │   (Write Side)      │
                    └────────┬────────────┘
                             │ append events
                             ▼
                    ┌─────────────────────┐
                    │    Event Store      │
                    │  (append-only log)  │
                    └────────┬────────────┘
                             │ subscribe
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │  Projection  │ │  Projection  │ │  Projection  │
    │  Workers     │ │  Workers     │ │  Workers     │
    │              │ │              │ │              │
    │ Participant  │ │  Programme   │ │  Outcome     │
    │ State        │ │  Dashboard   │ │  Longitudinal│
    └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
           │                │                │
           ▼                ▼                ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │  Read Model  │ │  Read Model  │ │  Read Model  │
    │  (PostgreSQL)│ │  (PostgreSQL)│ │  (PostgreSQL)│
    └──────────────┘ └──────────────┘ └──────────────┘
           │                │                │
           └────────────────┼────────────────┘
                            ▼
                   ┌─────────────────┐
                   │  Query Handlers │
                   │  (Read Side)    │
                   └─────────────────┘
```

---

## Offline Synchronisation Strategy

One of the most critical challenges for the Impact Measurement Platform is offline data collection in field environments. Event sourcing provides an elegant solution:

### Offline Event Queue

```sql
-- On the mobile device (SQLite or IndexedDB):
CREATE TABLE local_event_queue (
    local_sequence INTEGER PRIMARY KEY AUTOINCREMENT,
    event_id TEXT NOT NULL,          -- UUID generated locally
    stream_id TEXT NOT NULL,
    event_type TEXT NOT NULL,
    data TEXT NOT NULL,              -- JSON payload
    device_id TEXT NOT NULL,
    local_timestamp TEXT NOT NULL,   -- device clock time
    sync_status TEXT DEFAULT 'pending',  -- pending, syncing, synced, conflict
    server_position INTEGER          -- assigned after sync
);
```

### Sync Protocol

1. **Offline**: Data collector records form submissions as events in the local queue. Each event gets a locally generated UUID and a device-local timestamp.

2. **Online**: Device sends queued events to the server in batch. The server validates each event and appends it to the event store, assigning global positions and server timestamps.

3. **Conflict resolution**: Because events are immutable facts ("this answer was recorded at this time on this device"), there are no write conflicts in the traditional sense. If two devices record answers for the same participant, both events are stored. A reconciliation projection flags duplicate submissions for manual review.

4. **Idempotency**: The locally generated `event_id` serves as an idempotency key. If a sync is interrupted and retried, duplicate events are detected and skipped.

---

## Temporal Queries: The Core Advantage

The event store enables temporal queries that are impossible or extremely difficult with a traditional CRUD database:

### "What was the participant's state on a specific date?"

```python
def participant_state_at(participant_id: str, as_of: datetime):
    events = event_store.read_stream(
        stream_id=f"participant-{participant_id}",
        up_to=as_of
    )
    aggregate = ParticipantAggregate()
    for event in events:
        aggregate.apply(event)
    return aggregate.current_state()
```

### "What changed between two assessment dates?"

```python
def outcome_changes_between(enrolment_id: str, start: date, end: date):
    events = event_store.read_stream(
        stream_id=f"outcomes-{enrolment_id}",
        after=start,
        up_to=end
    )
    return [e for e in events if e.event_type in (
        'OutcomeMeasured', 'OutcomeCorrected'
    )]
```

### "Replay the complete evidence trail for a funder audit"

```python
def evidence_trail(programme_id: str, participant_id: str):
    # All events across all streams for this participant in this programme
    return event_store.read_by_correlation(
        correlation_filters={
            'programme_id': programme_id,
            'participant_id': participant_id
        },
        event_types=[
            'ParticipantEnrolled', 'FormSubmissionCompleted',
            'FormAnswerRecorded', 'OutcomeMeasured',
            'DistanceTravelledCalculated'
        ]
    )
```

---

## Pros and Cons

### Pros

1. **Perfect audit trail by design**: Every state change is an immutable event. There is no possibility of data being silently overwritten. This is exactly what funders and regulators need when they ask "show me the evidence trail."

2. **Native temporal modeling**: The platform's core purpose is measuring change over time. Event sourcing stores time-series data natively -- every measurement is an event with a timestamp, and historical queries are first-class operations rather than afterthoughts.

3. **Elegant offline sync**: Because events are immutable facts about what happened at a point in time, offline data collection becomes a simple queue-and-merge operation. There are no update conflicts because nothing is ever updated -- new events are simply appended.

4. **Evidence continuity is structural**: The chain from data collection through outcome measurement to reporting is captured in the event stream itself. The evidence trail is not reconstructed from current state; it IS the primary data store.

5. **Flexible read models**: Different stakeholders need different views of the same data. Programme managers need dashboards; evaluators need longitudinal outcome tables; funders need framework-aligned reports. Each view is a purpose-built projection that can evolve independently without affecting the source of truth.

6. **Correction without data loss**: When an outcome measurement is corrected, both the original and the correction are preserved as separate events. The read model shows the current value, but the event store retains the full history for audit purposes.

7. **Causal attribution support**: The complete event history for a participant -- enrolment, attendance, assessments, outcomes -- enables sophisticated causal analysis that requires knowing not just what changed but when each intervention occurred.

### Cons

1. **Operational complexity**: Event sourcing requires operating an event store, message broker, projection workers, and read model databases. This is significantly more infrastructure than a single PostgreSQL instance, which is a real concern for resource-constrained nonprofits.

2. **Eventual consistency**: Read models are updated asynchronously. After a form is submitted, the programme dashboard may take seconds to reflect the new data. For a platform where data collectors need immediate confirmation that their submission was recorded, this latency must be carefully managed.

3. **Projection rebuild cost**: If a projection has a bug or a new projection is needed, rebuilding from the event store can take hours or days for organisations with years of historical data. Snapshots mitigate this but add complexity.

4. **Event schema evolution**: As the platform evolves, event schemas change. Old events must remain readable, requiring version-aware deserialisation (upcasters). This is manageable but adds ongoing maintenance burden.

5. **Query complexity**: Ad-hoc queries that would be simple JOINs in a relational model (e.g., "show me all participants in Programme X with improved outcomes on Indicator Y") require either a purpose-built projection or reading from the event store and processing in application code.

6. **Team skill requirements**: Event sourcing is unfamiliar to most developers. An open-source project targeting the nonprofit sector needs contributors who can work with the pattern, which narrows the contributor pool.

7. **Storage growth**: Storing every event forever means storage grows monotonically. For a platform tracking thousands of participants over years, the event store will grow to hundreds of gigabytes. Compression and archival strategies are needed.

8. **GDPR erasure tension**: GDPR "right to erasure" conflicts with the immutability of the event store. The solution (crypto-shredding -- encrypting PII with per-participant keys and destroying the key) adds significant complexity.

---

## Migration and Scaling Considerations

### Initial Deployment

A PostgreSQL-based event store (as shown in the schema above) is sufficient for initial deployments. This avoids requiring EventStoreDB or Kafka, keeping infrastructure simple. The event store table, projection checkpoint table, and read model tables can all live in a single PostgreSQL instance.

### Growth Path

**10K-50K participants:**
- Add Redis caching for hot read models (dashboard projections)
- Implement snapshots for aggregates with long event histories (participants enrolled in multiple programmes over years)
- Partition the `event_store` table by `created_at` month

**50K-200K participants:**
- Migrate to EventStoreDB or Kafka for the event store to get native subscription support and better write throughput
- Deploy projection workers as independent services that can scale horizontally
- Use separate PostgreSQL instances for read models vs. event store

**200K+ participants:**
- Shard event store by organisation (each org is an independent stream namespace)
- Deploy read model databases per region for self-hosted deployments
- Implement event archival: move events older than N years to cold storage (S3/Parquet) while maintaining read model state

### Migration from Existing Systems

Migrating data from Salesforce, Apricot, or other incumbent systems into an event-sourced architecture requires generating "seed events" from the imported data:

```python
# For each imported participant record, generate a ParticipantRegistered event
# with metadata indicating it was imported, not organically created
for record in imported_records:
    event = ParticipantRegistered(
        participant_id=record.id,
        organisation_id=org_id,
        pseudonym_id=generate_pseudonym(record),
        demographics=extract_demographics(record),
        metadata={
            'source': 'salesforce_import',
            'import_batch_id': batch_id,
            'original_id': record.salesforce_id,
            'imported_at': datetime.now()
        }
    )
    event_store.append(event)
```

### GDPR Crypto-Shredding

To handle erasure requests while maintaining event immutability:

1. PII in events is encrypted with a per-participant key stored in a key management service
2. Non-PII event data (outcome values, attendance records) is stored in plain text
3. On erasure request, the participant's encryption key is destroyed
4. Events remain in the store but PII fields become unreadable
5. Read models are updated to remove/anonymise the participant's data
6. Aggregate statistics and anonymised outcome data are preserved
