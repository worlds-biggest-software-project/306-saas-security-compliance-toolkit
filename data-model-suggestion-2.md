# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: SaaS Security Compliance Toolkit · Created: 2026-05-24

## Philosophy

Compliance is inherently an audit discipline — every action, approval, and state change must be traceable, timestamped, and attributable. An event-sourced model makes the audit trail the source of truth rather than an afterthought. When an auditor asks "who approved this policy, when, and what changed?", the answer is a direct event query, not a reconstructed log.

Every compliance-relevant action is recorded as an immutable event: evidence collected, control status changed, policy approved, risk assessed, vendor reviewed. The current state of any entity (a control's compliance status, a policy's approval state, a vendor's risk score) is a materialised read model derived by projecting these events forward. This means the system can answer both "what is the current state?" and "what was the state on December 15th?" with equal confidence.

For compliance platforms specifically, this architecture aligns with SOC 2 Type 2 audits, which examine not just controls at a point in time but the operating effectiveness of controls over a period. The event stream IS the evidence of operating effectiveness — it shows every test result, every remediation, every policy acknowledgement as it happened.

**Best for:** Organizations where audit trail completeness is the primary requirement, where SOC 2 Type 2 period-of-time evidence is needed, and where "compliance as code" event pipelines feed dashboards and AI analysis.

**Trade-offs:**
- **Pro:** Complete, immutable audit trail — every state change is an event
- **Pro:** Temporal queries natural — "what was compliant on date X?"
- **Pro:** SOC 2 Type 2 evidence is the event stream itself
- **Pro:** Event stream feeds AI-powered compliance gap analysis
- **Con:** Read model eventual consistency
- **Con:** Event versioning needed as compliance requirements evolve
- **Con:** Higher storage volume from append-only events
- **Con:** More complex application code for projections

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope follows CloudEvents spec (type, source, time, data) |
| OSCAL (NIST) | Assessment results modeled as event streams; findings are events |
| NIST SP 800-53 | Control assessment events reference SP 800-53 control identifiers |
| SOC 2 Type 2 | Period-of-time evidence derived from event stream time ranges |
| ISO 27001 | Statement of Applicability reconstructable from control status events |
| GDPR Art. 30 | Processing activity changes tracked as events for records of processing |

---

## Event Store

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    stream_type TEXT NOT NULL CHECK (stream_type IN (
        'control', 'evidence', 'policy', 'vendor',
        'risk', 'integration', 'audit', 'user'
    )),
    stream_id UUID NOT NULL,
    sequence_number BIGINT NOT NULL,
    event_type TEXT NOT NULL,
    ce_source TEXT NOT NULL DEFAULT '/compliance-toolkit',
    ce_specversion TEXT NOT NULL DEFAULT '1.0',
    event_data JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- {"actor_id": "uuid", "actor_type": "user", "ip_address": "..."}
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_events_stream ON event_store(stream_id, sequence_number);
CREATE INDEX idx_events_tenant ON event_store(tenant_id, occurred_at DESC);
CREATE INDEX idx_events_type ON event_store(event_type, occurred_at DESC);
```

### Event Type Registry

```sql
CREATE TABLE event_type_registry (
    event_type TEXT PRIMARY KEY,
    stream_type TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO event_type_registry (event_type, stream_type, description) VALUES
-- Control lifecycle
('control.status_changed',      'control',     'Control implementation status changed'),
('control.test_executed',       'control',     'Automated or manual test run against control'),
('control.test_passed',         'control',     'Control test passed'),
('control.test_failed',         'control',     'Control test failed'),
('control.owner_assigned',      'control',     'Control ownership assigned'),
('control.mapping_created',     'control',     'Cross-framework mapping established'),
-- Evidence lifecycle
('evidence.collected',          'evidence',    'Evidence collected manually or via integration'),
('evidence.approved',           'evidence',    'Evidence reviewed and approved'),
('evidence.rejected',           'evidence',    'Evidence reviewed and rejected'),
('evidence.expired',            'evidence',    'Evidence reached expiration date'),
('evidence.superseded',         'evidence',    'Evidence replaced by newer version'),
('evidence.linked_to_control',  'evidence',    'Evidence linked to a control'),
-- Policy lifecycle
('policy.drafted',              'policy',      'New policy version drafted'),
('policy.submitted_for_review', 'policy',      'Policy submitted for review'),
('policy.approved',             'policy',      'Policy version approved'),
('policy.published',            'policy',      'Policy published to employees'),
('policy.acknowledged',         'policy',      'Employee acknowledged policy'),
('policy.review_due',           'policy',      'Policy periodic review is due'),
-- Vendor lifecycle
('vendor.onboarded',            'vendor',      'New vendor added'),
('vendor.assessed',             'vendor',      'Vendor risk assessment completed'),
('vendor.risk_scored',          'vendor',      'Vendor risk score calculated'),
('vendor.contract_renewed',     'vendor',      'Vendor contract renewed'),
('vendor.offboarded',           'vendor',      'Vendor offboarded'),
('vendor.finding_raised',       'vendor',      'Security finding raised for vendor'),
-- Risk lifecycle
('risk.identified',             'risk',        'New risk identified'),
('risk.assessed',               'risk',        'Risk likelihood/impact assessed'),
('risk.treatment_decided',      'risk',        'Risk treatment decision made'),
('risk.mitigated',              'risk',        'Risk mitigation implemented'),
('risk.accepted',               'risk',        'Risk formally accepted'),
('risk.review_completed',       'risk',        'Periodic risk review completed'),
-- Integration lifecycle
('integration.connected',       'integration', 'Integration connected'),
('integration.sync_completed',  'integration', 'Integration sync completed'),
('integration.sync_failed',     'integration', 'Integration sync failed'),
('integration.disconnected',    'integration', 'Integration disconnected'),
-- Audit lifecycle
('audit.started',               'audit',       'Audit engagement started'),
('audit.evidence_requested',    'audit',       'Auditor requested evidence'),
('audit.evidence_submitted',    'audit',       'Evidence submitted to auditor'),
('audit.finding_raised',        'audit',       'Audit finding raised'),
('audit.finding_remediated',    'audit',       'Audit finding remediated'),
('audit.completed',             'audit',       'Audit completed'),
-- User lifecycle
('user.invited',                'user',        'User invited to tenant'),
('user.role_changed',           'user',        'User role changed'),
('user.deactivated',            'user',        'User deactivated');
```

---

## Snapshots & Projections

```sql
CREATE TABLE stream_snapshots (
    stream_id UUID NOT NULL,
    stream_type TEXT NOT NULL,
    sequence_number BIGINT NOT NULL,
    snapshot_data JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_number)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_sequence BIGINT NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status TEXT NOT NULL DEFAULT 'running'
);
```

---

## Materialised Read Models

### Compliance Posture

```sql
CREATE TABLE rm_control_status (
    tenant_id UUID NOT NULL,
    control_id UUID NOT NULL,
    framework_id UUID NOT NULL,
    control_code TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'not_started',
    -- not_started, in_progress, implemented, failing, not_applicable
    last_test_result TEXT,
    -- pass, fail, error, not_tested
    last_test_at TIMESTAMPTZ,
    evidence_count INT NOT NULL DEFAULT 0,
    evidence_coverage TEXT NOT NULL DEFAULT 'none',
    -- full, partial, none
    owner_id UUID,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, control_id)
);

