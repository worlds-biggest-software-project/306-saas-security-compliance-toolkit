# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: SaaS Security Compliance Toolkit · Created: 2026-05-24

## Philosophy

Compliance automation is fundamentally a mapping problem: controls from multiple frameworks (SOC 2, ISO 27001, HIPAA, PCI DSS, GDPR) must be mapped to evidence sources, policies, vendors, and remediation tasks. A normalized relational model makes these many-to-many relationships explicit through junction tables, enabling precise cross-framework queries like "which SOC 2 controls overlap with ISO 27001 Annex A?" or "which evidence items satisfy controls across multiple frameworks?"

The OSCAL (Open Security Controls Assessment Language) standard from NIST defines a machine-readable format for control catalogs, profiles, and assessment results. This model aligns with OSCAL's entity hierarchy: catalog → control → parameter → assessment → finding. Each OSCAL concept maps to a dedicated table, making import/export of OSCAL documents straightforward.

Compliance platforms like Vanta and Drata manage hundreds of integrations (AWS, GitHub, Okta, HR systems) that continuously collect evidence. This model separates the integration infrastructure (connections, sync schedules) from the evidence it produces, and explicitly maps evidence to the controls it satisfies. Auditors can trace any control to its evidence chain through normalized foreign keys.

**Best for:** Teams needing precise cross-framework control mapping, OSCAL import/export capability, and auditor-facing compliance reports with clear evidence-to-control traceability.

**Trade-offs:**
- **Pro:** Cross-framework control mappings are explicit and queryable
- **Pro:** OSCAL-aligned entity hierarchy enables standards-based import/export
- **Pro:** Evidence-to-control traceability is a simple JOIN path
- **Pro:** Auditor collaboration with structured findings and responses
- **Con:** 30+ tables — significant schema to maintain
- **Con:** Adding a new framework requires populating multiple tables
- **Con:** Control metadata that varies by framework requires careful normalization
- **Con:** Rigid schema makes rapid prototyping slower

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OSCAL (NIST) | Entity hierarchy mirrors OSCAL catalog/profile/assessment/result structure |
| NIST SP 800-53 | Control families modeled as `control_families` rows; 1000+ controls imported as `controls` |
| AICPA Trust Services Criteria | SOC 2 criteria (CC, A, PI, C, P) as framework controls |
| ISO 27001:2022 Annex A | 93 controls across 4 categories imported as controls |
| PCI DSS v4.0.1 | 12 requirement groups modeled as control families |
| HIPAA Security Rule | Administrative/Physical/Technical safeguards as control families |
| NIST CSF 2.0 | 6 functions (Govern, Identify, Protect, Detect, Respond, Recover) as top-level categories |
| GDPR | Data processing activities and DPIA requirements as controls |

---

## Core Organization

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    plan TEXT NOT NULL DEFAULT 'free',
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email TEXT NOT NULL,
    name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'member',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_members (
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member',
    PRIMARY KEY (team_id, user_id)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
```

---

## Frameworks & Controls

```sql
CREATE TABLE frameworks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    version TEXT NOT NULL,
    authority TEXT NOT NULL,
    -- 'NIST', 'AICPA', 'ISO', 'PCI SSC', 'HHS', 'EU'
    framework_type TEXT NOT NULL DEFAULT 'regulatory',
    -- regulatory, industry, internal
    description TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name, version)
);

CREATE TABLE control_families (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id UUID NOT NULL REFERENCES frameworks(id) ON DELETE CASCADE,
    code TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    sort_order INT NOT NULL DEFAULT 0,
    UNIQUE (framework_id, code)
);

CREATE TABLE controls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id UUID NOT NULL REFERENCES frameworks(id) ON DELETE CASCADE,
    family_id UUID REFERENCES control_families(id),
    code TEXT NOT NULL,
    -- e.g., 'CC6.1', 'A.8.1', 'AC-2', '164.312(a)(1)'
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    guidance TEXT,
    control_type TEXT NOT NULL DEFAULT 'detective',
    -- preventive, detective, corrective, deterrent
    implementation_level TEXT NOT NULL DEFAULT 'required',
    -- required, addressable, recommended
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (framework_id, code)
);

