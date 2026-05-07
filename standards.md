# Standards & API Reference

> Project: Impact Measurement Platform · Generated: 2026-05-07

## Industry Standards & Specifications

### Impact Measurement Frameworks

**IRIS+ (GIIN Impact Reporting and Investment Standards)**
- URL: https://iris.thegiin.org/
- Maintained by the Global Impact Investing Network (GIIN), IRIS+ is the de-facto taxonomy for impact metrics in the social and environmental investing sector. Version 5.3c (released December 2025) provides a structured catalogue of over 650 standardised metrics organised into Impact Categories, Themes, and Strategic Goals, with evidence-backed Core Metrics Sets for common programme types. Any platform targeting impact investors, foundations, or funded nonprofits should support IRIS+ metric alignment. The IRIS+ catalogue is downloadable and freely available for integration.

**Impact Management Project (IMP) — Five Dimensions of Impact**
- URL: https://impactfrontiers.org/norms/five-dimensions-of-impact/
- The IMP convened over 2,000 organisations to establish a common language for describing impact across five dimensions: What, Who, How Much, Contribution, and Risk. The Five Dimensions Norms define the data categories that any impact management system should be able to record and compare. IRIS+ is explicitly mapped to these five dimensions. Any data model or report schema for an impact measurement platform should be traceable to these five dimensions.

**Common Impact Data Standard (CIDS) v3.2**
- URL: https://www.commonapproach.org/common-impact-data-standard/ | Developer docs: https://www.commonapproach.org/developers/data-standard/ | GitHub: https://github.com/commonapproach/CIDS
- CIDS (latest stable: v3.2, July 2025) is an OWL ontology with JSON-LD as the interchange format. It provides a standardised machine-readable representation of an organisation's impact model (theory of change, logic model, outcome chain), the evidence collected, and the effects produced. The design goal is data portability: an impact data capsule exported from one CIDS-aligned tool can be imported into any other aligned tool with minimal human effort. CIDS defines three tiers of alignment (basic, essential, full). Validation uses SHACL (files available at https://github.com/commonapproach/CIDS/tree/main/validation). This is the most technically mature open data standard for impact measurement interoperability and is the primary candidate for an open data exchange format in a new platform.

**Social Value International — SROI Standard and Principles**
- URL: https://www.socialvalueint.org/standards-and-guidance
- Social Value International publishes the seven Principles of Social Value (involve stakeholders, understand what changes, value the things that matter, only include what is material, do not over-claim, be transparent, verify the result) and the Guide to SROI. These principles define the methodological floor for any credible impact report. The associated Standards for Applying the Principles of Social Value covers Impact Measurement and Management (IMM) system design. Particularly relevant for platforms targeting UK/European social enterprises, public service commissioners, and blended-finance investors.

**UN Sustainable Development Goals (SDG) Indicators**
- URL: https://unstats.un.org/sdgs/ | SDG DSD Guidelines: https://unstats.un.org/sdgs/files/SDG-DSD-Guidelines.pdf
- The UN Statistics Division maintains a Global SDG Indicators Data Platform with machine-readable access via an SDMX (Statistical Data and Metadata eXchange) API. SDG DSD v1.24 was released March 2026. SDMX is an ISO standard (ISO/IS 17369:2013) for statistical data and metadata exchange. An impact measurement platform targeting internationally active nonprofits, NGOs, or government funders will need to map collected indicators to SDG goals and targets and export data in SDG-compatible formats. The SDMX API provides programmatic access to official SDG indicator metadata.

### ISO Standards

**ISO 26000:2010 — Social Responsibility**
- URL: https://www.iso.org/iso-26000-social-responsibility.html
- Voluntary guidance standard applicable to all organisation types including nonprofits. Defines seven principles of social responsibility and seven core subjects. Relevant to any platform documenting organisational accountability and stakeholder engagement practices. Not a certification standard; provides a framework reference.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001.html
- The leading international standard for information security management. Defines requirements for establishing, implementing, and maintaining an ISMS. Directly relevant because impact measurement platforms hold sensitive personal data about programme participants (including health and social services beneficiaries). Achieving ISO 27001 alignment signals enterprise readiness to large funders and government agencies.

