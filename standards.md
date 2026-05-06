# Standards & API Reference

> Project: SaaS Security Compliance Toolkit · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- Official URL: https://www.iso.org/standard/27001
- The internationally recognised ISMS standard. The 2022 revision reduced controls from 114 to 93, reorganised them into four categories (Organisational, People, Physical, Technological), and added 11 new controls covering threat intelligence, cloud security, and data masking. Transition from the 2013 version was required by October 2025. Any compliance toolkit must model these 93 Annex A controls and track implementation status against them.

**ISO/IEC 27002:2022 — Information Security Controls**
- Official URL: https://www.iso.org/standard/75652.html
- Companion guidance document to ISO 27001 providing implementation advice for each Annex A control. Compliance toolkits use 27002 to surface prescriptive guidance text alongside each control so users understand how to satisfy requirements, not just whether a control is implemented.

**ISO/IEC 27005:2022 — Information Security Risk Management**
- Official URL: https://www.iso.org/standard/80585.html
- Defines the information security risk assessment methodology required under ISO 27001 Clause 6.1. A toolkit must provide a risk register aligned with this standard, supporting risk identification, analysis, evaluation, treatment, and acceptance workflows.

**ISO/IEC 27017:2015 — Cloud Security Controls**
- Official URL: https://www.iso.org/standard/43757.html
- Extends ISO 27002 with cloud-specific controls covering shared responsibility, virtual machine hardening, and cloud service customer/provider obligations. Relevant for SaaS vendors needing to document cloud infrastructure controls for enterprise buyers.

**ISO/IEC 27018:2019 — Protection of PII in Public Clouds**
- Official URL: https://www.iso.org/standard/76559.html
- Code of practice for protecting personally identifiable information in public cloud environments. Increasingly cited by enterprise procurement as a baseline requirement alongside GDPR, making it a secondary certification target for SaaS vendors.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- Official URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational authorisation protocol used by all major compliance platforms (Vanta, Drata, Secureframe, Hyperproof) to authenticate API requests and control access to compliance data. Any integration API must implement OAuth 2.0 client credentials flow for machine-to-machine access and authorisation code flow for user-facing integrations.

**RFC 7519 — JSON Web Token (JWT)**
- Official URL: https://datatracker.ietf.org/doc/html/rfc7519
- Standard for representing claims securely between parties. Used alongside OAuth 2.0 in compliance platform APIs to encode token payloads, expiry, and scopes. Evidence collection API clients must handle JWT validation correctly to avoid privilege escalation.

**RFC 8414 — OAuth 2.0 Authorization Server Metadata**
- Official URL: https://datatracker.ietf.org/doc/html/rfc8414
- Defines the discovery endpoint (`.well-known/oauth-authorization-server`) enabling API clients to auto-discover token endpoint URLs and supported scopes. Simplifies integration onboarding for compliance platform API consumers.

**OpenID Connect Core 1.0**
- Official URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0, used to authenticate user identity during auditor portal logins and SSO integrations. Compliance platforms use OIDC to integrate with enterprise identity providers (Okta, Azure AD, Google Workspace) for automated user provisioning.

**RFC 7617 — The 'Basic' HTTP Authentication Scheme**
- Official URL: https://datatracker.ietf.org/doc/html/rfc7617
- Referenced for API key-based authentication patterns used by Drata's public API (Bearer token) and Secureframe (API key). While Basic auth is deprecated for most use cases, the token-in-header pattern it informs remains the fallback for simple integrations.

---

### Data Model & API Specifications

**OSCAL — Open Security Controls Assessment Language**
- Official URL: https://pages.nist.gov/OSCAL/ · GitHub: https://github.com/usnistgov/OSCAL
- NIST-led machine-readable format (XML, JSON, YAML) for representing security controls, system security plans, assessment results, and POA&Ms. OSCAL v1.x supports catalogs (control definitions), profiles (framework subsets), component definitions, system security plans, and assessment artefacts. A compliance toolkit that exports evidence packages and control implementations in OSCAL format enables interoperability with FedRAMP automation, auditor tooling, and CPRT. NIST published CSWP 53 in December 2025 charting OSCAL's roadmap including AI-driven compliance extensions.

**NIST SP 800-53 Rev 5 (v5.2.0) — Machine-Readable Control Catalog**
- Official URL: https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
- Download (OSCAL/JSON/XLSX): https://csrc.nist.gov/files/pubs/sp/800/53/r5/upd1/final/docs/sp800-53r5-control-catalog.xlsx
- The canonical US federal security controls catalog (1,000+ controls across 20 control families). Version 5.2.0 was released August 2025. A toolkit that maps SOC 2 Trust Services Criteria and ISO 27001 Annex A controls to SP 800-53 identifiers enables automated cross-framework gap analysis without manual spreadsheet maintenance.