CREATE TABLE control_mappings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    source_control_id UUID NOT NULL REFERENCES controls(id),
    target_control_id UUID NOT NULL REFERENCES controls(id),
    mapping_type TEXT NOT NULL DEFAULT 'equivalent',
    -- equivalent, partial, related, superset, subset
    confidence REAL NOT NULL DEFAULT 1.0,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_control_id, target_control_id)
);

CREATE TABLE control_objectives (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    control_id UUID NOT NULL REFERENCES controls(id),
    status TEXT NOT NULL DEFAULT 'not_started',
    -- not_started, in_progress, implemented, not_applicable
    owner_id UUID REFERENCES users(id),
    implementation_notes TEXT,
    tested_at TIMESTAMPTZ,
    test_result TEXT,
    -- pass, fail, partial, not_tested
    due_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, control_id)
);

CREATE INDEX idx_controls_framework ON controls(framework_id);
CREATE INDEX idx_control_mappings_source ON control_mappings(source_control_id);
CREATE INDEX idx_control_mappings_target ON control_mappings(target_control_id);
CREATE INDEX idx_control_objectives_status ON control_objectives(tenant_id, status);
```

---

## Evidence Management

```sql
CREATE TABLE evidence_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    description TEXT,
    collection_method TEXT NOT NULL DEFAULT 'manual',
    -- manual, automated, integration
    retention_days INT NOT NULL DEFAULT 365,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE evidence (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    evidence_type_id UUID REFERENCES evidence_types(id),
    title TEXT NOT NULL,
    description TEXT,
    source TEXT NOT NULL DEFAULT 'manual',
    -- manual, aws, github, okta, jira, hr_system, etc.
    source_reference TEXT,
    file_url TEXT,
    file_hash TEXT,
    status TEXT NOT NULL DEFAULT 'pending_review',
    -- pending_review, approved, rejected, expired, superseded
    collected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ,
    reviewed_by UUID REFERENCES users(id),
    reviewed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE evidence_controls (
    evidence_id UUID NOT NULL REFERENCES evidence(id) ON DELETE CASCADE,
    control_id UUID NOT NULL REFERENCES controls(id),
    satisfaction_level TEXT NOT NULL DEFAULT 'full',
    -- full, partial
    notes TEXT,
    PRIMARY KEY (evidence_id, control_id)
);

CREATE TABLE evidence_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    evidence_id UUID NOT NULL REFERENCES evidence(id) ON DELETE CASCADE,
    version_number INT NOT NULL,
    file_url TEXT NOT NULL,
    file_hash TEXT NOT NULL,
    uploaded_by UUID NOT NULL REFERENCES users(id),
    change_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (evidence_id, version_number)
);

CREATE INDEX idx_evidence_tenant ON evidence(tenant_id, status);
CREATE INDEX idx_evidence_expiry ON evidence(expires_at) WHERE status = 'approved';
CREATE INDEX idx_evidence_controls_control ON evidence_controls(control_id);
```

---

## Policy Management

```sql
CREATE TABLE policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    slug TEXT NOT NULL,
    category TEXT NOT NULL,
    -- information_security, acceptable_use, data_retention, incident_response, etc.
    status TEXT NOT NULL DEFAULT 'draft',
    -- draft, in_review, approved, published, archived
    current_version_id UUID,
    owner_id UUID REFERENCES users(id),
    review_frequency_days INT NOT NULL DEFAULT 365,
    next_review_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE TABLE policy_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID NOT NULL REFERENCES policies(id) ON DELETE CASCADE,
    version_number INT NOT NULL,
    content_markdown TEXT NOT NULL,
    content_html TEXT NOT NULL,
    change_summary TEXT,
    authored_by UUID NOT NULL REFERENCES users(id),
    approved_by UUID REFERENCES users(id),
    approved_at TIMESTAMPTZ,
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (policy_id, version_number)
);

