# Impact Measurement Platform — Phased Development Plan

> Project: 447-impact-measurement-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four
`data-model-suggestion-*.md` files. The persistence layer follows **data-model-suggestion-3
(Hybrid Relational + JSONB on PostgreSQL)**: normalised tables for stable entities
(organisations, users, participants, enrolments, indicators, outcome records) and JSONB columns
for variable, evolving structures (form definitions, answers, theory-of-change graphs, report
configs). This delivers the referential integrity that solves the *evidence continuity problem*
while keeping schema flexibility and a single, low-ops engine that resource-constrained nonprofits
can self-host. The audit-trail strengths of the event-sourced model (suggestion 2) are captured via
a PROV-DM-aligned `audit_log` table and an append-only `outcome_records` correction pattern, rather
than full event sourcing — avoiding the operational complexity that suggestion 2 itself flags as a
risk for the target sector.

---

## Core Requirements (synthesised)

- **What it does**: Unifies the full impact-evidence lifecycle — theory of change → data collection
  → outcome measurement → funder-ready reporting — under persistent participant identities so
  nonprofits can demonstrate *change*, not just *activity counts*.
- **Primary users**: Programme managers and evaluation directors at mid-sized nonprofits;
  field data collectors (offline mobile); org admins (privacy/consent); funders/board (report
  consumers); API clients (CRM integrations).
- **Key differentiators**: (1) unbreakable baseline→endline evidence chain; (2) AI-native
  qualitative analysis (clustering, thematic coding, sentiment) at scale; (3) probabilistic
  participant matching/dedup; (4) framework alignment to IRIS+, SDG, CIDS; (5) offline-first
  field collection; (6) natural-language querying.
- **Deployment**: Self-hosted (Docker Compose) and cloud, single PostgreSQL engine. Offline-first
  PWA for field collection.
- **Integration surface**: REST API (OpenAPI 3.1), webhooks, CSV/XLSForm import, OData v4 export,
  CIDS JSON-LD export, LLM providers (OpenAI/Anthropic/local via abstraction), KoboToolbox/ODK,
  Salesforce/Apricot.
- **Standards** (from `standards.md`): OpenAPI 3.1 + JSON Schema 2020-12; RFC 7807 errors; OAuth 2.0
  / OIDC; XLSForm import; OData v4 export; CIDS v3.2 JSON-LD export; IRIS+ / SDG / IMP Five
  Dimensions alignment; W3C PROV-DM provenance; ISO 27701-aligned privacy (GDPR pseudonymisation,
  consent, DSAR); FHIR R4 SDOH (deferred to backlog).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | Python 3.12 | The differentiating features are AI/ML-heavy (LLM qualitative analysis, embeddings dedup, time-series anomaly detection, causal inference). Python has the strongest ecosystem (`anthropic`/`openai`, `scikit-learn`, `pandas`, `recordlinkage`, `statsmodels`) and `pyodk` for ODK interop. |
| API framework | FastAPI | First-class async, Pydantic v2 models, and automatic OpenAPI 3.1 generation (a hard requirement from `standards.md`). RFC 7807 errors integrate cleanly via exception handlers. |
| Validation / schemas | Pydantic v2 | Single source of truth for request/response models, JSONB document validation, and JSON Schema 2020-12 export embedded in the OpenAPI doc. |
| Database | PostgreSQL 16 | Implements suggestion-3 hybrid model: normalised tables + JSONB with GIN indexes, `tsvector` full-text search, row-level security for multi-tenancy, `pgvector` extension for embedding-based dedup and qualitative clustering — all in one engine. Zero extra infra for self-hosters. |
| Vector search | pgvector extension | Stores participant dedup embeddings and qualitative-evidence embeddings; avoids a separate vector DB. |
| ORM / migrations | SQLAlchemy 2.0 (async) + Alembic | Native JSONB support, async sessions for FastAPI, version-controlled migrations safe for self-hosted upgrades. |
| Task queue | Celery + Redis | Async workloads: LLM analysis, report rendering, dedup batch jobs, anomaly scans, large imports. Redis doubles as cache and Celery broker/result backend. |
| Cache / broker | Redis 7 | Dashboard caching, Celery transport, rate-limit counters, session store. |
| Frontend (web) | Next.js 15 (React, TypeScript) + Tailwind + shadcn/ui | Server-rendered dashboards, ToC visual builder (React Flow), report previews. TypeScript types generated from the OpenAPI doc (`openapi-typescript`). |
| ToC graph editor | React Flow (`@xyflow/react`) | Node/edge canvas maps directly to the ToC JSONB `graph` document. |
| Mobile data collection | PWA (Next.js) + IndexedDB (Dexie.js) + Service Worker | Offline-first form fill in low-connectivity field environments; background sync queue replays submissions when online. Avoids native-app store friction. |
| Charts | Recharts | Outcome trend lines, distance-travelled bars, cohort comparisons in dashboards and reports. |
| Report rendering | Jinja2 → HTML → WeasyPrint (PDF); `python-docx` (Word); `openpyxl` (Excel) | Funder-ready PDF/Word/Excel with org branding from a single HTML template source where possible. |
| LLM access | Provider abstraction (`LLMClient`) over Anthropic + OpenAI + local (Ollama) | Nonprofits have varied budgets/data-sovereignty needs; abstraction lets self-hosters run local models. Prompt caching enabled for repeated system prompts. |
| Embeddings | `sentence-transformers` (local) or provider embeddings | Local default keeps PII embeddings on-prem for dedup. |
| Auth | OAuth 2.0 + OIDC (Authlib); JWT access tokens; API keys for machine clients | `standards.md` requires OAuth/OIDC for SSO (Entra ID, Google Workspace) and OAuth for partner API access. |
| Testing | pytest + pytest-asyncio + httpx.AsyncClient; Playwright (frontend E2E); testcontainers-python (real Postgres) | Unit, mocked-integration, real-DB integration, and browser E2E coverage. |
| Code quality | ruff (lint + format), mypy (types), pre-commit | Standard, fast Python toolchain. Frontend: eslint + prettier + tsc. |
| Packaging | uv (Python), pnpm (frontend) | Fast, reproducible dependency management. |
| Containerisation | Docker + docker-compose | One-command self-host: api, worker, db (pgvector image), redis, web. |
| CI | GitHub Actions | Lint, type-check, test matrix, Docker build, Alembic migration check. |

### Project Structure