**ISO/IEC 27701:2025 — Privacy Information Management Systems**
- URL: https://www.iso.org/standard/27701
- As of October 2025, ISO/IEC 27701 was revised as a standalone standard (previously an extension of ISO 27001) for Privacy Information Management Systems (PIMS). It maps directly to GDPR obligations for both PII controllers and processors and now includes guidance for cloud services, AI-related processing, and health data. Highly relevant: impact platforms processing participant personal data (names, dates of birth, demographics, health outcomes) must implement privacy-by-design. The 2025 edition is the definitive technical reference for GDPR compliance architecture.

### W3C & IETF Standards

**W3C PROV-DM (Provenance Data Model)**
- URL: https://www.w3.org/TR/prov-dm/
- W3C PROV-DM defines a data model for representing provenance information on the Web, built around three core concepts: entities (data artefacts), activities (processes that created or used them), and agents (responsible parties). For an impact measurement platform, PROV-DM provides a standards-compliant way to record data lineage — which survey collected which baseline result, which staff member entered which assessment, and which outcome record was derived from which intervention. This is critical for audit trails required by government funders.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational IETF standard for delegated API authorization. Any impact measurement platform exposing APIs to third-party case management systems, BI tools, or donor systems must implement OAuth 2.0 for secure access delegation.

**OpenID Connect Core 1.0 (OIDC)**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Authentication layer built on OAuth 2.0; the standard for federated identity and single sign-on. Directly relevant for platforms that need to integrate with corporate identity providers (Microsoft Entra ID, Google Workspace) used by larger nonprofits and government agencies.

**RFC 7807 — Problem Details for HTTP APIs**
- URL: https://datatracker.ietf.org/doc/html/rfc7807
- Standard machine-readable error response format for RESTful HTTP APIs. Ensures consistent and parseable error messages across all API endpoints.

### Data Model & API Specifications

**OpenAPI Specification 3.1.1**
- URL: https://spec.openapis.org/oas/v3.1.1.html | GitHub: https://github.com/OAI/OpenAPI-Specification
- The de-facto standard for describing RESTful APIs in YAML or JSON. OAS 3.1 introduced full alignment with JSON Schema Draft 2020-12. All platform API endpoints should be described in an OpenAPI 3.1 document to enable SDK generation, automated testing, and partner integrations. Governed by the OpenAPI Initiative (Linux Foundation).

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/specification
- The standard for validating and documenting the structure of JSON data. Used to define impact data payload schemas, outcome indicator structures, and CIDS export formats. OAS 3.1 is a superset of JSON Schema 2020-12.

**XLSForm Specification**
- URL: https://xlsform.org/en/ | ODK XForms: http://getodk.github.io/xforms-spec/
- Open standard for authoring mobile and web survey forms in Excel format, converted to ODK XForms (an XML-based form standard). XLSForm is the portable lingua franca of the data collection ecosystem: forms authored in XLSForm run on ODK Central, KoboToolbox, CommCare, SurveyCTO, and ONA. An impact measurement platform that authors or imports survey instruments should support XLSForm import/export to interoperate with the existing NGO/nonprofit data collection ecosystem.

**OData v4 (Open Data Protocol)**
- URL: https://www.odata.org/ | OASIS Standard: https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=odata
- ISO/IEC-approved, OASIS-standardised protocol for building queryable RESTful APIs. Used natively by ODK Central for programmatic data access. Relevant for exposing programme and outcome datasets to external BI tools (Power BI, Tableau) without requiring custom connectors.

### Health Data Standards (for health-adjacent programmes)

**HL7 FHIR R4 — SDOH Clinical Care IG (Gravity Project)**
- URL: https://hl7.org/fhir/us/sdoh-clinicalcare/ | GitHub: https://github.com/HL7/fhir-sdoh-clinicalcare
- The Gravity Project FHIR Implementation Guide (IG) defines how to exchange Social Determinants of Health (SDOH) data — housing stability, food security, employment, education — using HL7 FHIR R4 REST APIs. For impact measurement platforms operating in health, housing, or social services sectors in the US, SDOH-aligned data models and FHIR-compatible outcome records enable direct interoperability with electronic health records (EHRs) and government reporting systems. Relevant for platforms serving community health workers, housing programmes, or workforce development.