ALTER TABLE policies ADD CONSTRAINT fk_current_version
    FOREIGN KEY (current_version_id) REFERENCES policy_versions(id);

CREATE TABLE policy_acknowledgements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_version_id UUID NOT NULL REFERENCES policy_versions(id),
    user_id UUID NOT NULL REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address INET,
    UNIQUE (policy_version_id, user_id)
);

CREATE TABLE policy_controls (
    policy_id UUID NOT NULL REFERENCES policies(id) ON DELETE CASCADE,
    control_id UUID NOT NULL REFERENCES controls(id),
    PRIMARY KEY (policy_id, control_id)
);

CREATE INDEX idx_policies_tenant ON policies(tenant_id, status);
CREATE INDEX idx_policy_ack_user ON policy_acknowledgements(user_id);
```

---

## Vendor Risk Management

```sql
CREATE TABLE vendors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    website TEXT,
    category TEXT NOT NULL,
    -- cloud_infrastructure, saas, contractor, data_processor, etc.
    criticality TEXT NOT NULL DEFAULT 'low',
    -- critical, high, medium, low
    data_classification TEXT NOT NULL DEFAULT 'none',
    -- none, public, internal, confidential, restricted
    status TEXT NOT NULL DEFAULT 'active',
    -- active, under_review, suspended, offboarded
    contract_expires_at DATE,
    owner_id UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE vendor_assessments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vendor_id UUID NOT NULL REFERENCES vendors(id) ON DELETE CASCADE,
    assessment_type TEXT NOT NULL DEFAULT 'questionnaire',
    -- questionnaire, penetration_test, soc2_review, iso_audit, onsite
    overall_risk_score REAL,
    risk_level TEXT NOT NULL DEFAULT 'not_assessed',
    -- critical, high, medium, low, not_assessed
    findings_count INT NOT NULL DEFAULT 0,
    assessed_by UUID NOT NULL REFERENCES users(id),
    assessed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    next_assessment_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE vendor_questionnaire_responses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assessment_id UUID NOT NULL REFERENCES vendor_assessments(id) ON DELETE CASCADE,
    question_code TEXT NOT NULL,
    question_text TEXT NOT NULL,
    response TEXT,
    risk_flag BOOLEAN NOT NULL DEFAULT FALSE,
    notes TEXT,
    UNIQUE (assessment_id, question_code)
);

CREATE TABLE vendor_controls (
    vendor_id UUID NOT NULL REFERENCES vendors(id) ON DELETE CASCADE,
    control_id UUID NOT NULL REFERENCES controls(id),
    relevance TEXT NOT NULL DEFAULT 'direct',
    -- direct, indirect, shared_responsibility
    PRIMARY KEY (vendor_id, control_id)
);

CREATE INDEX idx_vendors_tenant ON vendors(tenant_id, status);
CREATE INDEX idx_vendor_assessments_vendor ON vendor_assessments(vendor_id);
```

---

## Risk Management

```sql
CREATE TABLE risk_register (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    category TEXT NOT NULL,
    -- operational, technical, compliance, strategic, financial
    likelihood INT NOT NULL CHECK (likelihood BETWEEN 1 AND 5),
    impact INT NOT NULL CHECK (impact BETWEEN 1 AND 5),
    inherent_risk_score INT GENERATED ALWAYS AS (likelihood * impact) STORED,
    residual_likelihood INT CHECK (residual_likelihood BETWEEN 1 AND 5),
    residual_impact INT CHECK (residual_impact BETWEEN 1 AND 5),
    residual_risk_score INT GENERATED ALWAYS AS (residual_likelihood * residual_impact) STORED,
    treatment TEXT NOT NULL DEFAULT 'mitigate',
    -- accept, mitigate, transfer, avoid
    treatment_plan TEXT,
    owner_id UUID REFERENCES users(id),
    status TEXT NOT NULL DEFAULT 'open',
    -- open, mitigating, accepted, closed
    review_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE risk_controls (
    risk_id UUID NOT NULL REFERENCES risk_register(id) ON DELETE CASCADE,
    control_id UUID NOT NULL REFERENCES controls(id),
    effectiveness TEXT NOT NULL DEFAULT 'partial',
    -- full, partial, planned
    PRIMARY KEY (risk_id, control_id)
);

CREATE INDEX idx_risk_register_tenant ON risk_register(tenant_id, status);
CREATE INDEX idx_risk_register_score ON risk_register(tenant_id, inherent_risk_score DESC);
```

---

## Integrations & Automated Evidence Collection

```sql
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    provider TEXT NOT NULL,
    -- aws, gcp, azure, github, gitlab, okta, google_workspace, jira, etc.
    display_name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    -- pending, connected, error, disabled
    credentials_vault_ref TEXT,
    config JSONB NOT NULL DEFAULT '{}',
    -- {"region": "us-east-1", "account_id": "123456789"}
    last_sync_at TIMESTAMPTZ,
    last_error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, provider)
);