```
impact-measurement-platform/
├── docker-compose.yml
├── docker-compose.prod.yml
├── .github/workflows/ci.yml
├── README.md
├── backend/
│   ├── pyproject.toml
│   ├── Dockerfile
│   ├── alembic.ini
│   ├── alembic/
│   │   └── versions/
│   ├── app/
│   │   ├── main.py                  # FastAPI app factory, router registration, RFC7807 handlers
│   │   ├── config.py                # Pydantic Settings (env vars)
│   │   ├── db/
│   │   │   ├── session.py           # async engine + session, RLS org context
│   │   │   ├── base.py              # SQLAlchemy declarative base + mixins
│   │   │   └── models/              # ORM models (organisation, user, participant, ...)
│   │   ├── schemas/                 # Pydantic request/response + JSONB document models
│   │   ├── api/
│   │   │   ├── deps.py              # auth, current_user, current_org, pagination
│   │   │   └── routes/             # one module per resource
│   │   ├── services/                # business logic (enrolment, outcomes, distance_travelled...)
│   │   ├── ai/
│   │   │   ├── llm_client.py        # provider abstraction
│   │   │   ├── qualitative.py       # clustering / coding / sentiment
│   │   │   ├── dedup.py             # probabilistic record linkage
│   │   │   ├── anomaly.py           # time-series anomaly detection
│   │   │   └── nlq.py               # natural-language query → SQL/chart
│   │   ├── integrations/            # xlsform, csv, odk, salesforce, cids, odata, webhooks
│   │   ├── reporting/               # template engine + pdf/docx/xlsx renderers
│   │   ├── security/                # crypto (PII encryption), consent, dsar, rbac, audit
│   │   └── workers/                 # celery app + tasks
│   ├── frameworks/                  # seed data: IRIS+, SDG, IMP, logic-model templates (JSON)
│   └── tests/
│       ├── unit/
│       ├── integration/
│       ├── e2e/
│       └── fixtures/
└── frontend/
    ├── package.json
    ├── Dockerfile
    ├── src/
    │   ├── app/                     # Next.js App Router pages
    │   ├── components/              # shared UI (shadcn)
    │   ├── features/                # toc-builder, forms, dashboards, reports, participants
    │   ├── lib/api/                 # generated OpenAPI client + types
    │   ├── lib/offline/             # Dexie schema, sync queue, service worker
    │   └── styles/
    └── tests/
        └── e2e/                     # Playwright specs
```

---

## Phase 1: Foundation — Project Skeleton, Multi-Tenancy, Auth

### Purpose
Establish the runnable backbone: containerised FastAPI + PostgreSQL (pgvector) + Redis, the base
SQLAlchemy/Alembic setup, organisation-scoped multi-tenancy with row-level security, user
authentication (password + JWT), RBAC, RFC 7807 error handling, and the audit-log spine. After this
phase the system boots, an org and admin user can be created, and every authenticated request is
tenant-scoped and audited.

### Tasks

#### 1.1 — Application scaffold, config, and Docker
**What**: Bootable FastAPI app with config from env vars and a working `docker-compose up`.

**Design**:
- `app/config.py` using `pydantic-settings`:
  ```python
  class Settings(BaseSettings):
      database_url: str
      redis_url: str = "redis://redis:6379/0"
      jwt_secret: str
      jwt_access_ttl_seconds: int = 3600
      pii_encryption_key: str            # base64 32-byte key (Fernet/AES-GCM)
      llm_provider: Literal["anthropic","openai","local"] = "anthropic"
      llm_api_key: str | None = None
      cors_origins: list[str] = ["http://localhost:3000"]
      model_config = SettingsConfigDict(env_prefix="IMP_", env_file=".env")
  ```
- `app/main.py`: `create_app()` factory registers routers, CORS, exception handlers; `GET /health`
  returns `{"status":"ok","db":bool,"redis":bool}`.
- `docker-compose.yml` services: `db` (image `pgvector/pgvector:pg16`), `redis`, `api`, `worker`, `web`.

**Testing**:
- `Unit: Settings loads from env → fields populated, defaults applied`
- `Unit: missing IMP_DATABASE_URL → ValidationError naming the field`
- `Integration (real): GET /health with db+redis up → 200 {status:ok,db:true,redis:true}`
- `E2E: docker-compose up → api container healthy, /health reachable on :8000`

#### 1.2 — Database base, organisations, RLS, migrations
**What**: Alembic-managed schema for `organisations` and `users` with row-level security.

**Design** (per suggestion-3):
- `organisations(id uuid pk, name, slug unique, settings jsonb default '{}', country_code char(2),
  created_at, updated_at, deleted_at)` + GIN index on `settings`.
- `users(id uuid pk, organisation_id fk, email, password_hash, first_name, last_name,
  role check in ('org_admin','programme_manager','evaluator','data_collector','report_viewer','api_client'),
  preferences jsonb default '{}', is_active bool, last_login_at, created_at, updated_at,
  unique(organisation_id,email))`.
- RLS: enable on every tenant table; policy `USING (organisation_id = current_setting('app.current_org_id')::uuid)`.
  `db/session.py` sets `SET LOCAL app.current_org_id = :org` at the start of each request transaction.
- `TimestampMixin` (created_at/updated_at) and `SoftDeleteMixin` (deleted_at) in `db/base.py`.

**Testing**:
- `Unit: slug uniqueness validator rejects duplicate slug`
- `Integration (real): create two orgs, set app.current_org_id=A, SELECT users → only org A rows`
- `Integration (real): query without org context set → RLS returns zero rows (fail-closed)`
- `Integration (real): alembic upgrade head then downgrade base → no errors, schema empty`

#### 1.3 — Authentication, JWT, API keys
**What**: Password login issuing JWTs, API-key auth for machine clients, current-user dependency.

**Design**:
- `POST /auth/login {email,password}` → `{access_token, token_type:"bearer", expires_in}`. Passwords
  hashed with `argon2`. JWT claims: `sub`(user_id), `org`(organisation_id), `role`, `exp`.
- `api/deps.py`: `get_current_user`, `get_current_org`, `require_role(*roles)`. API keys stored as
  `api_keys(id, organisation_id, name, key_hash, scopes jsonb, last_used_at, revoked_at)`; presented
  via `Authorization: Bearer imp_<key>` or `X-API-Key`.
- OAuth2/OIDC SSO scaffolding (Authlib) with provider config in org settings — deferred wiring to
  backlog but interface stubbed.

**Testing**:
- `Unit: argon2 verify correct/incorrect password → True/False`
- `Unit: expired JWT → 401 with RFC7807 body`
- `Integration (mocked): login valid creds → 200 token; decoding yields correct org+role claims`
- `Integration: protected route without token → 401; with valid token → 200`
- `Integration: require_role('org_admin') hit by 'data_collector' → 403`