CREATE INDEX idx_rm_control_status_framework ON rm_control_status(tenant_id, framework_id, status);

CREATE TABLE rm_compliance_posture (
    tenant_id UUID NOT NULL,
    framework_id UUID NOT NULL,
    total_controls INT NOT NULL DEFAULT 0,
    implemented INT NOT NULL DEFAULT 0,
    in_progress INT NOT NULL DEFAULT 0,
    not_started INT NOT NULL DEFAULT 0,
    failing INT NOT NULL DEFAULT 0,
    not_applicable INT NOT NULL DEFAULT 0,
    compliance_pct REAL NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, framework_id)
);
```

### Evidence Status

```sql
CREATE TABLE rm_evidence_status (
    tenant_id UUID NOT NULL,
    evidence_id UUID NOT NULL,
    title TEXT NOT NULL,
    source TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending_review',
    controls_satisfied INT NOT NULL DEFAULT 0,
    collected_at TIMESTAMPTZ NOT NULL,
    expires_at TIMESTAMPTZ,
    days_until_expiry INT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, evidence_id)
);

CREATE INDEX idx_rm_evidence_expiring ON rm_evidence_status(tenant_id, expires_at)
    WHERE status = 'approved' AND expires_at IS NOT NULL;
```

### Vendor Risk

```sql
CREATE TABLE rm_vendor_risk (
    tenant_id UUID NOT NULL,
    vendor_id UUID NOT NULL,
    vendor_name TEXT NOT NULL,
    criticality TEXT NOT NULL,
    risk_level TEXT NOT NULL DEFAULT 'not_assessed',
    overall_risk_score REAL,
    open_findings INT NOT NULL DEFAULT 0,
    last_assessed_at TIMESTAMPTZ,
    next_assessment_date DATE,
    contract_expires_at DATE,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, vendor_id)
);