CREATE TABLE integration_tests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id UUID NOT NULL REFERENCES integrations(id) ON DELETE CASCADE,
    test_code TEXT NOT NULL,
    -- e.g., 'aws.iam.mfa_enabled', 'github.branch_protection'
    title TEXT NOT NULL,
    description TEXT,
    control_id UUID REFERENCES controls(id),
    schedule_cron TEXT NOT NULL DEFAULT '0 */6 * * *',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (integration_id, test_code)
);

CREATE TABLE test_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    test_id UUID NOT NULL REFERENCES integration_tests(id) ON DELETE CASCADE,
    status TEXT NOT NULL,
    -- pass, fail, error, warning
    resource_id TEXT,
    resource_type TEXT,
    details JSONB NOT NULL DEFAULT '{}',
    evidence_id UUID REFERENCES evidence(id),
    executed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_integrations_tenant ON integrations(tenant_id);
CREATE INDEX idx_test_results_test ON test_results(test_id, executed_at DESC);
CREATE INDEX idx_test_results_status ON test_results(test_id, status) WHERE status = 'fail';
```

---

## Remediation & Tasks

```sql
CREATE TABLE remediation_tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    description TEXT,
    source_type TEXT NOT NULL,
    -- test_failure, audit_finding, risk_treatment, vendor_finding, manual
    source_id UUID,
    control_id UUID REFERENCES controls(id),
    priority TEXT NOT NULL DEFAULT 'medium',
    -- critical, high, medium, low
    status TEXT NOT NULL DEFAULT 'open',
    -- open, in_progress, blocked, resolved, wont_fix
    assignee_id UUID REFERENCES users(id),
    due_date DATE,
    resolved_at TIMESTAMPTZ,
    resolved_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE task_comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID NOT NULL REFERENCES remediation_tasks(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id),
    body TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_remediation_tenant ON remediation_tasks(tenant_id, status);
CREATE INDEX idx_remediation_assignee ON remediation_tasks(assignee_id, status);
```

---

## Audit Collaboration

```sql
CREATE TABLE audits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    framework_id UUID NOT NULL REFERENCES frameworks(id),
    title TEXT NOT NULL,
    audit_type TEXT NOT NULL DEFAULT 'type_2',
    -- type_1, type_2, recertification, internal
    auditor_firm TEXT,
    status TEXT NOT NULL DEFAULT 'planning',
    -- planning, fieldwork, review, completed
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_id UUID NOT NULL REFERENCES audits(id) ON DELETE CASCADE,
    control_id UUID REFERENCES controls(id),
    request_text TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    -- pending, in_progress, submitted, accepted, needs_revision
    assignee_id UUID REFERENCES users(id),
    due_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_request_evidence (
    request_id UUID NOT NULL REFERENCES audit_requests(id) ON DELETE CASCADE,
    evidence_id UUID NOT NULL REFERENCES evidence(id),
    submitted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (request_id, evidence_id)
);