#### 1.4 — RFC 7807 errors and audit log
**What**: Standard machine-readable errors and a PROV-DM-aligned audit trail.

**Design**:
- Exception handler emits `application/problem+json`: `{type,title,status,detail,instance,errors?}`.
- `audit_log(id bigserial, organisation_id, user_id, action check in ('create','read','update',
  'delete','export','login','logout','consent_change','data_access_request'), entity_type,
  entity_id uuid, old_values jsonb, new_values jsonb, ip_address inet, user_agent, created_at)`.
  Indices on `(organisation_id)`, `(entity_type,entity_id)`, `(created_at)`.
- PROV mapping documented: `user_id`=Agent, `action`=Activity, `entity_*`=Entity.
- `security/audit.py: record_audit(...)` called from a service decorator on mutating operations.

**Testing**:
- `Unit: validation error → problem+json with errors[] listing field + message`
- `Unit: record_audit serialises old/new to jsonb, sets action+entity`
- `Integration: create participant → audit_log row action=create, new_values populated`
- `Integration: 404 unknown route → problem+json status=404`

---

## Phase 2: Domain Core — Programmes, Participants, Enrolments, Indicators

### Purpose
Build the stable relational backbone that all evidence links to. After this phase an org can define
programmes and cohorts, register privacy-preserving participants (encrypted PII + JSONB
demographics), enrol them, and define outcome indicators — the entities the evidence chain depends
on. This is the heart of the *evidence continuity* guarantee.

### Tasks

#### 2.1 — Programmes and cohorts
**What**: CRUD for programmes (with JSONB `config`) and cohorts.

**Design** (suggestion-3 schema):
- `programmes(id, organisation_id fk, name, description, programme_type, status check
  ('draft','active','paused','completed','archived'), start_date, end_date, config jsonb,
  created_by, timestamps)`; GIN index on `config`. `config` holds `target_participant_count`,
  `budget`, `measurement_schedule` (baseline/midpoint/endline/followups with timing+window_days),
  `eligibility_criteria`, `tags`.
- `cohorts(id, programme_id fk, name, description, start_date, end_date, is_control_group,
  metadata jsonb)`.
- Endpoints: `POST/GET/PATCH /programmes`, `GET /programmes/{id}`, nested `/programmes/{id}/cohorts`.
- Pydantic `MeasurementSchedule`, `EligibilityRule` validate the `config` JSONB shape.

**Testing**:
- `Unit: MeasurementSchedule with negative window_days → ValidationError`
- `Unit: programme status transition draft→active allowed; completed→draft rejected`
- `Integration: POST /programmes → 201; GET lists only current org's programmes`
- `Integration: create cohort under another org's programme → 404 (RLS)`

#### 2.2 — Participants with encrypted PII and pseudonymisation
**What**: Participant registry with encrypted PII, JSONB demographics, dedup hash fields.

**Design**:
- `participants(id, organisation_id fk, pseudonym_id unique-per-org, pii_encrypted bytea,
  demographics jsonb, dedup_hashes jsonb, dedup_embedding vector(384), dedup_cluster_id uuid,
  status check ('active','inactive','withdrawn','deceased','merged'), merged_into_id fk,
  timestamps)`. GIN index on `demographics`; ivfflat index on `dedup_embedding`.
- `security/crypto.py`: AES-256-GCM envelope encryption. `encrypt_pii(dict)->bytes`,
  `decrypt_pii(bytes)->dict`. Per-participant data key wrapped by master key (`pii_encryption_key`).
- `pseudonym_id` auto-generated (`P-` + base32(random)). PII (`first_name,last_name,date_of_birth,
  email,phone`) never stored plaintext; demographics (gender, age_group, ethnicity, region,
  postcode_area, custom_fields) stored in JSONB for disaggregation.
- Endpoints: `POST /participants` (accepts PII, returns pseudonym + demographics only unless caller
  has `read_pii` scope), `GET /participants` (list, never returns PII), `GET /participants/{id}/pii`
  (gated, audited).

**Testing**:
- `Unit: encrypt_pii then decrypt_pii round-trips dict exactly`
- `Unit: tampered ciphertext → decryption raises (GCM auth fail)`
- `Integration: POST participant → 201, response has no plaintext name; pii_encrypted non-null in DB`
- `Integration: GET /participants/{id}/pii as report_viewer → 403; as evaluator with scope → 200 + audit row`

#### 2.3 — Enrolments and re-enrolment
**What**: Link participants to programmes/cohorts with temporal status tracking.

**Design**:
- `programme_enrolments(id, participant_id fk, programme_id fk, cohort_id fk, enrolment_date,
  exit_date, exit_reason, status check ('enrolled','active','on_hold','completed','withdrawn',
  'lost_to_followup'), intake_data jsonb, timestamps)`. GIN on `intake_data`.
- Service rule: a participant may have only one non-terminal enrolment per programme; re-enrolment
  after a terminal status creates a new row, preserving history.
- Endpoint: `POST /programmes/{id}/enrolments`, `PATCH /enrolments/{id}` (status/exit transitions
  validated by a state machine).

**Testing**:
- `Unit: enrol participant already actively enrolled in same programme → DomainError`
- `Unit: status transition completed→active rejected; lost_to_followup→active (re-engage) allowed`
- `Integration: enrol → 201; second active enrol same programme → 409 problem+json`
- `Integration: exit then re-enrol → two rows, first status=completed`

#### 2.4 — Indicators and measurement definitions
**What**: Outcome indicator catalogue with measurement metadata.

**Design**:
- `indicators(id, organisation_id fk, code, name, description, measurement_type check
  ('numeric','percentage','scale','boolean','categorical','text'), unit_of_measure, direction check
  ('increase','decrease','maintain','target'), data_source, collection_frequency, is_core,
  is_active, scale_config jsonb, timestamps)`. `scale_config` holds min/max/labels for scale types.
- Endpoints: `POST/GET/PATCH /indicators`. Validation: scale indicators require `scale_config`.

**Testing**:
- `Unit: measurement_type=scale without scale_config → ValidationError`
- `Unit: direction defaults to 'increase'`
- `Integration: create indicator → 201; duplicate (org,code) → 409`

---

## Phase 3: Data Collection — Form Builder, Submissions, Outcome Capture