---

## Similar Products — Developer Documentation & APIs

### ODK Central

- **Description:** Open-source mobile and web data collection platform used by WHO, CDC, USAID, Red Cross, and thousands of NGOs. Industry-standard for baseline and endline survey collection in low-connectivity environments.
- **API Documentation:** https://odkcentral.docs.apiary.io/ (interactive REST API reference)
- **SDKs/Libraries:** ruODK (R): https://docs.ropensci.org/ruODK/ | pyODK (Python): https://getodk.github.io/pyodk/
- **Developer Guide:** https://docs.getodk.org/
- **Standards:** REST/JSON, OData v4 for data access, XLSForm / ODK XForms for form definitions
- **Authentication:** API token authentication (Bearer token in Authorization header)
- **Notes:** ODK Central exposes three data access paths: OData service endpoints (queryable), RESTful submission downloads (ZIP archives), and REST API for form/project management. The Entities feature (introduced 2023, actively developed) enables persistent longitudinal records — a participant entity is created at baseline and updated at each subsequent contact, making ODK Central natively suitable for outcome tracking.

### KoboToolbox

- **Description:** Open-source data collection platform built on ODK, widely used by humanitarian and development organisations. Free tier for NGOs; paid tiers for larger organisations.
- **API Documentation:** https://support.kobotoolbox.org/api.html | API v2 Swagger: https://[server]/api/v2/docs/
- **SDKs/Libraries:** No official SDK; community Python and R wrappers available
- **Developer Guide:** https://support.kobotoolbox.org/
- **Standards:** REST/JSON, OpenAPI (schema downloadable in YAML or JSON from /api/v2/schema/), OData-compatible exports
- **Authentication:** API token (Authorization: Token header) or Basic Auth. V1 API deprecated January 2026; v2 API is current.
- **Notes:** REST Services feature allows webhook-style POST of submission data to external URLs in JSON or XML. Supports synchronous exports via API for pipeline integration. API token obtained from account settings or /token/?format=json endpoint.

### Salesforce Nonprofit Cloud — Outcome Management

- **Description:** Enterprise platform used by thousands of large nonprofits and foundations globally. Outcome Management objects provide a standard data model for impact strategy, indicator definition, goals, and results.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.nonprofit_cloud.meta/nonprofit_cloud/outcome_management_data_model.htm
- **SDKs/Libraries:** Salesforce SDKs (JavaScript, Python, Java, .NET): https://developer.salesforce.com/developer-centers/nonprofit-cloud
- **Developer Guide:** https://developer.salesforce.com/developer-centers/nonprofit-cloud (Developer Center)
- **Standards:** REST/JSON (Salesforce REST API), SOAP API, Bulk API 2.0, Streaming API (Bayeux/CometD), OpenAPI-compatible API descriptions
- **Authentication:** OAuth 2.0 (connected apps), SAML for SSO
- **Key Objects:** ImpactStrategy, ImpactStrategyAssignment, IndicatorDefinition, IndicatorResult, OutcomeActivity, Program. REST reference: https://developer.salesforce.com/docs/atlas.en-us.nonprofit_cloud.meta/nonprofit_cloud/rest-apis-case-management.htm

### Bonterra Impact Management (Apricot)

- **Description:** Case management and outcome tracking SaaS widely deployed across human-services nonprofits in the US. API access available to Enterprise and Pro tier customers.
- **API Documentation:** https://intercom.help/Bonterra-Apricot/en/articles/11475034-faqs-apricot-api-integration
- **SDKs/Libraries:** No official SDK; integration partners (Treadwell Data, Sidekick Solutions) provide Zapier and Power Automate connectors
- **Developer Guide:** https://help.bonterratech.com/everyaction/s/article/2979003-api-developer-documentation
- **Standards:** REST/JSON; Zapier integration available for no-code automation
- **Authentication:** API key (Enterprise/Pro tiers only)
- **Notes:** Integration capabilities include Bonterra Impact Management API, Connect API, and Automated Import via SFTP. Low-code platforms (Power Automate, Workato, Zapier) are the primary integration surface for most implementers.

