# 447 – Impact Measurement Platform — Feature & Functionality Survey

> Candidate #447 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Sopact Sense | Commercial SaaS | Proprietary | https://www.sopact.com/ |
| Bonterra Apricot | Commercial SaaS | Proprietary | https://www.bonterratech.com/product/apricot |
| Makerble | Commercial SaaS | Proprietary | https://about.makerble.com/ |
| LiveImpact | Commercial SaaS | Proprietary | https://www.liveimpact.org/ |
| Salesforce Nonprofit Cloud | Commercial SaaS | Proprietary | https://www.salesforce.com/nonprofit/ |
| Efforts to Outcomes (ETO) | Commercial SaaS | Proprietary (Bonterra) | https://www.socialsolutions.com/ |
| KoboToolbox | Open Source | Open Source | https://www.kobotoolbox.org/ |
| ODK (Open Data Kit) | Open Source | Apache 2.0 | https://getodk.org/ |
| Metabase | Open Source | AGPL / Commercial | https://www.metabase.com/ |

## Feature Analysis by Solution

### Sopact Sense

**Core features**
- Unified data origin platform collecting baseline, programme activity, and endline data in single system
- Persistent participant IDs enabling longitudinal tracking and automatic pre/post matching
- Continuous AI-powered analysis of quantitative and qualitative data as it arrives
- Theory of change modelling and logic model frameworks
- Framework alignment (IRIS+, SDGs, funder-specific templates)
- Outcome aggregation to programme and organisation levels
- Funder-ready report generation
- Multi-document evidence analysis (PDFs, transcripts, case notes alongside survey scores)

**Differentiating features**
- AI-native architecture positioning framework as secondary to data flow (framework-agnostic approach)
- Qualitative data theming at scale: open-ended responses analysed in minutes vs. weeks of manual coding
- "Evidence continuity" framing: solves the linkage problem across baseline, participation, and endline
- Continuous intelligence model vs. annual reporting cycle

**UX patterns**
- Data-first design; frameworks applied retroactively
- Streaming data model; analysis runs against incoming data immediately
- Narrative + quantitative evidence unified in same analysis pipeline

**Integration points**
- API/webhook capabilities (not explicitly documented but mentioned for integrations)
- Export-ready reporting for funder submission

**Known gaps**
- Limited information on specific CRM integrations (Salesforce, Apricot, etc.)
- Pricing and scalability details sparse in public documentation
- Case management integration details unclear

**Licence / IP notes**
- Proprietary commercial SaaS; no IP concerns

---

### Bonterra Apricot

**Core features**
- Integrated case management with outcome tracking
- Visual, real-time outcome dashboards
- Customizable data collection forms and workflows
- Integrated case notes linked to problems, goals, and action steps
- Attendance registration and management reporting
- Customizable reporting and funder-facing reports
- Data Standards functionality for aligning metrics across programmes
- Que Data Studio: plain-language natural language querying for instant chart/dashboard generation
- Electronic signatures and document storage
- Client progress tracking integrated with case management

**Differentiating features**
- Que Data Studio: natural language queries ("What were outcomes for clients aged 18-25?") generate visualizations instantly
- Integrated case notes with outcome tracking (not separate systems)
- Data Standards module for cross-programme metric alignment

**UX patterns**
- Case manager-centric: outcome tracking embedded in service delivery workflows
- Natural language interface for non-technical reporting
- Progress-focused (case notes → goals → outcomes lineage)

**Integration points**
- Bonterra ecosystem (Apricot + other Bonterra tools)
- Que Data Studio integration with existing data
- Likely API/webhook integration (inherited from Bonterra platform)

**Known gaps**
- Limited standalone impact measurement (closely tied to case management)
- Theory of change or logic model tools not explicitly mentioned
- Qualitative data analysis capabilities underspecified
- Mobile offline data collection not documented

**Licence / IP notes**
- Proprietary commercial SaaS owned by Bonterra; no IP concerns

---

### Makerble