**OpenAPI 3.1 Specification**
- Official URL: https://spec.openapis.org/oas/v3.1.0
- The standard format for describing REST APIs. All leading compliance platforms (Drata, Hyperproof, Secureframe) publish OpenAPI specs for their developer portals. A compliance toolkit must provide an OpenAPI 3.1 spec for its evidence collection and control management API so that integrators can auto-generate clients, use Postman/Insomnia collections, and leverage API gateways.

**GraphQL Specification (June 2018 / October 2021)**
- Official URL: https://spec.graphql.org/
- Sprinto's developer API uses GraphQL rather than REST, reflecting a trend among compliance platforms toward flexible query patterns that avoid over-fetching. A compliance toolkit targeting developer-friendly integrations should evaluate GraphQL for complex querying of controls, evidence chains, and risk assessments.

**JSON Schema (Draft 2020-12)**
- Official URL: https://json-schema.org/specification
- Used for validating evidence payloads, control metadata, and webhook event schemas. Compliance toolkits that accept structured evidence via API should publish JSON Schema definitions for all accepted payload types, enabling client-side validation before submission.

---

### Security & Authentication Standards

**NIST Cybersecurity Framework 2.0 (CSF 2.0)**
- Official URL: https://www.nist.gov/cyberframework
- Full PDF: https://nvlpubs.nist.gov/nistpubs/CSWP/NIST.CSWP.29.pdf
- Published February 2024. CSF 2.0 adds a sixth core function (Govern) to the original five (Identify, Protect, Detect, Respond, Recover) and expands scope beyond critical infrastructure to all organisations. It is increasingly used by SaaS vendors as a risk management framework alongside SOC 2. A compliance toolkit should offer CSF 2.0 function and subcategory mapping alongside traditional compliance frameworks.

**AICPA Trust Services Criteria (TSC) 2017 with 2022 Revised Points of Focus**
- Official URL: https://www.aicpa-cima.com/resources/download/2017-trust-services-criteria-with-revised-points-of-focus-2022
- The authoritative control criteria for SOC 2 audits, covering five Trust Services Categories: Security (CC), Availability (A), Processing Integrity (PI), Confidentiality (C), and Privacy (P). Security (Common Criteria, CC1–CC9) is mandatory; the other four are optional. A compliance toolkit must model all common criteria and their associated points of focus to enable control mapping and gap assessments. The 2022 revision updated guidance around logical access and risk management controls.

**PCI DSS v4.0.1**
- Official URL: https://www.pcisecuritystandards.org/standards/
- Resource Hub: https://blog.pcisecuritystandards.org/pci-dss-v4-0-resource-hub
- The payment card data security standard, now at v4.0.1 (published June 2024). Full compliance became mandatory from March 31, 2024, with additional future-dated requirements becoming mandatory from April 1, 2025. A toolkit supporting multi-framework compliance must model PCI DSS 12 requirements and map them to overlapping SOC 2 and ISO 27001 controls.

**HIPAA Security Rule**
- Official URL: https://www.hhs.gov/hipaa/for-professionals/security/index.html
- Mandates administrative, physical, and technical safeguards for electronic protected health information (ePHI). The January 2025 HIPAA Security Rule NPRM proposed making MFA mandatory. A compliance toolkit must support HIPAA control families and Business Associate Agreement (BAA) workflows for SaaS products in the healthcare vertical.

**OWASP Application Security Verification Standard (ASVS) 4.0.3**
- Official URL: https://owasp.org/www-project-application-security-verification-standard/
- GitHub: https://github.com/OWASP/ASVS
- A three-level verification standard covering architecture, authentication, session management, access control, cryptography, error handling, data protection, and more. ASVS Level 1 covers OWASP Top 10; Level 2 targets most enterprise applications; Level 3 is for high-assurance applications. A compliance toolkit can use ASVS as a technical security checklist that feeds evidence into SOC 2 CC6 (logical and physical access) controls.

**CSA Cloud Controls Matrix (CCM) v4 and CAIQ v4**
- Official URL: https://cloudsecurityalliance.org/research/cloud-controls-matrix/
- The Cloud Security Alliance's CCM provides 197 security objectives across 17 domains, structured for cloud providers and SaaS vendors. The Consensus Assessments Initiative Questionnaire (CAIQ v4) is a yes/no self-assessment that vendors complete to publish to the CSA STAR registry. Organisations already SOC 2 or ISO 27001 compliant satisfy approximately 75–80% of CAIQ requirements, making CCM a natural cross-mapping target for a compliance toolkit.