### Makerble

- **Description:** UK-based impact measurement SaaS for charities, NGOs, and social enterprises. Supports theory of change frameworks, distance-travelled calculations, and multi-perspective surveys.
- **API Documentation:** Not publicly documented; described as an "Open API" with one-way and two-way integration support
- **SDKs/Libraries:** Not publicly available
- **Developer Guide:** https://about.makerble.com/features-1 (feature overview only)
- **Standards:** REST implied; CSV import/export; integrates with Mailchimp, Google Drive, Google Calendar
- **Authentication:** Not publicly documented
- **Notes:** Developer documentation is not publicly accessible; API capabilities are referenced in marketing materials only. Prospective integrators should contact Makerble directly for API credentials and technical documentation.

### Metabase

- **Description:** Open-source (AGPL) business intelligence and data visualisation platform. Used as the analytics layer in stacks where data is collected elsewhere (ODK, KoboToolbox) and reported via Metabase.
- **API Documentation:** https://www.metabase.com/docs/latest/api
- **SDKs/Libraries:** Metabase Embedding SDK (React): https://www.metabase.com/docs/latest/embedding/sdk/quickstart | GitHub: https://github.com/metabase/metabase
- **Developer Guide:** https://www.metabase.com/docs/latest/
- **Standards:** REST/JSON for the management API; OpenAPI schema available; embedded dashboards via JWT or API key; Agent API for AI query generation
- **Authentication:** API keys (development/local only); JWT SSO (production embedding, requires Pro/Enterprise licence); session-based authentication for interactive use
- **Notes:** Full app embedding and modular SDK allow Metabase dashboards and queries to be embedded in third-party applications. The Agent API (documented at /docs/latest/ai/agent-api) supports AI-driven natural language queries against connected databases.

### CIDS-Aligned Tools (Common Approach Ecosystem)

- **Description:** A growing number of impact measurement tools implement the Common Impact Data Standard (CIDS) to enable data portability between platforms. Software developers can align to CIDS to ensure their tool can exchange data with any other aligned tool.
- **API Documentation:** https://www.commonapproach.org/developers/data-standard/
- **SDKs/Libraries:** OWL ontology files + SHACL validation: https://github.com/commonapproach/CIDS
- **Developer Guide:** https://ontology.commonapproach.org/cids-en.html
- **Standards:** OWL ontology with JSON-LD interchange format; SHACL for data validation; semantic versioning
- **Authentication:** N/A (data exchange standard, not a hosted service)
- **Notes:** CIDS v3.2 (July 2025) is the current stable version. Implementing CIDS alignment enables interoperability with the growing ecosystem of tools in Canada, the UK, and internationally. This is the primary open standard to consider for the impact data exchange layer of a new platform.

---

## Notes

**Gaps and evolving areas:**

- No single standard has achieved universal adoption across the impact measurement ecosystem. IRIS+ dominates impact investing; CIDS is gaining traction in Canada and internationally; SROI/SVI is prominent in UK public services commissioning; SDG alignment is required for internationally funded programmes. A platform should support export to multiple frameworks rather than betting on one.

- FHIR SDOH is relevant specifically for platforms serving health-adjacent programmes (community health workers, housing and homelessness, food security). Implementing FHIR compatibility would enable integration with EHR systems and open the government/public health funder segment.

- Sopact Sense does not publish public API or developer documentation; its integration capabilities are referenced in marketing materials only. Any integration with Sopact would require a commercial relationship.

- The CIDS JSON-LD format and SHACL validation files are the most immediately actionable open technical standard for a new platform's data exchange layer. Aligning the platform's internal outcome data model to CIDS would enable interoperability with other CIDS-aligned tools without requiring bespoke connectors.

- MCP (Model Context Protocol) — not currently defined for impact measurement. A future opportunity is to build an MCP Server that exposes programme outcome data to AI assistants (Claude, Copilot, etc.), allowing staff to query impact data in natural language via their existing chat interfaces.