CREATE INDEX idx_rm_vendor_risk_level ON rm_vendor_risk(tenant_id, risk_level);
```

### Policy Compliance

```sql
CREATE TABLE rm_policy_status (
    tenant_id UUID NOT NULL,
    policy_id UUID NOT NULL,
    title TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'draft',
    current_version INT NOT NULL DEFAULT 0,
    total_employees INT NOT NULL DEFAULT 0,
    acknowledged_count INT NOT NULL DEFAULT 0,
    acknowledgement_pct REAL NOT NULL DEFAULT 0,
    next_review_date DATE,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, policy_id)
);
```

### Risk Register Summary

```sql
CREATE TABLE rm_risk_summary (
    tenant_id UUID NOT NULL,
    risk_id UUID NOT NULL,
    title TEXT NOT NULL,
    category TEXT NOT NULL,
    inherent_score INT NOT NULL,
    residual_score INT NOT NULL,
    treatment TEXT NOT NULL,
    status TEXT NOT NULL,
    owner_name TEXT,
    mitigating_controls INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, risk_id)
);

CREATE INDEX idx_rm_risk_score ON rm_risk_summary(tenant_id, residual_score DESC);
```

---

## Reference Data

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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE frameworks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    version TEXT NOT NULL,
    authority TEXT NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name, version)
);

CREATE TABLE controls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id UUID NOT NULL REFERENCES frameworks(id) ON DELETE CASCADE,
    code TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    control_type TEXT NOT NULL DEFAULT 'detective',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (framework_id, code)
);

CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    provider TEXT NOT NULL,
    display_name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    credentials_vault_ref TEXT,
    config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, provider)
);

CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name TEXT NOT NULL,
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_controls_framework ON controls(framework_id);
```

---

## Example: Compliance Timeline Reconstruction

```sql
-- Reconstruct the compliance status of a specific control over a 90-day audit period
SELECT
    occurred_at,
    event_type,
    event_data->>'status' AS status,
    event_data->>'test_result' AS test_result,
    metadata->>'actor_id' AS changed_by
FROM event_store
WHERE stream_type = 'control'
  AND stream_id = 'control-uuid'
  AND tenant_id = 'tenant-uuid'
  AND occurred_at BETWEEN '2026-01-01' AND '2026-03-31'
ORDER BY sequence_number;
```

### Example: Evidence Freshness Report

```sql
-- Find all controls where evidence has expired or will expire within 30 days
SELECT
    cs.control_code,
    cs.status,
    es.title AS evidence_title,
    es.expires_at,
    es.days_until_expiry
FROM rm_control_status cs
JOIN rm_evidence_status es ON es.tenant_id = cs.tenant_id
WHERE cs.tenant_id = 'tenant-uuid'
  AND es.status = 'approved'
  AND es.days_until_expiry <= 30
ORDER BY es.days_until_expiry;
```

### Example: SOC 2 Type 2 Operating Effectiveness

```sql
-- Show all test executions for a control during the audit period
SELECT
    occurred_at,
    event_data->>'test_code' AS test_code,
    event_data->>'result' AS result,
    event_data->>'details' AS details
FROM event_store
WHERE stream_type = 'control'
  AND stream_id = 'control-uuid'
  AND event_type IN ('control.test_passed', 'control.test_failed')
  AND occurred_at BETWEEN '2025-07-01' AND '2026-06-30'
ORDER BY occurred_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store, event_type_registry, stream_snapshots |
| Projections | 1 | projection_checkpoints |
| Read Models | 6 | rm_control_status, rm_compliance_posture, rm_evidence_status, rm_vendor_risk, rm_policy_status, rm_risk_summary |
| Reference Data | 6 | tenants, users, frameworks, controls, integrations, api_keys |
| **Total** | **16** | |

---

## Key Design Decisions

1. **Compliance state as event stream** — A control's status changing from "not_started" to "implemented" is an event, not an UPDATE. This means SOC 2 Type 2 auditors can query the event stream directly to verify operating effectiveness over the audit period.

2. **Evidence lifecycle events** — `evidence.collected`, `evidence.approved`, `evidence.expired` events create a complete chain of custody. The `rm_evidence_status` read model computes `days_until_expiry` for dashboard alerting.

3. **Policy acknowledgement as events** — Each `policy.acknowledged` event contains the user, timestamp, and policy version. The `rm_policy_status` read model aggregates acknowledgement rates in real time.

4. **Vendor risk scoring as events** — `vendor.assessed` and `vendor.risk_scored` events preserve the full history of risk assessments. When a vendor's risk changes, the event stream shows exactly when and why.

5. **Integration sync as events** — `integration.sync_completed` and `integration.sync_failed` events provide a timeline of automated evidence collection. If an auditor asks "was this integration running continuously during the audit period?", the event stream answers directly.

6. **Compliance posture as projection** — `rm_compliance_posture` aggregates per-framework compliance percentages from `rm_control_status`. Dashboard queries hit the read model; the event stream is the authoritative source.

7. **CloudEvents envelope** — `ce_source` and `ce_specversion` on every event enable forwarding to external SIEM platforms, compliance aggregators, or AI analysis pipelines via standard event infrastructure.

8. **Temporal queries for audit periods** — The most important compliance queries are temporal: "what was the state during Q1?" Event sourcing makes these queries natural — replay events up to any point in time to reconstruct historical state.
