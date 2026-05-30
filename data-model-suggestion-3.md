# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: SaaS Security Compliance Toolkit · Created: 2026-05-24

## Philosophy

Compliance frameworks differ wildly in structure: SOC 2 has 5 Trust Services Categories with ~60 criteria, ISO 27001 has 4 themes with 93 controls, NIST SP 800-53 has 20 families with 1000+ controls, PCI DSS has 12 requirements with 300+ sub-requirements, and HIPAA has 3 safeguard categories. A fully normalized model would need framework-specific columns or an EAV pattern to accommodate these differences. A JSONB hybrid places the invariant structure (every control has an ID, a code, and a status) in relational columns while framework-specific metadata, integration payloads, and assessment details live in JSONB columns.

This approach is particularly effective for compliance platforms because:
- **Control metadata varies by framework** — SOC 2 controls have "Trust Services Categories," ISO controls have "themes" and "attributes," NIST controls have "baselines" and "enhancements." JSONB accommodates all without schema changes.
- **Evidence payloads vary by source** — AWS Config snapshots, GitHub branch protection rules, Okta MFA policies, and HR system employee lists have completely different shapes. JSONB stores them natively.
- **Vendor questionnaire responses** — SIG Lite has 100+ questions, CAIQ has 300+. JSONB stores responses without a row-per-question explosion.

GIN indexes on JSONB columns enable containment queries (`@>`) for filtering controls by framework-specific attributes without sacrificing query performance.

**Best for:** Rapid development with multi-framework flexibility, platforms that must support new frameworks without schema migrations, and teams comfortable with PostgreSQL JSONB query patterns.

**Trade-offs:**
- **Pro:** New frameworks added without schema changes
- **Pro:** Evidence payloads stored natively regardless of source shape
- **Pro:** Fewer tables — faster development, simpler migrations
- **Pro:** GIN-indexed JSONB queries competitive with relational for containment
- **Con:** JSONB fields lack database-level constraints
- **Con:** Application must validate JSONB structure
- **Con:** Complex JSONB queries harder to optimize than relational JOINs
- **Con:** Reporting across JSONB fields requires careful extraction

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OSCAL (NIST) | Control `metadata` JSONB stores OSCAL properties; import/export via application layer |
| NIST SP 800-53 | Control families, baselines, enhancements stored in `metadata` JSONB |
| AICPA TSC | SOC 2 criteria, points of focus stored in control `metadata` |
| ISO 27001:2022 | Annex A themes, attributes, implementation guidance in `metadata` |
| PCI DSS v4.0.1 | Requirements, testing procedures, guidance in `metadata` |
| HIPAA | Safeguard types, addressable vs. required in `metadata` |

---

## Core Tables

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    plan TEXT NOT NULL DEFAULT 'free',
    settings JSONB NOT NULL DEFAULT '{}',
    -- {"default_review_frequency_days": 365, "evidence_retention_days": 730}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email TEXT NOT NULL,
    name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'member',
    permissions JSONB NOT NULL DEFAULT '{}',
    -- {"can_approve_policies": true, "can_manage_vendors": true, "frameworks": ["soc2", "iso27001"]}
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
```

---

## Frameworks & Controls

```sql
CREATE TABLE frameworks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    slug TEXT NOT NULL,
    -- 'soc2', 'iso27001', 'nist-800-53', 'pci-dss-4', 'hipaa'
    name TEXT NOT NULL,
    version TEXT NOT NULL,
    authority TEXT NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- SOC 2: {"trust_services_categories": ["CC", "A", "PI", "C", "P"]}
    -- ISO: {"annex_a_themes": ["organizational", "people", "physical", "technological"]}
    -- NIST: {"baselines": ["low", "moderate", "high"], "families": 20}
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug, version)
);

CREATE TABLE controls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id UUID NOT NULL REFERENCES frameworks(id) ON DELETE CASCADE,
    code TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    control_type TEXT NOT NULL DEFAULT 'detective',
    metadata JSONB NOT NULL DEFAULT '{}',
    -- SOC 2: {"category": "CC", "points_of_focus": ["logical_access", "system_operations"]}
    -- ISO: {"theme": "organizational", "attributes": ["preventive", "confidentiality"]}
    -- NIST: {"family": "AC", "baseline": ["low", "moderate", "high"], "enhancements": ["AC-2(1)", "AC-2(2)"]}
    -- PCI: {"requirement_group": 8, "testing_procedure": "8.3.1.a", "guidance": "..."}
    -- HIPAA: {"safeguard": "technical", "standard": "access_control", "required": true}
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
    confidence REAL NOT NULL DEFAULT 1.0,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_control_id, target_control_id)
);