CREATE TABLE audit_findings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_id UUID NOT NULL REFERENCES audits(id) ON DELETE CASCADE,
    control_id UUID REFERENCES controls(id),
    severity TEXT NOT NULL DEFAULT 'observation',
    -- critical, major, minor, observation
    finding_text TEXT NOT NULL,
    management_response TEXT,
    remediation_task_id UUID REFERENCES remediation_tasks(id),
    status TEXT NOT NULL DEFAULT 'open',
    -- open, remediated, accepted, closed
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audits_tenant ON audits(tenant_id);
CREATE INDEX idx_audit_requests_audit ON audit_requests(audit_id, status);
CREATE INDEX idx_audit_findings_audit ON audit_findings(audit_id);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    actor_id UUID,
    actor_type TEXT NOT NULL DEFAULT 'user',
    -- user, system, integration
    action TEXT NOT NULL,
    -- created, updated, deleted, approved, acknowledged, etc.
    resource_type TEXT NOT NULL,
    resource_id UUID NOT NULL,
    changes JSONB NOT NULL DEFAULT '{}',
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);
```

---

## Row-Level Security

```sql
ALTER TABLE evidence ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON evidence
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

ALTER TABLE controls ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON controls
    USING (framework_id IN (
        SELECT id FROM frameworks WHERE tenant_id = current_setting('app.current_tenant_id')::UUID
    ));

ALTER TABLE vendors ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON vendors
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

ALTER TABLE risk_register ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON risk_register
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organization | 4 | tenants, users, teams, team_members |
| Frameworks & Controls | 5 | frameworks, control_families, controls, control_mappings, control_objectives |
| Evidence | 4 | evidence_types, evidence, evidence_controls, evidence_versions |
| Policies | 4 | policies, policy_versions, policy_acknowledgements, policy_controls |
| Vendor Risk | 4 | vendors, vendor_assessments, vendor_questionnaire_responses, vendor_controls |
| Risk Management | 2 | risk_register, risk_controls |
| Integrations | 3 | integrations, integration_tests, test_results |
| Remediation | 2 | remediation_tasks, task_comments |
| Audit Collaboration | 4 | audits, audit_requests, audit_request_evidence, audit_findings |
| Infrastructure | 1 | audit_log |
| **Total** | **33** | |

---

## Key Design Decisions

1. **Cross-framework control mappings** — The `control_mappings` table with `mapping_type` (equivalent, partial, related, superset, subset) enables queries like "show me all ISO 27001 controls that map to this SOC 2 criterion." This is the core value proposition of multi-framework compliance platforms.

2. **Control objectives per tenant** — `control_objectives` separates the framework-defined control (shared) from the tenant's implementation status. The same `controls` row can be referenced by multiple tenants without duplicating framework data.

3. **Evidence-to-control junction** — `evidence_controls` enables one piece of evidence to satisfy multiple controls across frameworks, and one control to require multiple evidence items. The `satisfaction_level` field tracks partial vs. full evidence coverage.

4. **Policy version control** — `policy_versions` provides a complete history of policy changes with approval workflow. `policy_acknowledgements` tracks employee sign-off per version, critical for SOC 2 Type 2 audits.

5. **Computed risk scores** — `inherent_risk_score` and `residual_risk_score` are `GENERATED ALWAYS AS` columns, ensuring the 5×5 risk matrix calculation is always consistent and queryable.

6. **Integration test framework** — `integration_tests` maps automated checks (e.g., "AWS IAM MFA enabled") to controls, with `test_results` providing continuous evidence collection. Failed tests can auto-create remediation tasks.

7. **Audit collaboration workflow** — The `audits` → `audit_requests` → `audit_request_evidence` chain models the real-world audit process: auditor requests evidence for specific controls, internal team submits evidence, auditor accepts or requests revision.

8. **Partitioned audit log** — `audit_log` is partitioned by `created_at` for retention management. In compliance platforms, the audit log itself is auditable evidence.