**CMMC 2.0 — Cybersecurity Maturity Model Certification**
- Official URL: https://dodcio.defense.gov/CMMC/
- US Department of Defense supply-chain certification requirement, mapped to NIST SP 800-171 controls. Level 1 (Foundational, 17 practices) is self-assessed; Level 2 (Advanced, 110 practices) requires a third-party assessment. Growing relevance for SaaS vendors selling to DoD contractors makes CMMC a secondary framework target for compliance toolkits.

**GDPR (EU) 2016/679 and CCPA/CPRA**
- GDPR Official Text: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32016R0679
- CCPA: https://oag.ca.gov/privacy/ccpa
- Data protection regulations requiring data mapping, lawful basis documentation, consent management, breach notification, and subject rights workflows. A compliance toolkit must support GDPR Article 30 Records of Processing Activities (RoPA) and CCPA opt-out mechanisms as evidence artefacts linked to privacy controls.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- Official URL: https://modelcontextprotocol.io/
- An emerging open standard for connecting AI assistants to external data sources and tools. A compliance toolkit with MCP server support could expose controls, evidence gaps, and risk assessments as context to AI coding assistants and GRC copilots, enabling natural-language compliance queries. The compliance domain is an early candidate for MCP adoption given its structured data model.

---

## Similar Products — Developer Documentation & APIs

### Vanta
- **Description:** The market-leading compliance automation platform with 300+ native integrations, supporting SOC 2, ISO 27001, HIPAA, GDPR, PCI DSS, and CMMC. Used by 8,000+ companies globally.
- **API Documentation:** https://developer.vanta.com/docs/vanta-api-overview
- **Authentication:** https://developer.vanta.com/docs/api-access-setup
- **Build Integrations Guide:** https://developer.vanta.com/docs/build-integrations
- **Custom Resources:** https://developer.vanta.com/docs/vanta-custom-resource
- **Standards:** REST/JSON; OAuth 2.0 (client credentials and authorisation code flows); OpenAPI spec available
- **Authentication:** OAuth 2.0 — client ID/secret issued per application type
- **Rate Limits:** 5 req/min (auth endpoints), 20 req/min (integration endpoints), 50 req/min (management endpoints)
- **Key Capabilities:** Automate evidence collection, bulk operations, resource and asset querying, custom integration data ingestion

### Drata
- **Description:** Compliance automation platform known for its "map once, apply everywhere" cross-framework approach; raised $328M. Strong CI/CD and developer tool integrations.
- **API Documentation:** https://developers.drata.com/api-docs/
- **Developer Portal:** https://developers.drata.com/
- **Standards:** REST/JSON; Bearer token authentication (API key); OpenAPI/Swagger spec; Postman/Insomnia collections available
- **Authentication:** Bearer token (API key); OAuth 2.0 playground available
- **Developer Tools:** Sandbox environment, GraphQL playground, API Explorer
- **Access Tier:** Advanced plan and above required for API access
- **Key Capabilities:** Evidence management, control status, personnel monitoring, vendor risk, webhooks management

### Secureframe
- **Description:** Compliance platform optimised for non-technical GRC buyers; raised $79M. Covers SOC 2, ISO 27001, HIPAA, PCI DSS, GDPR, CCPA, FedRAMP.
- **API Documentation:** https://developer.secureframe.com/
- **Help Centre:** https://support.secureframe.com/hc/en-us/articles/38295501741459-Secureframe-Developer-Portal-API
- **Custom Integrations:** https://secureframe.com/features/api
- **Standards:** REST/JSON; API key authentication; OpenAPI spec available
- **Authentication:** API key (generated per organisation)
- **Key Capabilities:** Automate compliance workflows, fetch audit data, manage users, ingest custom evidence

### Hyperproof
- **Description:** Risk and compliance operations platform; provides Hypersync SDK for building custom evidence integrations. Strong audit workflow and cross-framework coverage.
- **Developer Portal:** https://developer.hyperproof.app/
- **API Overview:** https://developer.hyperproof.app/hyperproof-api
- **Hypersync SDK:** https://developer.hyperproof.app/hypersync-sdk
- **Controls API:** https://developer.hyperproof.app/hyperproof-api/controls/controls.openapi
- **Standards:** REST/JSON; OAuth 2.0 (authorisation code and client credentials flows); OpenAPI spec published per resource
- **Authentication:** OAuth 2.0 — authorisation code flow for multi-tenant apps, client credentials for M2M
- **Key Endpoints:**
  - `POST /v1/controls/` — create control
  - `POST /v1/controls/{controlId}/proof` — attach evidence
  - `GET /v1/controls/{controlId}` — retrieve control details