CREATE TABLE control_status (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    control_id UUID NOT NULL REFERENCES controls(id),
    status TEXT NOT NULL DEFAULT 'not_started',
    owner_id UUID REFERENCES users(id),
    implementation JSONB NOT NULL DEFAULT '{}',
    -- {"notes": "...", "implemented_at": "2026-01-15", "test_frequency": "quarterly"}
    last_test JSONB NOT NULL DEFAULT '{}',
    -- {"result": "pass", "tested_at": "2026-05-01", "tested_by": "uuid", "details": "..."}
    evidence_ids UUID[] NOT NULL DEFAULT '{}',
    due_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, control_id)
);

CREATE INDEX idx_controls_framework ON controls(framework_id);
CREATE INDEX idx_controls_metadata ON controls USING GIN (metadata);
CREATE INDEX idx_control_status_tenant ON control_status(tenant_id, status);
CREATE INDEX idx_control_mappings_source ON control_mappings(source_control_id);
```

---

## Evidence Management

```sql
CREATE TABLE evidence (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    description TEXT,
    source TEXT NOT NULL DEFAULT 'manual',
    -- manual, aws, gcp, azure, github, gitlab, okta, google_workspace, jira, hr_system
    status TEXT NOT NULL DEFAULT 'pending_review',
    payload JSONB NOT NULL DEFAULT '{}',
    -- AWS: {"service": "iam", "resource_type": "policy", "resource_id": "arn:...", "config_snapshot": {...}}
    -- GitHub: {"repo": "org/repo", "branch_protection": {...}, "collected_via": "api"}
    -- Okta: {"policy_type": "mfa", "policy_config": {...}, "user_count": 150}
    -- Manual: {"file_url": "s3://...", "file_hash": "sha256:...", "file_type": "pdf"}
    control_ids UUID[] NOT NULL DEFAULT '{}',
    collected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ,
    reviewed_by UUID REFERENCES users(id),
    reviewed_at TIMESTAMPTZ,
    review_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_evidence_tenant ON evidence(tenant_id, status);
CREATE INDEX idx_evidence_source ON evidence(tenant_id, source);
CREATE INDEX idx_evidence_expiry ON evidence(expires_at) WHERE status = 'approved';
CREATE INDEX idx_evidence_payload ON evidence USING GIN (payload);
CREATE INDEX idx_evidence_controls ON evidence USING GIN (control_ids);
```

---

## Policies

```sql
CREATE TABLE policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    slug TEXT NOT NULL,
    category TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'draft',
    owner_id UUID REFERENCES users(id),
    current_version JSONB NOT NULL DEFAULT '{}',
    -- {"version_number": 3, "content_markdown": "...", "content_html": "...",
    --  "authored_by": "uuid", "approved_by": "uuid", "approved_at": "2026-05-01"}
    version_history JSONB NOT NULL DEFAULT '[]',
    -- [{"version_number": 1, "change_summary": "Initial draft", "published_at": "..."},
    --  {"version_number": 2, "change_summary": "Updated access control section", "published_at": "..."}]
    control_ids UUID[] NOT NULL DEFAULT '{}',
    review_frequency_days INT NOT NULL DEFAULT 365,
    next_review_date DATE,
    acknowledgement_stats JSONB NOT NULL DEFAULT '{}',
    -- {"total_employees": 50, "acknowledged": 45, "pct": 90.0, "last_sent_at": "2026-05-01"}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE TABLE policy_acknowledgements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID NOT NULL REFERENCES policies(id) ON DELETE CASCADE,
    version_number INT NOT NULL,
    user_id UUID NOT NULL REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address INET,
    UNIQUE (policy_id, version_number, user_id)
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
    criticality TEXT NOT NULL DEFAULT 'low',
    data_classification TEXT NOT NULL DEFAULT 'none',
    status TEXT NOT NULL DEFAULT 'active',
    owner_id UUID REFERENCES users(id),
    contract JSONB NOT NULL DEFAULT '{}',
    -- {"start_date": "2025-01-01", "end_date": "2026-12-31", "value_cents": 120000, "auto_renew": true}
    risk_profile JSONB NOT NULL DEFAULT '{}',
    -- {"overall_score": 3.2, "risk_level": "medium", "last_assessed_at": "2026-03-15",
    --  "open_findings": 2, "data_access": ["pii", "financial"], "subprocessors": ["aws", "stripe"]}
    questionnaire_responses JSONB NOT NULL DEFAULT '{}',
    -- {"sig_lite": {"completed_at": "2026-03-01", "score": 85,
    --   "responses": {"1.1": {"answer": "yes", "evidence": "..."}, "1.2": {...}}},
    --  "custom": {"responses": [...]}}
    control_ids UUID[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vendors_tenant ON vendors(tenant_id, status);
CREATE INDEX idx_vendors_risk ON vendors USING GIN (risk_profile);
CREATE INDEX idx_vendors_controls ON vendors USING GIN (control_ids);
```

---

## Risk Register

```sql
CREATE TABLE risks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    category TEXT NOT NULL,
    owner_id UUID REFERENCES users(id),
    status TEXT NOT NULL DEFAULT 'open',
    treatment TEXT NOT NULL DEFAULT 'mitigate',
    scoring JSONB NOT NULL DEFAULT '{}',
    -- {"inherent": {"likelihood": 4, "impact": 5, "score": 20},
    --  "residual": {"likelihood": 2, "impact": 5, "score": 10},
    --  "risk_appetite": 15}
    treatment_plan JSONB NOT NULL DEFAULT '{}',
    -- {"description": "Implement MFA and access reviews", "due_date": "2026-06-30",
    --  "milestones": [{"task": "Deploy MFA", "status": "done"}, {"task": "Quarterly access review", "status": "in_progress"}]}
    control_ids UUID[] NOT NULL DEFAULT '{}',
    review_history JSONB NOT NULL DEFAULT '[]',
    -- [{"reviewed_at": "2026-03-01", "reviewed_by": "uuid", "outcome": "accepted",
    --   "notes": "Risk accepted at current residual level"}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risks_tenant ON risks(tenant_id, status);
CREATE INDEX idx_risks_scoring ON risks USING GIN (scoring);
```

---

## Integrations & Automated Testing

```sql
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    provider TEXT NOT NULL,
    display_name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    credentials_vault_ref TEXT,
    config JSONB NOT NULL DEFAULT '{}',
    -- {"region": "us-east-1", "account_id": "123456789", "roles": ["SecurityAudit"]}
    sync_state JSONB NOT NULL DEFAULT '{}',
    -- {"last_sync_at": "2026-05-24T10:00:00Z", "records_synced": 150,
    --  "last_error": null, "next_sync_at": "2026-05-24T16:00:00Z"}
    tests JSONB NOT NULL DEFAULT '[]',
    -- [{"test_code": "aws.iam.mfa_enabled", "title": "Root MFA enabled",
    --    "control_id": "uuid", "schedule": "0 */6 * * *", "active": true},
    --  {"test_code": "aws.iam.password_policy", "title": "Password policy compliant", ...}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, provider)
);

CREATE TABLE test_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id UUID NOT NULL REFERENCES integrations(id) ON DELETE CASCADE,
    test_code TEXT NOT NULL,
    status TEXT NOT NULL,
    -- pass, fail, error, warning
    resource_id TEXT,
    resource_type TEXT,
    details JSONB NOT NULL DEFAULT '{}',
    -- {"expected": "mfa_enabled", "actual": "mfa_disabled", "resource_arn": "arn:aws:iam::root"}
    evidence_id UUID REFERENCES evidence(id),
    executed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_integrations_tenant ON integrations(tenant_id);
CREATE INDEX idx_test_results_integration ON test_results(integration_id, executed_at DESC);
CREATE INDEX idx_test_results_failing ON test_results(integration_id, test_code) WHERE status = 'fail';
```

---

## Remediation Tasks

```sql
CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    description TEXT,
    source_type TEXT NOT NULL,
    source_id UUID,
    control_id UUID REFERENCES controls(id),
    priority TEXT NOT NULL DEFAULT 'medium',
    status TEXT NOT NULL DEFAULT 'open',
    assignee_id UUID REFERENCES users(id),
    due_date DATE,
    comments JSONB NOT NULL DEFAULT '[]',
    -- [{"author_id": "uuid", "body": "Working on this", "created_at": "2026-05-20T..."}]
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tasks_tenant ON tasks(tenant_id, status);
CREATE INDEX idx_tasks_assignee ON tasks(assignee_id, status);
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
    auditor_firm TEXT,
    status TEXT NOT NULL DEFAULT 'planning',
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    requests JSONB NOT NULL DEFAULT '[]',
    -- [{"id": "uuid", "control_code": "CC6.1", "request_text": "Provide MFA configuration evidence",
    --   "status": "submitted", "assignee_id": "uuid", "due_date": "2026-06-15",
    --   "evidence_ids": ["uuid1", "uuid2"], "submitted_at": "2026-06-10"}]
    findings JSONB NOT NULL DEFAULT '[]',
    -- [{"id": "uuid", "control_code": "CC6.1", "severity": "minor",
    --   "finding_text": "MFA not enforced for service accounts",
    --   "management_response": "Will implement by Q3", "task_id": "uuid", "status": "remediated"}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audits_tenant ON audits(tenant_id, status);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    actor_id UUID,
    action TEXT NOT NULL,
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

## Example Queries

### Find all controls requiring MFA across frameworks

```sql
SELECT f.name AS framework, c.code, c.title
FROM controls c
JOIN frameworks f ON f.id = c.framework_id
WHERE f.tenant_id = 'tenant-uuid'
  AND c.metadata @> '{"points_of_focus": ["logical_access"]}'
ORDER BY f.name, c.code;
```

### Vendor risk dashboard

```sql
SELECT name, criticality,
       risk_profile->>'risk_level' AS risk_level,
       (risk_profile->>'overall_score')::REAL AS risk_score,
       (risk_profile->>'open_findings')::INT AS open_findings,
       contract->>'end_date' AS contract_end
FROM vendors
WHERE tenant_id = 'tenant-uuid'
  AND status = 'active'
ORDER BY (risk_profile->>'overall_score')::REAL DESC NULLS LAST;
```

### Evidence coverage gaps

```sql
SELECT c.code, c.title, cs.status,
       COALESCE(array_length(cs.evidence_ids, 1), 0) AS evidence_count
FROM control_status cs
JOIN controls c ON c.id = cs.control_id
JOIN frameworks f ON f.id = c.framework_id
WHERE cs.tenant_id = 'tenant-uuid'
  AND f.slug = 'soc2'
  AND COALESCE(array_length(cs.evidence_ids, 1), 0) = 0
  AND cs.status != 'not_applicable'
ORDER BY c.sort_order;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core | 2 | tenants, users |
| Frameworks & Controls | 4 | frameworks, controls, control_mappings, control_status |
| Evidence | 1 | evidence (payload varies by source in JSONB) |
| Policies | 2 | policies (versions in JSONB), policy_acknowledgements |
| Vendor Risk | 1 | vendors (assessments, questionnaires in JSONB) |
| Risk Management | 1 | risks (scoring, treatment plan, review history in JSONB) |
| Integrations | 2 | integrations (tests in JSONB), test_results |
| Remediation | 1 | tasks (comments in JSONB) |
| Audit | 1 | audits (requests, findings in JSONB) |
| Infrastructure | 1 | audit_log |
| **Total** | **16** | |

---

## Key Design Decisions

1. **Control metadata as JSONB** — The `metadata` column on `controls` stores framework-specific attributes (SOC 2 points of focus, ISO themes, NIST baselines). A GIN index enables queries like `metadata @> '{"family": "AC"}'` without framework-specific columns.

2. **Evidence payload as JSONB** — AWS config snapshots, GitHub webhook payloads, and manual file uploads have completely different structures. Storing them all in a `payload` JSONB column avoids a complex inheritance hierarchy. The `source` column indicates which shape to expect.

3. **Vendor questionnaire responses as JSONB** — SIG Lite (100+ questions), CAIQ (300+), and custom questionnaires have different structures. Storing the full response set in `questionnaire_responses` JSONB avoids a row-per-question table that would reach millions of rows across vendors.

4. **Audit requests and findings as JSONB arrays** — An audit typically has 20-50 evidence requests and 5-15 findings. Storing these as JSONB arrays on the `audits` table keeps audit collaboration self-contained. For platforms with hundreds of findings per audit, this should be normalized.

5. **Policy version history as JSONB** — `version_history` stores a summary array, while `current_version` stores the full content. This avoids a separate `policy_versions` table while preserving change history.

6. **Risk scoring as JSONB** — The `scoring` JSONB stores inherent/residual likelihood, impact, and score in a structured object. This keeps the risk register as a single table while supporting different scoring methodologies (5×5, 3×3, custom).

7. **UUID arrays for control references** — `control_ids` arrays on evidence, policies, vendors, and risks replace junction tables. GIN-indexed for containment queries. Trade-off: no foreign key enforcement, but dramatically simpler queries and fewer tables.

8. **Comments as JSONB arrays** — Task comments stored inline as a JSONB array. Appropriate for low-volume comments (typical for compliance tasks); should be normalized if comment volume is high.