### Purpose
Deliver the primary value loop: build configurable survey/assessment instruments, collect responses
(web), and persist typed answers that feed outcome records. After this phase an org can author a
baseline form, collect a submission, and have outcome values recorded against indicators — the first
link of the evidence chain becomes operational.

### Tasks

#### 3.1 — Form definitions as JSONB documents
**What**: Versioned form instruments stored as validated JSONB documents.

**Design** (suggestion-3 JSONB-heavy tier):
- `form_definitions(id, organisation_id fk, programme_id fk null, name, form_type check
  ('baseline_assessment','midpoint_assessment','endline_assessment','followup_survey','intake_form',
  'exit_form','feedback_360','case_note','custom'), version, status check ('draft','published',
  'archived'), is_template, schema jsonb, timestamps)`.
- `schema` JSONB validated by Pydantic `FormSchema`:
  ```python
  class Choice(BaseModel): label: str; value: str; is_other: bool = False
  class Question(BaseModel):
      id: str; text: str; help_text: str | None = None
      type: Literal['text','textarea','number','decimal','date','single_choice',
                    'multiple_choice','likert_scale','ranking','file_upload',
                    'geolocation','barcode','signature','matrix','calculated']
      required: bool = False
      indicator_id: UUID | None = None        # links answer → indicator
      choices: list[Choice] = []
      scale: dict | None = None               # {min,max,min_label,max_label}
      display_condition: str | None = None     # skip-logic expression
      validation: dict | None = None
  class Section(BaseModel): id: str; title: str; repeatable: bool = False; questions: list[Question]
  class FormSchema(BaseModel): sections: list[Section]; estimated_minutes: int | None = None
  ```
- Publishing freezes the schema and bumps `version` on subsequent edits (immutable published versions).
- Endpoints: `POST/GET/PATCH /forms`, `POST /forms/{id}/publish`.

**Testing**:
- `Unit: single_choice question with empty choices → ValidationError`
- `Unit: duplicate question id within form → ValidationError`
- `Unit: editing a published form → creates version+1, original frozen`
- `Integration: POST form draft → 201; publish → status=published, schema immutable`

#### 3.2 — Skip logic evaluator
**What**: Safe evaluator for `display_condition` expressions controlling question/section visibility.

**Design**:
- Grammar: `field_ref operator value` with `and`/`or`, e.g. `q_age >= 18 and q_region in ['NE','MW']`.
  Implemented with a small sandboxed AST evaluator (no `eval`) over a dict of prior answers.
  `services/skiplogic.py: evaluate(condition:str, answers:dict)->bool`.
- Used both server-side (validation) and exported to the frontend for live form behaviour.

**Testing**:
- `Unit: 'q_age >= 18' with {q_age:20} → True; {q_age:16} → False`
- `Unit: 'in' operator with list membership → correct`
- `Unit: malformed expression → ConditionParseError (never executes arbitrary code)`

#### 3.3 — Submissions and typed answers
**What**: Capture form submissions and per-question typed answers.

**Design**:
- `form_submissions(id, form_id fk, form_version, enrolment_id fk, submitted_by, submission_type
  check ('self_report','staff_administered','peer_assessment','supervisor_assessment','automated'),
  status check ('draft','submitted','validated','flagged','rejected'), collection_method check
  ('web','mobile_online','mobile_offline','paper_digitised','phone','api_import'), measurement_period,
  started_at, submitted_at, gps_lat numeric, gps_lng numeric, device_id, client_event_id uuid unique,
  sync_status, created_at)`.
- `form_answers(id, submission_id fk, question_id, value_text, value_numeric, value_date,
  value_boolean, value_choice_ids text[], raw_value, created_at)`. One typed column populated per
  answer type; `value_choice_ids` for multi-select.
- `client_event_id` is the offline idempotency key (Phase 6).
- Endpoint: `POST /forms/{id}/submissions` validates answers against the frozen form schema
  (required, type coercion, skip-logic consistency).

**Testing**:
- `Unit: numeric question receiving text answer → ValidationError`
- `Unit: required question omitted (and visible per skip logic) → ValidationError`
- `Unit: required question hidden by skip logic → omission allowed`
- `Integration: POST submission → 201, answers persisted with correct typed columns`
- `Integration: duplicate client_event_id → returns existing submission (idempotent), no dup row`

#### 3.4 — Outcome records derivation
**What**: Materialise `outcome_records` from indicator-linked answers, with correction history.

**Design**:
- `outcome_records(id, enrolment_id fk, indicator_id fk, measurement_period check ('baseline',
  'month_1','month_3','month_6','midpoint','month_12','endline','followup_3m','followup_6m',
  'followup_12m','custom'), measurement_date, value_numeric, value_text, value_categorical,
  form_submission_id fk, form_answer_id fk, data_source, is_validated, validated_by, superseded_by
  uuid null, created_at, updated_at)`. Composite index `(enrolment_id,indicator_id,measurement_period)`.
- On submission, `services/outcomes.derive_from_submission()` creates one outcome record per
  indicator-linked answer. A correction inserts a new row and sets `superseded_by` on the old row
  (append-only correction pattern — preserves the funder-audit trail without full event sourcing).
- Endpoints: `GET /enrolments/{id}/outcomes`, `POST /enrolments/{id}/outcomes` (manual entry),
  `PATCH /outcomes/{id}` (creates supersession).

**Testing**:
- `Unit: submission with two indicator-linked answers → two outcome_records`
- `Unit: correcting a value → new row, old row superseded_by set, both retained`
- `Integration: baseline + endline submissions → two outcome rows queryable by period`

---

## Phase 4: Outcome Analytics — Distance Travelled, Aggregation, Dashboards

### Purpose
Turn raw outcome records into the evidence story: per-participant Distance Travelled, programme- and
org-level roll-ups with demographic disaggregation, and dashboards. After this phase a programme
manager sees "X% of participants improved on indicator Y" with statistical rigour.

### Tasks

#### 4.1 — Distance Travelled calculation
**What**: Compute baseline→latest change per enrolment/indicator with reliable-change statistics.

**Design**:
- `distance_travelled_scores(id, enrolment_id, indicator_id, baseline_value, baseline_date,
  current_value, current_date, absolute_change, percentage_change, standardised_change,
  reliable_change_index, is_reliable_change, direction check ('improved','maintained','declined'),
  calculated_at, unique(enrolment_id,indicator_id))`.