- **Key Capabilities:** Controls CRUD, proof/evidence upload and management, custom fields, labels, issues tracking, external contacts

### Sprinto
- **Description:** Cloud-native GRC and compliance automation platform targeting mid-market SaaS; 20+ frameworks; 100+ pre-built integrations; raised $20M.
- **Developer API Docs:** https://developer.sprinto.com/docs/sprinto-developer-api-documentation
- **API Reference:** https://docs.sprinto.com/api-references/sprinto-developer-api
- **Standards:** GraphQL; JSON responses; HTTPS required for all calls; API key authentication
- **Authentication:** API key (admin-generated); rate-limited per IP and API key
- **Key Capabilities:** Programmatic access to controls, evidence, checks, and user management; custom workflow automation; 100+ native connectors (AWS, Okta, Jira, Slack)

### Tugboat Logic (now OneTrust Certification Automation)
- **Description:** Acquired by OneTrust in 2021, now positioned as OneTrust GRC's certification automation module. Simplifies SOC 2, ISO 27001, and HIPAA readiness with policy templates and automated evidence collection.
- **API Tracker:** https://apitracker.io/a/tugboatlogic
- **OneTrust Developer Hub:** https://developer.onetrust.com/
- **Standards:** REST/JSON; OAuth 2.0
- **Authentication:** OAuth 2.0 via OneTrust identity platform
- **Key Capabilities:** Policy template management, evidence collection, control gap assessment, auditor collaboration; full OneTrust GRC platform integration post-acquisition

### Comp AI (Open Source)
- **Description:** Open-source compliance platform for SOC 2, ISO 27001, HIPAA, and GDPR; self-hostable with MIT-licensed codebase. Targeted at teams wanting full data sovereignty.
- **GitHub:** https://github.com/trycompai/comp (community-maintained)
- **Product Page:** https://www.comp.ai/
- **Standards:** REST/JSON; API key or session authentication depending on deployment
- **Authentication:** Session-based for self-hosted; API key for programmatic access
- **Key Capabilities:** Evidence collection automation, policy management, controls framework, auditor access portal; fully inspectable and forkable codebase

### RegScale (OSCAL-native GRC)
- **Description:** GRC platform built natively on OSCAL, enabling machine-readable compliance data exchange. Targets FedRAMP and DoD use cases where OSCAL is mandatory, with commercial extension to SOC 2 and ISO 27001.
- **API Documentation:** https://developer.regscale.com/
- **OSCAL Support Blog:** https://regscale.com/blog/regscale-support-owasp-asvs/
- **Standards:** REST/JSON; OSCAL XML/JSON/YAML native import/export; OpenAPI 3.1 spec
- **Authentication:** API key and OAuth 2.0
- **Key Capabilities:** OSCAL catalog/profile import, system security plan generation, automated FedRAMP package production, cross-framework control mapping, continuous monitoring feeds

---

## Notes

### Emerging and Evolving Areas

**AI Governance Frameworks**: Several new AI-specific compliance frameworks are emerging in 2025–2026. The EU AI Act entered enforcement phases in 2025, and CSA published the AI-CAIQ (AI Consensus Assessments Initiative Questionnaire) targeting AI system operators. NIST AI RMF 1.0 is increasingly cited alongside CSF 2.0. A forward-looking compliance toolkit should model AI governance controls as a first-class framework alongside SOC 2 and ISO 27001.

**DORA (Digital Operational Resilience Act)**: The EU's Digital Operational Resilience Act came into full effect in January 2025, mandating ICT risk management, incident reporting, digital operational resilience testing, and third-party risk oversight for financial entities operating in the EU. SaaS vendors serving European financial services customers must evidence DORA compliance, making it a material new framework for compliance toolkits to support.

**FedRAMP Automation**: FedRAMP is mandating OSCAL-formatted System Security Plans (SSPs) for new authorisation packages. This positions OSCAL as the de facto machine-readable standard for US government compliance artefacts, and compliance toolkits that generate valid OSCAL output will have a significant advantage in the federal and state/local government verticals.

**Continuous Compliance Direction**: SOC 2 is evolving toward continuous evidence feeds and real-time control monitoring rather than point-in-time sampling. Compliance platform APIs must expose webhook or event-stream mechanisms so that toolkits can push evidence on-change rather than collecting it in periodic batch jobs.

**Cross-Framework Gap Analysis Data Models**: No open standard currently exists for describing the mapping relationships between SOC 2, ISO 27001, NIST CSF, PCI DSS, and CMMC controls. This represents an open opportunity for a toolkit to define and publish an open control-mapping schema (extending OSCAL relationships) that the broader GRC community can adopt.