**Core features**
- Impact Measurement Framework management
- Theory of Change reporting dashboards
- Logic Model dashboards for real-time monitoring
- Outcome and indicator library creation; customizable KPIs
- Baseline, midline, endline survey automation with Distance Travelled calculation
- Qualitative and quantitative data collection through surveys and assessment tools
- 360° Feedback Surveys for multi-perspective assessment
- KPI tracking across Growth, Behaviour Change, Attitude Change, Health/Wellbeing
- KPI auto-aggregation to programme and organisation levels
- Map-based impact visualization (heatmaps and location pins)
- Flexible real-time and stakeholder/funder/investor reporting
- CSV import and open API (one-way and two-way integrations)

**Differentiating features**
- Automatic Distance Travelled calculation (statistical change measurement)
- 360° Feedback surveys (peer, supervisor, self assessment)
- Map-based impact visualization with heatmaps and individual story pins
- Template KPIs to reduce setup time
- Flexible aggregation model (tags to participants, roll-up to programme/organisation)

**UX patterns**
- Framework-driven design (Theory of Change → Logic Model → KPI tracking)
- Outcome-first: KPIs defined before data collection
- Participatory: multiple perspectives through 360° approach
- Geospatial: impact visualized on maps alongside narrative stories

**Integration points**
- CSV import capability
- Open API with one-way and two-way integration support
- Third-party database integration

**Known gaps**
- Limited discussion of baseline matching or automatic participant deduplication
- Qualitative data analysis beyond survey collection underspecified
- Offline mobile data collection not mentioned
- CRM/case management integration not documented

**Licence / IP notes**
- Proprietary commercial SaaS; no IP concerns

---

### LiveImpact

**Core features**
- Integrated case management (intake, service tracking, outcomes reporting)
- Smart forms for data collection with mobile readiness
- Real-time outcome dashboards and customizable reporting
- Report builder for demographics, data visualizations, charts, graphs
- AI reporting feature: natural language queries generate insights
- AI-powered decision support, chat, and smart search
- Client portal with self-service profile management, scheduling, and self-reported participation
- Donor and fundraising management alongside case management
- Volunteer intake, scheduling, and time tracking
- GDPR/privacy compliance features

**Differentiating features**
- Unified platform: case management + donor + volunteer management
- AI chat and decision support for staff
- Client self-service portal with progress tracking
- Integrated fundraising alongside impact reporting (recognising donors view volunteer/client data)

**UX patterns**
- All-in-one nonprofit operations: case, donor, volunteer, outcomes
- Staff and client co-design: staff track outcomes, clients self-report
- AI-augmented: chatbot for exploration, natural language reporting

**Integration points**
- Bonterra ecosystem
- Likely standard API/webhook (inherited from platform)

**Known gaps**
- Theory of change or logic model design tools not documented
- Qualitative data analysis beyond smart forms underspecified
- Offline mobile data collection not mentioned
- API documentation sparse

**Licence / IP notes**
- Proprietary commercial SaaS; no IP concerns

---

### Salesforce Nonprofit Cloud

**Core features**
- Program Management module for service tracking and participant enrolment
- Outcome Management objects: indicator definition, goal setting, impact tracking
- Dynamic Assessments for collecting outcome data from programme participants
- Standard data model for donors, constituents, and outcome records
- Outcome indicators with target-setting and progress tracking
- Integration with broader Salesforce ecosystem
- Customizable workflows and dashboards
- Data model supports programme and constituent relationships

**Differentiating features**
- Salesforce platform power: unlimited customisation via Apex, flows, formulas
- Enterprise-scale: handles complex multi-programme, multi-site operations
- Donor + outcome integration: same system for major gifts and impact tracking
- Outcome Management objects provide standard approach (vs. custom workarounds in NPSP)

**UX patterns**
- Platform-centric: requires Salesforce expertise to configure
- Data model-first: standard objects (Outcome, Indicator, Goal) provided
- Customisation-heavy: configurability via clicks or code

**Integration points**
- Native to Salesforce ecosystem
- Integrations with hundreds of third-party apps via AppExchange
- APIs for custom integrations
- Salesforce Marketing Cloud, Analytics Cloud available

**Known gaps**
- Requires substantial customisation; not plug-and-play for impact measurement
- Theory of change design tools not included
- Qualitative data analysis minimal
- Mobile offline data collection requires custom development
- AI/ML impact analysis not native (though Salesforce Einstein can be added)