- `services/distance_travelled.py`: pulls baseline + latest outcome for the pair; computes
  absolute/percentage change; standardised change (Cohen's d using cohort SD); Reliable Change Index
  `RCI = (x_post - x_pre) / SEdiff`. `direction` respects indicator `direction` (a decrease is
  "improved" for decrease-direction indicators).
- Recomputed by a Celery task on new outcome records; endpoint `GET /enrolments/{id}/distance-travelled`.

**Testing**:
- `Unit: baseline 40 endline 60 increase-direction → absolute_change=20, direction=improved`
- `Unit: decrease-direction indicator (e.g. anxiety) baseline 8 endline 4 → direction=improved`
- `Unit: RCI below 1.96 threshold → is_reliable_change=False`
- `Unit: no baseline present → score not computed, returns null gracefully`

#### 4.2 — Programme & org aggregation
**What**: Roll up individual outcomes to programme/cohort/org summaries with disaggregation.

**Design**:
- `programme_outcome_summaries(id, programme_id, indicator_id, cohort_id null, reporting_period_start,
  reporting_period_end, participant_count, measured_count, mean_baseline, mean_endline, mean_change,
  median_change, std_dev_change, improved_count, maintained_count, declined_count,
  improved_percentage, target_value, target_achieved, disaggregation jsonb, calculated_at, unique(...))`.
- `disaggregation` JSONB holds breakdowns by demographic key (e.g. by gender, age_group) computed via
  GIN-indexed `participants.demographics`.
- Celery task `recompute_summaries(programme_id)`; endpoint `GET /programmes/{id}/outcomes/summary?
  indicator=&cohort=&disaggregate_by=`.

**Testing**:
- `Unit: 10 enrolments, 7 improved → improved_percentage=70.00`
- `Unit: disaggregate_by=gender → summary buckets sum to measured_count`
- `Integration: summary endpoint returns mean_change matching hand-computed fixture`

#### 4.3 — Dashboard API and web dashboards
**What**: Aggregated dashboard endpoints and the programme dashboard UI.

**Design**:
- `GET /programmes/{id}/dashboard` returns enrolment counts by status, attendance rate, indicators
  tracked, baseline/endline coverage, overall improvement rate, active anomaly count. Cached in Redis
  (60s TTL), invalidated on relevant writes.
- Frontend `features/dashboards`: Recharts trend lines (mean per period), distance-travelled bar,
  cohort comparison; filters by cohort/date.

**Testing**:
- `Integration: dashboard endpoint returns counts matching seeded fixture`
- `Integration: cache hit on second call (assert single DB aggregation via spy)`
- `E2E (Playwright): load programme dashboard → charts render, filter by cohort updates values`

---

## Phase 5: Frameworks & Reporting — Alignment, Templates, Export

### Purpose
Make outcomes funder-ready: map internal indicators to IRIS+/SDG/CIDS, and generate branded
PDF/Word/Excel reports combining narrative and charts. After this phase an org produces a funder
submission directly from its data.

### Tasks

#### 5.1 — Framework catalogue and seed data
**What**: Load IRIS+, SDG, and IMP Five Dimensions reference frameworks.

**Design**:
- `frameworks(id, name, version, framework_type check ('iris_plus','sdg','funder_template',
  'government','custom','sroi','social_value'), source_url, is_system, created_at)`.
- `framework_metrics(id, framework_id fk, external_code, name, description, category,
  parent_metric_id, dimension jsonb, sort_order)`. `dimension` records IMP Five-Dimensions mapping
  (`what/who/how_much/contribution/risk`) per `standards.md`.
- Seed loader reads `backend/frameworks/*.json` (IRIS+ core metrics subset, SDG goals/targets, IMP).

**Testing**:
- `Unit: seed loader idempotent (re-run does not duplicate metrics)`
- `Integration: GET /frameworks → includes iris_plus, sdg, imp; metrics queryable by code`

#### 5.2 — Indicator → framework mapping
**What**: Map internal indicators to framework metrics.

**Design**:
- `indicator_framework_mappings(id, indicator_id fk, framework_metric_id fk, mapping_confidence check
  ('exact','approximate','partial','derived'), mapping_notes, mapped_by, unique(indicator_id,
  framework_metric_id))`.
- Endpoint `POST /indicators/{id}/mappings`. (AI-suggested mappings added in Phase 7.)

**Testing**:
- `Unit: duplicate mapping → 409`
- `Integration: map indicator to SDG-1.1 → GET /indicators/{id} includes mapping`

#### 5.3 — Report templates and rendering
**What**: Branded report generation to PDF/Word/Excel.

**Design**:
- `report_templates(id, organisation_id, name, report_type check ('funder_report','annual_impact',
  'programme_summary','outcome_dashboard','participant_progress','custom'), framework_id null,
  layout jsonb, is_system, timestamps)`. `layout` defines ordered blocks: `narrative`, `kpi_grid`,
  `outcome_chart`, `distance_travelled_table`, `framework_alignment_table`, `quote` (qualitative).
- `generated_reports(id, template_id, programme_id, generated_by, title, reporting_period_start,
  reporting_period_end, format check ('pdf','docx','xlsx','html'), file_url, status check
  ('generating','completed','failed','expired'), generated_at, expires_at)`.
- `reporting/render.py`: assembles a context from summaries + charts (matplotlib PNG for embed),
  renders Jinja2 HTML with org branding (logo/colours from `organisations.settings.branding`), then
  WeasyPrint→PDF, `python-docx`→Word, `openpyxl`→Excel. Runs as a Celery task; endpoint returns 202 +
  poll URL.

**Testing**:
- `Unit: render HTML context includes branding logo + period`
- `Integration (Celery eager): generate PDF report → status=completed, file non-empty, valid PDF header`
- `Integration: generate xlsx → openpyxl reopens, sheet has expected indicator rows`
- `E2E: request report → poll until completed → download link returns file`

#### 5.4 — Standards-aligned export: CIDS JSON-LD and OData v4
**What**: Interoperability exports per `standards.md`.

**Design**:
- `integrations/cids.py`: export an organisation/programme impact model as CIDS v3.2 JSON-LD
  (`@context` from commonapproach, ImpactModel→TheoryOfChange→Outcome→Indicator→IndicatorResult).
  Endpoint `GET /programmes/{id}/export/cids` → `application/ld+json`.
- `integrations/odata.py`: expose read-only OData v4 entity sets for `Outcomes`, `Participants`
  (pseudonymised), `Submissions` so external BI (Power BI/Tableau) can connect. `GET /odata/$metadata`.

**Testing**:
- `Unit: CIDS export validates against committed SHACL shapes (pyshacl) → conforms`
- `Integration: GET /odata/Outcomes?$top=5 → 5 rows, OData JSON envelope`
- `Integration: CIDS export of programme with ToC → contains hasOutcome links`

---

## Phase 6: Offline-First Mobile Data Collection

### Purpose
Enable field data collection in low-connectivity environments — a table-stakes capability matched by
KoboToolbox/ODK but absent from most impact-specific incumbents. After this phase a data collector
fills forms offline on a phone/tablet, and submissions sync automatically and idempotently when
connectivity returns.

### Tasks

#### 6.1 — PWA shell, offline form cache, IndexedDB store
**What**: Installable PWA that caches published forms and stores submissions locally.

**Design**:
- Service worker (Workbox) precaches app shell; runtime-caches `GET /forms/{id}` (published schema).
- Dexie.js stores: `forms` (cached schemas), `submissions` (with `client_event_id` UUID, `sync_status`,
  `local_timestamp`), `answers`. Form renderer consumes `FormSchema`, applies skip-logic client-side
  (shared evaluator ported to TS).
- Captures geolocation, signatures (canvas), photos (stored as Blobs).

**Testing**:
- `E2E (Playwright, offline mode): load form online, go offline, fill + save → stored in IndexedDB`
- `Unit (TS): skip-logic evaluator parity tests mirror backend cases`

#### 6.2 — Background sync and idempotent upload
**What**: Replay queued submissions to the API when online; conflict-free merge.

**Design**:
- Background Sync API (fallback: online-event listener) drains the queue, POSTing each submission with
  its `client_event_id`. Server dedups on `form_submissions.client_event_id` (unique) → safe retries.
- Because submissions are immutable facts, there are no update conflicts (adopting suggestion-2's
  offline insight); duplicate device captures are flagged for review, not rejected.
- `sync_status` lifecycle: `pending → syncing → synced | conflict`.

**Testing**:
- `Integration: POST same client_event_id twice → one row, second returns 200 existing`
- `E2E: fill 3 forms offline, restore connectivity → all 3 appear server-side, IndexedDB marked synced`
- `E2E: interrupt sync mid-batch, retry → no duplicates`

#### 6.3 — XLSForm / CSV import & KoboToolbox/ODK interop
**What**: Import survey instruments and existing data to avoid double entry.

**Design**:
- `integrations/xlsform.py`: parse XLSForm (`pyxform`) → internal `FormSchema`. `integrations/csv.py`:
  map CSV columns → participants/enrolments/outcomes via a mapping profile; tracked in `import_batches
  (id, organisation_id, integration_id null, source_type, source_filename, status check ('pending',
  'validating','importing','completed','completed_with_errors','failed','rolled_back'), total_records,
  imported_records, skipped_records, error_records, error_log jsonb, imported_by, timestamps)`.
- `integrations/odk.py`: pull submissions from ODK Central via `pyodk` (OData) into submissions.

**Testing**:
- `Unit: XLSForm with select_one → Question type=single_choice with choices`
- `Integration: CSV import of 100 rows, 3 malformed → status=completed_with_errors, error_records=3`
- `Integration (mocked ODK): pull → submissions created with collection_method=api_import`

---

## Phase 7: AI-Native Analysis — Qualitative Coding, Dedup, Anomalies, NLQ

### Purpose
Deliver the differentiating AI layer that no incumbent matches at scale: LLM qualitative analysis,
probabilistic participant matching, outcome anomaly detection, and natural-language querying. After
this phase the platform turns unread narrative evidence into structured insight and surfaces issues
proactively.

### Tasks

#### 7.1 — LLM client abstraction with prompt caching
**What**: Provider-agnostic LLM access with caching and cost tracking.

**Design**:
- `ai/llm_client.py`: `LLMClient.complete(system, messages, response_schema=None)` over Anthropic,
  OpenAI, and local (Ollama). Structured output via JSON-schema-constrained responses. Prompt caching
  on stable system prompts. `llm_usage(id, organisation_id, feature, model, input_tokens,
  output_tokens, cost_estimate, created_at)` for budgeting.

**Testing**:
- `Unit (mocked): complete returns parsed structured object matching response_schema`
- `Unit: provider switch via config → correct client instantiated`
- `Unit: usage row recorded with token counts`

#### 7.2 — Qualitative evidence storage and AI coding
**What**: Store narrative evidence and apply LLM thematic coding + sentiment + clustering.

**Design**:
- `qualitative_evidence(id, organisation_id, enrolment_id null, programme_id null, evidence_type check
  ('case_note','interview_transcript','open_ended_response','focus_group','observation',
  'story_of_change','other'), content text, content_tsv tsvector generated, embedding vector(384),
  word_count, language, form_submission_id null, recorded_date, created_at)`; GIN on tsv, ivfflat on
  embedding.
- `thematic_codes(id, organisation_id, code, label, description, parent_code_id, source check
  ('manual','ai_generated','framework_derived'), unique(organisation_id,code))`;
  `qualitative_evidence_codes(id, evidence_id fk, code_id fk, confidence_score, assigned_by check
  ('human','ai'), excerpt, unique(evidence_id,code_id))`; `sentiment_analysis(id, evidence_id fk,
  overall_sentiment, sentiment_score, model_version, analysed_at)`.
- `ai/qualitative.py`: Celery tasks `code_evidence` (LLM assigns codes + excerpts + confidence),
  `analyse_sentiment`, `cluster_programme_evidence` (embeddings + HDBSCAN/k-means → theme labels via
  LLM summarisation). System prompt template included in module docstring.

**Testing**:
- `Unit (mocked LLM): code_evidence → codes persisted with confidence + excerpt`
- `Unit: clustering 50 fixture texts → ≥2 clusters, each with LLM-generated label`
- `Integration: tsvector + embedding both populated on insert`

#### 7.3 — Probabilistic participant matching / dedup
**What**: Flag likely duplicate participants across re-enrolments.

**Design**:
- `ai/dedup.py`: blocking on `dedup_hashes` (phonetic name via Metaphone, DOB hash, location), then
  scoring with `recordlinkage` (Jaro-Winkler on names) + cosine similarity of `dedup_embedding`.
  Pairs above threshold create `dedup_candidates(id, organisation_id, participant_a, participant_b,
  score, factors jsonb, status check ('pending','confirmed','rejected'), reviewed_by, created_at)`.
- Confirming a match sets one participant `status=merged, merged_into_id=...` and re-points enrolments;
  evidence chain preserved. PII embeddings computed locally to keep data on-prem.

**Testing**:
- `Unit: 'Jon Smith' vs 'John Smith' same DOB → score above threshold → candidate`
- `Unit: confirm merge → loser status=merged, enrolments re-pointed, no orphan outcomes`
- `Integration: dedup batch over fixture of 200 (10 known dupes) → ≥9 candidates surfaced`

#### 7.4 — Anomaly detection and NLQ
**What**: Time-series anomaly alerts on cohort metrics; natural-language query interface.

**Design**:
- `ai/anomaly.py`: scheduled Celery beat scan compares rolling cohort means to expected trajectory
  (STL/EWMA); deviations beyond z-threshold create `outcome_anomalies(id, programme_id, indicator_id,
  cohort_id null, anomaly_type, severity, description, detected_value, expected_value, deviation_score,
  detected_at, acknowledged_at, acknowledged_by, resolution_notes)`.
- `ai/nlq.py`: LLM translates a question into a constrained, parameterised query against a curated
  semantic view (allow-listed tables/columns only — no arbitrary SQL), returns data + chart spec.
  Endpoint `POST /nlq {question}` → `{answer, data, chart}`.

**Testing**:
- `Unit: injected downward shift in cohort series → anomaly type=unexpected_decline`
- `Unit (mocked LLM): NLQ 'improvement for ages 18-25' → query filtered by age_group, never raw SQL`
- `Integration: NLQ result data matches direct aggregation for same filter`

---

## Phase 8: Theory of Change, 360° Surveys, Adaptive Management

### Purpose
Add the framework-design and multi-perspective capabilities (Sopact/Makerble parity) and the
differentiating adaptive-management loop: surface whether ToC assumptions hold. After this phase orgs
design causal models visually, link them to indicators/forms, run 360° assessments, and track
assumption validity.

### Tasks

#### 8.1 — Theory of Change graph (JSONB) + builder UI
**What**: Versioned ToC graph document and visual editor.

**Design**:
- `theories_of_change(id, programme_id fk, version, name, is_current, graph jsonb, created_by,
  timestamps, unique(programme_id,version))`. `graph` = `{narrative, nodes:[{id,type in
  (input,activity,output,outcome,impact),title,description,position,style,indicators:[uuid],
  metadata}], edges:[{id,from,to,label,assumption,evidence_strength in (strong,moderate,weak,
  untested)}], assumptions:[{id,edge_id,description,status in (untested,supported,
  partially_supported,refuted),evidence_notes}]}` validated by Pydantic `TocGraph`.
- Editing a current ToC creates `version+1`; outcome records remain linked to the version under which
  collected. Logic-model templates seeded from `backend/frameworks/logic_models/*.json`.
- Frontend React Flow canvas reads/writes the `graph` document; endpoint `PUT /programmes/{id}/toc`.

**Testing**:
- `Unit: edge referencing missing node id → ValidationError`
- `Unit: saving edit to current ToC → new version, prior is_current=false`
- `E2E: add nodes/edges in builder, save, reload → graph persisted`

#### 8.2 — 360° feedback surveys
**What**: Multi-perspective assessment (self/peer/supervisor) feeding one outcome view.

**Design**:
- Reuses `form_definitions.form_type='feedback_360'` and `form_submissions.submission_type`
  (`self_report`/`peer_assessment`/`supervisor_assessment`). A `feedback_panels(id, enrolment_id,
  indicator_id, respondents jsonb)` row groups the perspectives; aggregation averages/contrasts them.
- Endpoint `GET /enrolments/{id}/360/{indicator_id}` returns per-perspective values + variance.

**Testing**:
- `Unit: panel with self=4, peer=3, supervisor=5 → mean=4, variance computed`
- `Integration: three submissions of differing types → grouped into one panel view`

#### 8.3 — Adaptive management: assumption tracking
**What**: Surface whether ToC edge assumptions are supported by collected evidence.

**Design**:
- `services/adaptive.py`: for each ToC edge linking outcome nodes, compare expected vs observed
  outcome movement; update assumption `status` and `evidence_notes`; emit a digest. Endpoint
  `GET /programmes/{id}/toc/assumptions` returns assumptions with current support status.

**Testing**:
- `Unit: outcome node with no improvement where ToC predicts improvement → assumption status=refuted`
- `Integration: recompute assumptions after endline data → statuses updated, audit row written`

---

## Phase 9: Privacy, Consent & DSAR Compliance (ISO 27701 / GDPR)

### Purpose
Harden the privacy posture required to serve regulated, government-funded, and EU programmes:
consent lifecycle, Data Subject Access Requests, retention, and crypto-erasure. After this phase the
platform is defensibly GDPR/ISO 27701-aligned.

### Tasks

#### 9.1 — Consent management
**What**: Record, withdraw, and enforce participant consents.

**Design**:
- `participant_consents(id, participant_id fk, consent_type, granted, details jsonb (method,
  document_url, legal_basis, parent_guardian, witness), granted_at, withdrawn_at, expires_at,
  recorded_by, created_at)`. Service gate: data-sharing/export operations check active consent of the
  relevant type; withdrawal is enforced immediately and audited (`action=consent_change`).

**Testing**:
- `Unit: export requiring data_sharing_anonymised with withdrawn consent → blocked`
- `Integration: grant then withdraw → audit rows; subsequent export denied`

#### 9.2 — DSAR workflow and crypto-erasure
**What**: Handle access/rectification/erasure/portability/restriction requests within 30 days.

**Design**:
- `data_access_requests(id, organisation_id, participant_id null, request_type check ('access',
  'rectification','erasure','portability','restriction'), status check ('received','in_progress',
  'completed','denied'), requested_at, due_by (=requested_at+30d), completed_at, handled_by, notes,
  created_at)`.
- `access`/`portability` → generate a participant data bundle (decrypted PII + records) as JSON/PDF.
- `erasure` → crypto-shred: destroy the participant's wrapped data key so `pii_encrypted` is
  unrecoverable; null demographics; retain pseudonymised outcome rows for aggregate integrity
  (suggestion-2's crypto-shredding insight applied to the relational model).

**Testing**:
- `Unit: due_by = requested_at + 30 days`
- `Integration: erasure request completed → decrypt_pii raises (key destroyed), outcome_records remain`
- `Integration: access request → bundle contains all linked submissions/outcomes for participant`

#### 9.3 — Retention enforcement
**What**: Automatic retention per org policy.

**Design**:
- Celery beat job reads `organisations.settings.privacy.data_retention_months`; participants past
  retention with terminal status get PII crypto-shredded; action audited.

**Testing**:
- `Unit: participant exited beyond retention window → flagged for shredding`
- `Integration: retention job runs → PII shredded, audit recorded, aggregates intact`

---

## Phase 10: Integrations, Public API, Webhooks & SDK Surface

### Purpose
Open the platform to the ecosystem: documented OpenAPI 3.1 surface, webhooks, OAuth for partners, and
CRM connectors — eliminating double data entry, a root problem from `research.md`.

### Tasks

#### 10.1 — Public OpenAPI 3.1 surface and OAuth for partners
**What**: Stable, versioned, documented API with OAuth 2.0 client-credentials for machine clients.

**Design**:
- Mount versioned router `/api/v1`; serve `/openapi.json` (3.1) and Swagger/Redoc UI. OAuth 2.0
  client-credentials grant for partner apps (`oauth_clients` table); scopes map to RBAC.
- Generate frontend TS client from the spec in CI.

**Testing**:
- `Integration: GET /openapi.json → valid OAS 3.1 (validate with openapi-spec-validator)`
- `Integration: client-credentials token → access scoped endpoints; out-of-scope → 403`

#### 10.2 — Webhooks
**What**: Outbound webhooks on key events.

**Design**:
- `webhook_endpoints(id, organisation_id, url, secret, event_types text[], is_active, created_at)`;
  `webhook_deliveries(id, endpoint_id, event_type, payload jsonb, status, attempts, response_code,
  next_retry_at, created_at)`. Events: `submission.created`, `outcome.recorded`, `anomaly.detected`,
  `report.completed`, `dsar.received`. HMAC-SHA256 signature header; Celery retry with backoff.

**Testing**:
- `Unit: signature header = HMAC-SHA256(secret, body)`
- `Integration (mocked HTTP): outcome.recorded → delivery POSTed; 500 response → retry scheduled`

#### 10.3 — Salesforce / Apricot connectors
**What**: Two-way sync of participants/outcomes with major CRMs.

**Design**:
- `integrations/salesforce.py` (OAuth, maps NPC `IndicatorResult`/`Program` objects),
  `integrations/apricot.py` (API key). Sync runs as scheduled Celery jobs writing `import_batches`
  audit rows; field-mapping profiles stored in `integrations.config_encrypted`.
- `integrations(id, organisation_id, integration_type check ('salesforce','apricot','eto',
  'kobotoolbox','odk','csv_import','api','webhook'), name, config_encrypted bytea, is_active,
  last_sync_at, last_sync_status)`.

**Testing**:
- `Unit: SF outcome object → internal outcome_record mapping correct`
- `Integration (mocked SF API): sync pulls 2 records → 2 outcome_records, batch status=completed`

---

## Phase 11: Hardening, Observability & Release

### Purpose
Production-readiness: performance at scale, observability, security hardening, and a reproducible
release.

### Tasks

#### 11.1 — Performance & scale
**What**: Materialised views, partitioning, indexes, read-path caching.

**Design**:
- Materialised view for org-wide impact summary refreshed on schedule; declarative monthly
  partitioning on `audit_log`, `form_submissions`, `outcome_records`; verify GIN/ivfflat indexes;
  PgBouncer in compose. Load test with 100k participants / 1M outcome records (Locust).

**Testing**:
- `Perf: programme summary query p95 < 500ms at 100k participants (seeded)`
- `Perf: dashboard endpoint p95 < 200ms with cache warm`

#### 11.2 — Observability & rate limiting
**What**: Structured logs, metrics, tracing, abuse protection.

**Design**:
- JSON structured logging with request/correlation IDs (PROV correlation); Prometheus `/metrics`;
  OpenTelemetry traces; Redis-backed per-org/IP rate limiting returning RFC 7807 `429`.

**Testing**:
- `Integration: exceed rate limit → 429 problem+json with retry-after`
- `Integration: /metrics exposes request counters`

#### 11.3 — Security review & release packaging
**What**: Security pass and shippable artefacts.

**Design**:
- Dependency scan, secret scan, OWASP Top-10 checklist (esp. A01 access control via RLS tests, A03
  injection via parameterised/NLQ allow-list, A02 crypto for PII). `docker-compose.prod.yml`, backup
  job (pg_dump + WAL), seed/admin bootstrap CLI, versioned migrations smoke-tested on a prod-shaped DB.

**Testing**:
- `Integration: cross-tenant access attempts (every resource) → 404/403, asserted in a matrix test`
- `Integration: fresh docker-compose.prod up + migrate → seeded admin can log in, create programme E2E`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (auth, multi-tenancy, audit)  ─── required by everything
    │
Phase 2: Domain Core (programmes, participants, indicators) ─── requires 1
    │
Phase 3: Data Collection (forms, submissions, outcomes) ─── requires 2
    │
Phase 4: Outcome Analytics (distance travelled, aggregation, dashboards) ─── requires 3
    │
    ├── Phase 5: Frameworks & Reporting        ─── requires 4 (parallel with 6)
    ├── Phase 6: Offline Mobile Collection      ─── requires 3 (parallel with 5)
    └── Phase 7: AI-Native Analysis             ─── requires 3 (qual) + 4 (anomalies/NLQ)
         │
    Phase 8: ToC / 360° / Adaptive Mgmt         ─── requires 2,3 (adaptive needs 4); parallel with 5–7
    Phase 9: Privacy / Consent / DSAR           ─── requires 2 (can start early; parallel with 5–8)
         │
Phase 10: Integrations / Public API / Webhooks  ─── requires 3,4 (CRM sync benefits from 7 dedup)
    │
Phase 11: Hardening & Release                   ─── requires all
```

Parallelism: once Phase 4 lands, Phases 5, 6, 7 proceed concurrently. Phase 9 (privacy) and Phase 8
(ToC/adaptive) can run alongside them. Phase 10 depends on 3–4 and benefits from 7. Phase 11 closes.

---

## Definition of Done (per phase)

1. All tasks implemented.
2. All unit and integration tests pass (`pytest`); frontend E2E (`playwright`) green where applicable.
3. Linting and formatting pass (`ruff`; `eslint`/`prettier`).
4. Type checking passes (`mypy` backend; `tsc` frontend).
5. Docker build succeeds; `docker-compose up` brings the stack healthy.
6. Feature works end-to-end against a real PostgreSQL (testcontainers or compose db).
7. New config options documented (env vars in README + `.env.example`).
8. New/changed API endpoints appear in the auto-generated OpenAPI 3.1 document and validate.
9. Alembic migration created, reversible, and applied in CI; RLS policies present on new tenant tables.
10. Mutating operations write an `audit_log` row; privacy-sensitive operations enforce consent and are audited.
```