**Licence / IP notes**
- Proprietary commercial SaaS; no IP concerns
- Often requires significant Salesforce consulting for impact measurement setup

---

### Efforts to Outcomes (ETO)

**Core features**
- Comprehensive case management for large-scale operations
- Programme outcome tracking
- Client data collection linked to case records
- Performance management and impact measurement
- Funder-ready reporting
- Multi-site, multi-programme management
- Advanced security protocols for sensitive client data
- Batch processing for large data collection efforts
- Compliance and audit trail support

**Differentiating features**
- Enterprise focus: designed for large organisations (Harlem Children's Zone, NYC, Boston, HUD, etc.)
- Maturity: 25+ year legacy as the standard for large human-services nonprofits
- Multi-partner collaboration features (especially for government agencies and city collaboratives)
- Batch processing for field data collection at scale

**UX patterns**
- Legacy system: powerful but requires training
- Government/institutional focus: compliance-first design
- Batch-oriented: designed for large-scale programmes with many participants

**Integration points**
- APIs for programme/outcome data
- Batch import/export capabilities
- Limited third-party integration documentation in public domain

**Known gaps**
- Modern UX not documented (legacy system)
- Mobile offline data collection underspecified
- Qualitative data analysis capabilities sparse
- Theory of change or logic model design not documented

**Licence / IP notes**
- Proprietary commercial SaaS (owned by Bonterra); no IP concerns

---

### KoboToolbox

**Core features**
- Open-source data collection platform for online and offline surveys
- Mobile data collection via KoboCollect app (Android) or web forms (Enketo)
- Offline capability: forms cached and data stored locally, syncs on reconnection
- Advanced form logic (skip logic, cascading selects, branching)
- Rich data types: text, numbers, selections, dates, media (photos, video), barcodes, signatures
- Map-based data collection with location capture
- Tabular and map-based monitoring dashboards
- Server data management with accept/reject workflows
- Role-based access control and audit logs
- Data export to Excel, Power BI, Python, R
- Longitudinal tracking via Entities feature (persistent records updated over time)
- Free for NGOs; paid tiers for larger organisations

**Differentiating features**
- Entities: persistent person/place/thing records that can be updated over time (valuable for longitudinal outcome tracking)
- Offline-first design: built for field environments with unreliable connectivity
- Cost-effective: free tier for nonprofits
- Vibrant open-source community

**UX patterns**
- Form-builder design for rapid survey creation
- Field-first: mobile app as primary interface for data collectors
- Cloud-plus-local: online editing, offline collection
- Community-driven: open-source ethos enables customisation

**Integration points**
- Open APIs for custom integrations
- Export to standard formats (Excel, CSV)
- Integration with Power BI, Python, R for analysis
- Webhook support for automation
- REST API for custom applications

**Known gaps**
- No built-in outcome aggregation or dashboard for impact measurement
- Theory of change or logic model tools not included
- Analysis layer minimal (exports data for analysis elsewhere)
- No case management functionality
- Qualitative data handling basic (text fields only; no advanced NLP)

**Licence / IP notes**
- Open source (based on ODK); free and modifiable
- Allows commercial use and modification
- Community support model

---

### ODK (Open Data Kit)

**Core features**
- Open-source mobile data collection on Android devices and web
- Online and offline form-based data collection
- Entities feature: persistent records for people, places, things; updatable over time
- Form designer with conditional logic, branching, and complex skip patterns
- Rich data types: text, numbers, selections, dates, multimedia (photos, audio, video), barcodes, GPS
- Server data synchronization with encryption and role-based access
- Accept/reject workflows for data quality
- Audit logs and access controls
- Data export and integrations with Power BI, Python, R
- Extensive use in public health, development, research contexts
- Used by WHO, CDC, USAID, Red Cross, Carter Center, Jane Goodall Institute

**Differentiating features**
- Entities: longitudinal tracking through updateable person/place records (superior to form-only systems for outcome measurement)
- End-to-end encryption for sensitive health/development data
- Established global community and ecosystem
- Production-ready for complex, multi-year programmes
- Offline-first architecture designed for low-resource settings

**UX patterns**
- Form-centric: primary interaction via structured questionnaires
- Field-focused: mobile app as primary platform
- Collaborative: server-based with role-based permissions
- Enterprise-capable: handles millions of submissions securely

**Integration points**
- REST API for custom integrations
- Data export to Excel, CSV
- Ecosystem integrations with analytics tools (Power BI, Python, R, etc.)
- No native dashboard or BI tool included

**Known gaps**
- No built-in outcome measurement, aggregation, or analysis features
- Theory of change or logic model tools absent
- Dashboard/visualisation minimal (exports data for analysis elsewhere)
- Requires external BI tool for impact reporting
- No case management functionality
- Qualitative analysis minimal

**Licence / IP notes**
- Apache 2.0 open-source license
- Permissive; allows commercial use and modification
- Active community and professional support available
- No proprietary restrictions

---

### Metabase

**Core features**
- Open-source business intelligence and data visualisation
- No-code query builder for non-technical users
- Interactive dashboards with real-time updates
- Automated dashboard sharing via email or Slack subscriptions
- Connections to SQL databases, Google Analytics, Salesforce, and other data sources
- Data transformation and aggregation (semantic layer)
- Canonical metrics definition in Data Studio
- MetaBot AI: conversational query generation ("What was our outcome improvement?")
- Scheduled dashboard updates and alerts
- Data export and embedding capabilities

**Differentiating features**
- MetaBot AI: natural language interface to quantitative data
- No-code SQL alternative: visual query builder for non-technical users
- Semantic layer: curated metrics and data transformation
- Lightweight: easy to self-host vs. heavy BI platforms
- Free tier: AGPL open-source version available

**UX patterns**
- Data-driven: designed for self-serve analytics exploration
- Democratic: non-technical staff can create dashboards and queries
- Real-time: dashboards auto-refresh on schedule
- Notification-driven: teams subscribe to dashboard changes

**Integration points**
- Connections to 30+ databases (PostgreSQL, MySQL, SQLite, Oracle, Snowflake, etc.)
- Google Analytics, Salesforce, Slack integrations
- REST API for custom integrations
- Embedded dashboard API for third-party apps

**Known gaps**
- No case management or survey tools (analysis-only layer)
- Requires external data collection system (e.g., KoboToolbox, ODK) to feed data
- Not designed for field data collection
- Theory of change or logic model tools absent
- Qualitative data analysis minimal (quantitative focus)

**Licence / IP notes**
- Open source AGPL license for core product
- Commercial edition available with proprietary features
- AGPL: if extended and distributed, modifications must be shared back
- Suitable for nonprofits under AGPL (source must remain accessible)

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

These capabilities are present in nearly every solution and are essential for impact measurement:

- Data collection from programme participants (surveys, forms, assessments)
- Participant identifier/tracking across time (baseline → mid-point → endline)
- Outcome indicator definition and measurement
- Data aggregation to programme and organisation levels
- Basic reporting/dashboards showing outcomes
- Export capability for external funder reporting
- Role-based access control and user management
- Data encryption and privacy protection

### Differentiating Features

Capabilities present in some solutions that provide competitive advantage:

- **AI-powered qualitative analysis** (Sopact) – Open-ended responses automatically themed, clustered, and summarized without manual coding
- **Natural language query interface** (LiveImpact, Bonterra Que, Metabase MetaBot) – Stakeholders ask questions in plain English, system generates visualizations
- **Theory of Change and Logic Model builders** (Sopact, Makerble) – Visual frameworks linked to data collection and reporting, not separate artefacts
- **Longitudinal entity tracking** (ODK, KoboToolbox) – Persistent updateable person/place/thing records enabling multi-year outcome tracking without re-identification
- **360° feedback surveys** (Makerble) – Multi-perspective assessment combining self, peer, supervisor feedback
- **Distance Travelled calculation** (Makerble) – Automatic statistical change measurement
- **Offline-first mobile** (KoboToolbox, ODK) – Form-based data collection in low-connectivity environments with automatic sync
- **Unified nonprofit operations** (LiveImpact) – Case management + donor + volunteer outcomes in single system, recognizing holistic constituent journeys
- **Framework alignment tooling** (Sopact) – Automatic mapping of internal indicators to external frameworks (IRIS+, SDGs, funder templates)

### Underserved Areas / Opportunities

Gaps that multiple solutions share, representing genuine opportunities for differentiation:

- **Participant matching and deduplication** – While most systems support longitudinal tracking, intelligent matching of re-enrolling participants (same person, different spelling/date format) is manual or absent. Opportunity: probabilistic record linkage using name + date-of-birth + location embeddings to auto-flag likely duplicates.
- **Evidence continuity without double-entry** – Research.md cites this as the root problem. Few solutions truly eliminate the need to re-enter data across systems. Opportunity: unified data origin architecture (Sopact model) that collects evidence once and applies multiple frameworks retroactively.
- **Qualitative data at scale** – Most platforms support text fields but lack systematic analysis. Opportunity: NLP-powered clustering, sentiment analysis, and thematic coding of open-ended responses and case notes to surface patterns automatically.
- **Causal inference and impact attribution** – No platform addresses how to attribute outcome change to programme activities vs. external factors. Opportunity: Propensity score matching, diff-in-diff models, or randomised assignment tools for rigorous impact evaluation.
- **Real-time anomaly detection** – Outcome trends are typically reviewed quarterly. Opportunity: Automated alerts when cohort-level metrics deviate from expected trajectories (early warning for programme issues).
- **Personalised engagement triggers** – Systems track outcomes but don't automate follow-up. Opportunity: Conditional logic that flags participants at-risk of dropout or requiring acceleration, triggering automatic outreach (SMS, email, staff alert).
- **Theory of Change adaptive management** – Frameworks are static. Opportunity: Feedback loops that surface whether assumptions in the Theory of Change are holding true, prompting iteration.
- **Composable data models** – Most solutions impose a data structure. Opportunity: Flexible schema allowing organisations to define their own outcome hierarchies without software customisation.

### AI-Augmentation Candidates

Features currently implemented with manual/rule-based approaches where AI could measurably improve outcomes:

- **Qualitative coding and thematic analysis** – Currently manual; months of labour. AI: LLM-powered clustering of open-ended responses, entity extraction from case notes, sentiment analysis across narratives.
- **Participant matching/deduplication** – Currently manual or rule-based name matching. AI: probabilistic record linkage using embeddings (similar names, nearby locations, temporal proximity).
- **Anomaly and signal detection** – Currently missed. AI: time-series models to flag outcome trends that deviate from programme expectations; early warning for dropout risk.
- **Causal inference** – Currently not attempted. AI: propensity score matching, synthetic controls, or instrumental variables to estimate programme impact vs. selection bias.
- **Natural language reporting** – Currently template-driven. AI: LLMs to generate narrative impact reports combining quantitative outcomes with qualitative evidence, tailored to funder language.
- **Survey instrument improvement** – Currently static. AI: recommend skip logic adjustments and question reordering based on respondent engagement patterns.
- **Outcome prediction** – Currently none. AI: predict likely outcome trajectories based on baseline + early-intervention data; identify high-risk participants.
- **Evidence synthesis** – Currently manual. AI: cross-outcome pattern detection (e.g., "housing stability programmes show stronger employment outcomes when paired with job training").

---

## Legal & IP Summary

All commercial platforms reviewed are proprietary SaaS offerings with standard commercial licensing terms. No copyright, patent, or licensing conflicts were identified.

Open-source options: KoboToolbox is based on ODK (Apache 2.0, permissive). ODK itself is Apache 2.0. Metabase uses AGPL for the open-source edition, meaning if code is extended and redistributed, modifications must be shared back with the community. For nonprofits using Metabase purely for internal analysis (not redistributing modified code), AGPL poses minimal burden. If the new platform bundles Metabase-derived reporting features, independent legal review of AGPL compatibility is recommended.

No material was omitted due to IP uncertainty. All feature descriptions paraphrased from public documentation; no proprietary code or confidential design docs consulted.

---

## Recommended Feature Scope

Based on the above analysis, a new impact measurement platform should prioritise:

**Must-have (MVP)**
- Participant data collection via configurable surveys/forms with mobile support
- Persistent participant identifiers enabling baseline → mid-point → endline tracking
- Outcome indicator definition and measurement
- Data aggregation to programme and organisation levels
- Basic dashboards and reporting
- Funder-ready report export (PDF, Word, Excel)
- Role-based access control and data encryption
- Offline mobile data collection capability

**Should-have (v1.1)**
- Theory of Change and Logic Model visual builders
- Framework alignment (IRIS+, SDGs, funder-specific templates)
- AI-powered qualitative data analysis (open-ended response clustering and thematic coding)
- Natural language query interface for non-technical staff
- 360° feedback surveys for multi-perspective assessment
- Distance Travelled and statistical change calculations
- Longitudinal entity tracking (updateable person/place/thing records)
- Integration with case management systems (Apricot, LiveImpact, etc.) via APIs
- Batch import/export for large-scale programmes

**Nice-to-have (backlog)**
- Probabilistic participant matching/deduplication for re-enrolled participants
- Real-time anomaly detection and trend alerting
- Outcome impact prediction models
- Personalised engagement triggers for at-risk participants
- Evidence synthesis and cross-outcome pattern detection
- Automated narrative report generation combining quantitative + qualitative evidence
- Adaptive Theory of Change management with assumption tracking
- Composable data models for organisations to define outcome hierarchies without software customisation
- Native integration with fundraising platforms (donor outcome recognition)

---

## Sources

- [Sopact: AI-Native Stakeholder Intelligence Platform](https://www.sopact.com/)
- [Impact Measurement: The New Architecture for 2026 | Sopact](https://www.sopact.com/use-case/impact-measurement)
- [Impact Evaluation: Methods, Frameworks & AI Tools 2026 | Sopact](https://www.sopact.com/use-case/impact-evaluation)
- [Nonprofit Impact Measurement | Sopact](https://www.sopact.com/use-case/nonprofit-impact-measurement)
- [Bonterra Apricot | Case Management Software](https://www.bonterratech.com/product/apricot)
- [Bonterra Case Management Solutions](https://www.bonterratech.com/solutions/case-management-software)
- [Bonterra Apricot 2026: Benefits, Features & Pricing | Software Advice](https://www.softwareadvice.com/nonprofit/apricot-profile/)
- [Makerble | Impact Measurement and Outcome Tracking](https://about.makerble.com/)
- [Real-Time Impact Tracking Software | Makerble](https://discover.makerble.com/features/makerble-impact)
- [Makerble Features: Reporting, CRM, Data Collection](https://about.makerble.com/features-1)
- [LiveImpact | Case Management & Donor Management Software](https://www.liveimpact.org/)
- [LiveImpact Case Management for Nonprofits](https://www.liveimpact.org/program-case-management)
- [How to Effectively Measure and Share Your Nonprofit's Impact in 2026 | LiveImpact](https://www.liveimpact.org/blog/measure-share-nonprofit-impact)
- [Salesforce Nonprofit Cloud | Program Management](https://www.salesforce.com/nonprofit/program-management-software/)
- [Outcome Management with Nonprofit Cloud | Salesforce](https://help.salesforce.com/s/articleView?id=sfdo.npc_outcome_management_with_nonprofit_cloud.htm)
- [Salesforce Nonprofit Success Pack Complete Guide 2024 | A+ Plusify](https://aplusify.com/salesforce-nonprofit-success-pack-complete-guide/)
- [Efforts to Outcomes (ETO) Software | Social Solutions](https://www.socialsolutions.com/)
- [What is ETO? | Bonterra ETO Help Center](https://intercom.help/bonterra-eto/en/articles/11633366-what-is-eto)
- [KoboToolbox | Data Collection Platform](https://www.kobotoolbox.org/)
- [KoboToolbox: Collecting Data Offline](https://support.kobotoolbox.org/data-offline.html)
- [KoboToolbox Software Overview](https://www.kobotoolbox.org/about-us/software/)
- [Open Data Kit (ODK) | Better Evaluation](https://www.betterevaluation.org/tools-resources/open-data-kit-odk)
- [ODK | Collect data anywhere](https://getodk.org/)
- [Open Data Kit | United Nations Development Programme](https://www.undp.org/policy-centre/singapore/open-data-kit)
- [Metabase | Open Source Business Intelligence](https://www.metabase.com/)
- [Metabase GitHub Repository](https://github.com/metabase/metabase)
- [Metabase: Analytics Dashboards & Reporting](https://www.metabase.com/features/analytics-dashboards)
