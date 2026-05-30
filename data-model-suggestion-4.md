# Data Model Suggestion 4: Graph-Relational

> Project: SaaS Security Compliance Toolkit · Created: 2026-05-24

## Philosophy

Compliance is a graph problem. Frameworks contain controls. Controls map to other controls across frameworks. Controls are satisfied by evidence. Evidence is collected from integrations. Integrations connect to vendors. Vendors introduce risks. Risks are mitigated by controls. Policies implement controls. Users own controls, policies, and risks. Every entity in a compliance platform is connected to every other entity through a web of relationships that doesn't fit neatly into a tree.

This model uses a property graph layer (`graph_nodes` and `graph_edges` tables) alongside relational operational tables. The graph enables relationship-heavy queries that are awkward or impossible with JOINs: "what controls are affected if vendor X is compromised?", "what's the complete evidence chain for SOC 2 CC6.1 including cross-mapped ISO controls?", "which risks have no mitigating controls?", "show me all entities connected to this user."

The cross-framework mapping problem is particularly well-suited to graphs. When a SOC 2 control maps to an ISO 27001 control which maps to a NIST control, and all three are partially satisfied by the same evidence from an AWS integration that's provided by a vendor with an expiring contract — that's a graph traversal, not a JOIN chain.

**Best for:** Platforms where cross-framework overlap analysis, impact analysis ("what breaks if this vendor goes down?"), and compliance gap visualization are core features.

**Trade-offs:**
- **Pro:** Cross-framework control mapping is natural graph traversal
- **Pro:** Impact analysis ("what's affected if X fails?") is a BFS/DFS query
- **Pro:** Gap detection — orphaned nodes (controls with no evidence) are trivial to find
- **Pro:** Visualization — the graph IS the compliance map
- **Con:** Two data access patterns (relational for CRUD, graph for analysis)
- **Con:** Graph consistency must be maintained alongside relational tables
- **Con:** More complex application code for dual read paths
- **Con:** Graph queries require careful indexing to avoid full scans

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OSCAL (NIST) | Control catalog relationships modeled as graph edges |
| NIST SP 800-53 | Control enhancements as parent→child edges; family groupings as containment edges |
| AICPA TSC / ISO 27001 | Cross-framework mappings as `maps_to` edges with confidence scores |
| NIST CSF 2.0 | Function→Category→Subcategory hierarchy as graph edges |
| ISO 27005 | Risk-to-control relationships as `mitigated_by` edges |

---

## Graph Layer

```sql
CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    node_type TEXT NOT NULL CHECK (node_type IN (
        'framework', 'control', 'evidence', 'policy',
        'vendor', 'risk', 'integration', 'user', 'task'
    )),
    ref_id UUID NOT NULL,
    label TEXT NOT NULL,
    properties JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, node_type, ref_id)
);

CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    source_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type TEXT NOT NULL CHECK (edge_type IN (
        'contains',          -- framework → control, control → sub-control
        'maps_to',           -- control → control (cross-framework)
        'satisfied_by',      -- control → evidence
        'implements',        -- policy → control
        'mitigated_by',      -- risk → control
        'collected_from',    -- evidence → integration
        'provided_by',       -- integration → vendor
        'introduces',        -- vendor → risk
        'owns',              -- user → control/policy/risk/vendor
        'assigned_to',       -- task → user
        'remediates',        -- task → control/risk/finding
        'depends_on'         -- control → control (dependency)
    )),
    properties JSONB NOT NULL DEFAULT '{}',
    -- maps_to: {"mapping_type": "equivalent", "confidence": 0.95}
    -- satisfied_by: {"satisfaction_level": "full", "expires_at": "2027-01-01"}
    -- mitigated_by: {"effectiveness": "partial"}
    -- introduces: {"risk_level": "high", "data_access": ["pii"]}
    weight REAL NOT NULL DEFAULT 1.0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, target_id, edge_type)
);

CREATE INDEX idx_graph_nodes_tenant ON graph_nodes(tenant_id, node_type);
CREATE INDEX idx_graph_nodes_ref ON graph_nodes(ref_id);
CREATE INDEX idx_graph_edges_source ON graph_edges(source_id, edge_type);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_id, edge_type);
CREATE INDEX idx_graph_edges_tenant ON graph_edges(tenant_id, edge_type);
CREATE INDEX idx_graph_edges_properties ON graph_edges USING GIN (properties);
```

---

## Operational Tables

### Organization

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

CREATE INDEX idx_users_tenant ON users(tenant_id);
```

### Frameworks & Controls

```sql
CREATE TABLE frameworks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    slug TEXT NOT NULL,
    name TEXT NOT NULL,
    version TEXT NOT NULL,
    authority TEXT NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug, version)
);

CREATE TABLE controls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id UUID NOT NULL REFERENCES frameworks(id) ON DELETE CASCADE,
    parent_control_id UUID REFERENCES controls(id),
    code TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    control_type TEXT NOT NULL DEFAULT 'detective',
    metadata JSONB NOT NULL DEFAULT '{}',
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (framework_id, code)
);

CREATE TABLE control_status (
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    control_id UUID NOT NULL REFERENCES controls(id),
    status TEXT NOT NULL DEFAULT 'not_started',
    owner_id UUID REFERENCES users(id),
    last_test_result TEXT,
    last_test_at TIMESTAMPTZ,
    implementation_notes TEXT,
    due_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, control_id)
);

CREATE INDEX idx_controls_framework ON controls(framework_id);
CREATE INDEX idx_controls_parent ON controls(parent_control_id);
CREATE INDEX idx_control_status_status ON control_status(tenant_id, status);
```

### Evidence

```sql
CREATE TABLE evidence (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    description TEXT,
    source TEXT NOT NULL DEFAULT 'manual',
    status TEXT NOT NULL DEFAULT 'pending_review',
    file_url TEXT,
    file_hash TEXT,
    payload JSONB NOT NULL DEFAULT '{}',
    collected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ,
    reviewed_by UUID REFERENCES users(id),
    reviewed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_evidence_tenant ON evidence(tenant_id, status);
CREATE INDEX idx_evidence_expiry ON evidence(expires_at) WHERE status = 'approved';
```

### Policies

```sql
CREATE TABLE policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    slug TEXT NOT NULL,
    category TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'draft',
    owner_id UUID REFERENCES users(id),
    content_markdown TEXT,
    content_html TEXT,
    current_version INT NOT NULL DEFAULT 1,
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
    change_summary TEXT,
    authored_by UUID NOT NULL REFERENCES users(id),
    approved_by UUID REFERENCES users(id),
    approved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (policy_id, version_number)
);

CREATE TABLE policy_acknowledgements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id UUID NOT NULL REFERENCES policies(id) ON DELETE CASCADE,
    version_number INT NOT NULL,
    user_id UUID NOT NULL REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (policy_id, version_number, user_id)
);

CREATE INDEX idx_policies_tenant ON policies(tenant_id, status);
```

### Vendors

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
    contract_expires_at DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE vendor_assessments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vendor_id UUID NOT NULL REFERENCES vendors(id) ON DELETE CASCADE,
    assessment_type TEXT NOT NULL DEFAULT 'questionnaire',
    overall_risk_score REAL,
    risk_level TEXT NOT NULL DEFAULT 'not_assessed',
    assessed_by UUID NOT NULL REFERENCES users(id),
    findings JSONB NOT NULL DEFAULT '[]',
    assessed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    next_assessment_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vendors_tenant ON vendors(tenant_id, status);
```

### Risks

```sql
CREATE TABLE risks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    category TEXT NOT NULL,
    likelihood INT NOT NULL CHECK (likelihood BETWEEN 1 AND 5),
    impact INT NOT NULL CHECK (impact BETWEEN 1 AND 5),
    inherent_score INT GENERATED ALWAYS AS (likelihood * impact) STORED,
    residual_likelihood INT CHECK (residual_likelihood BETWEEN 1 AND 5),
    residual_impact INT CHECK (residual_impact BETWEEN 1 AND 5),
    residual_score INT GENERATED ALWAYS AS (residual_likelihood * residual_impact) STORED,
    treatment TEXT NOT NULL DEFAULT 'mitigate',
    treatment_plan TEXT,
    owner_id UUID REFERENCES users(id),
    status TEXT NOT NULL DEFAULT 'open',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risks_tenant ON risks(tenant_id, status);
```

### Integrations

```sql
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    provider TEXT NOT NULL,
    display_name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    credentials_vault_ref TEXT,
    config JSONB NOT NULL DEFAULT '{}',
    last_sync_at TIMESTAMPTZ,
    last_error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, provider)
);

CREATE TABLE test_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id UUID NOT NULL REFERENCES integrations(id) ON DELETE CASCADE,
    test_code TEXT NOT NULL,
    control_id UUID REFERENCES controls(id),
    status TEXT NOT NULL,
    resource_id TEXT,
    resource_type TEXT,
    details JSONB NOT NULL DEFAULT '{}',
    executed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_test_results_integration ON test_results(integration_id, executed_at DESC);
```

### Remediation Tasks

```sql
CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    title TEXT NOT NULL,
    description TEXT,
    source_type TEXT NOT NULL,
    source_id UUID,
    priority TEXT NOT NULL DEFAULT 'medium',
    status TEXT NOT NULL DEFAULT 'open',
    assignee_id UUID REFERENCES users(id),
    due_date DATE,
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tasks_tenant ON tasks(tenant_id, status);
```

### Audit Collaboration

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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_findings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_id UUID NOT NULL REFERENCES audits(id) ON DELETE CASCADE,
    control_id UUID REFERENCES controls(id),
    severity TEXT NOT NULL DEFAULT 'observation',
    finding_text TEXT NOT NULL,
    management_response TEXT,
    task_id UUID REFERENCES tasks(id),
    status TEXT NOT NULL DEFAULT 'open',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audits_tenant ON audits(tenant_id);
CREATE INDEX idx_audit_findings_audit ON audit_findings(audit_id);
```

### Audit Log

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
```

---

## Graph Queries

### Cross-Framework Control Overlap

```sql
-- Find all ISO 27001 controls that map (directly or transitively) to SOC 2 CC6.1
WITH RECURSIVE control_chain AS (
    -- Start from SOC 2 CC6.1
    SELECT gn.id AS node_id, gn.ref_id, gn.label, 0 AS depth
    FROM graph_nodes gn
    JOIN controls c ON c.id = gn.ref_id
    JOIN frameworks f ON f.id = c.framework_id
    WHERE gn.tenant_id = 'tenant-uuid'
      AND gn.node_type = 'control'
      AND f.slug = 'soc2'
      AND c.code = 'CC6.1'

    UNION ALL

    -- Traverse maps_to edges
    SELECT gn2.id, gn2.ref_id, gn2.label, cc.depth + 1
    FROM control_chain cc
    JOIN graph_edges ge ON ge.source_id = cc.node_id AND ge.edge_type = 'maps_to'
    JOIN graph_nodes gn2 ON gn2.id = ge.target_id
    WHERE cc.depth < 3
)
SELECT DISTINCT cc.label, c.code, f.name AS framework
FROM control_chain cc
JOIN controls c ON c.id = cc.ref_id
JOIN frameworks f ON f.id = c.framework_id
WHERE f.slug = 'iso27001';
```

### Vendor Impact Analysis

```sql
-- "What controls are at risk if vendor X is compromised?"
-- Traverse: vendor → (introduces) → risk → (mitigated_by) → control
-- Also: vendor → (provided_by) ← integration → (collected_from) ← evidence → (satisfied_by) ← control
WITH vendor_impact AS (
    -- Risks introduced by vendor
    SELECT ge.target_id AS risk_node_id
    FROM graph_nodes gn
    JOIN graph_edges ge ON ge.source_id = gn.id AND ge.edge_type = 'introduces'
    WHERE gn.tenant_id = 'tenant-uuid'
      AND gn.node_type = 'vendor'
      AND gn.ref_id = 'vendor-uuid'

    UNION

    -- Integrations provided by vendor
    SELECT ge2.target_id AS integration_node_id
    FROM graph_nodes gn
    JOIN graph_edges ge ON ge.target_id = gn.id AND ge.edge_type = 'provided_by'
    JOIN graph_edges ge2 ON ge2.target_id = ge.source_id AND ge2.edge_type = 'collected_from'
    WHERE gn.tenant_id = 'tenant-uuid'
      AND gn.node_type = 'vendor'
      AND gn.ref_id = 'vendor-uuid'
)
SELECT DISTINCT c.code, c.title, f.name AS framework
FROM vendor_impact vi
JOIN graph_edges ge ON ge.source_id = vi.risk_node_id AND ge.edge_type = 'mitigated_by'
JOIN graph_nodes gn ON gn.id = ge.target_id AND gn.node_type = 'control'
JOIN controls c ON c.id = gn.ref_id
JOIN frameworks f ON f.id = c.framework_id;
```

### Evidence Coverage Gaps (Orphan Detection)

```sql
-- Find controls with no evidence (no incoming 'satisfied_by' edges)
SELECT c.code, c.title, f.name AS framework, cs.status
FROM graph_nodes gn
JOIN controls c ON c.id = gn.ref_id
JOIN frameworks f ON f.id = c.framework_id
JOIN control_status cs ON cs.control_id = c.id AND cs.tenant_id = gn.tenant_id
LEFT JOIN graph_edges ge ON ge.source_id = gn.id AND ge.edge_type = 'satisfied_by'
WHERE gn.tenant_id = 'tenant-uuid'
  AND gn.node_type = 'control'
  AND ge.id IS NULL
  AND cs.status != 'not_applicable'
ORDER BY f.name, c.sort_order;
```

### Compliance Path Visualization

```sql
-- Build the full graph for a specific framework (for visualization)
SELECT
    gn_src.node_type AS source_type,
    gn_src.label AS source_label,
    ge.edge_type,
    gn_tgt.node_type AS target_type,
    gn_tgt.label AS target_label,
    ge.properties
FROM graph_edges ge
JOIN graph_nodes gn_src ON gn_src.id = ge.source_id
JOIN graph_nodes gn_tgt ON gn_tgt.id = ge.target_id
WHERE ge.tenant_id = 'tenant-uuid'
  AND (gn_src.node_type = 'control' OR gn_tgt.node_type = 'control')
ORDER BY gn_src.node_type, gn_src.label;
```

### Risk Propagation

```sql
-- Find risks with no mitigating controls (unmitigated risk)
SELECT r.title, r.category, r.inherent_score, r.status
FROM graph_nodes gn
JOIN risks r ON r.id = gn.ref_id
LEFT JOIN graph_edges ge ON ge.source_id = gn.id AND ge.edge_type = 'mitigated_by'
WHERE gn.tenant_id = 'tenant-uuid'
  AND gn.node_type = 'risk'
  AND ge.id IS NULL
  AND r.status = 'open'
ORDER BY r.inherent_score DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_nodes, graph_edges |
| Organization | 2 | tenants, users |
| Frameworks & Controls | 3 | frameworks, controls, control_status |
| Evidence | 1 | evidence |
| Policies | 3 | policies, policy_versions, policy_acknowledgements |
| Vendors | 2 | vendors, vendor_assessments |
| Risk Management | 1 | risks |
| Integrations | 2 | integrations, test_results |
| Remediation | 1 | tasks |
| Audit | 2 | audits, audit_findings |
| Infrastructure | 1 | audit_log |
| **Total** | **20** | |

---

## Key Design Decisions

1. **Graph layer for cross-framework mapping** — The `maps_to` edge type between control nodes makes cross-framework queries (SOC 2 → ISO 27001 → NIST) a graph traversal instead of a self-join chain on a mapping table. Edge `properties` store mapping confidence and type (equivalent, partial, related).

2. **Vendor impact analysis via graph** — The path `vendor → introduces → risk → mitigated_by → control` and `vendor → provided_by ← integration → collected_from ← evidence → satisfied_by ← control` enables answering "what breaks if this vendor goes down?" — a query that would require 4+ JOINs in a relational model.

3. **Evidence gap detection as orphan finding** — Controls with no incoming `satisfied_by` edges are immediately identifiable. This is the compliance equivalent of finding disconnected nodes — a fundamental graph operation.

4. **Control hierarchy via graph AND self-referential FK** — Controls have `parent_control_id` for simple parent-child lookups AND `contains` edges in the graph for traversal. The graph handles NIST SP 800-53's enhancement hierarchy (AC-2 → AC-2(1) → AC-2(2)) and cross-family dependencies.

5. **Ownership as graph edges** — `owns` edges from user nodes to control/policy/risk/vendor nodes enable queries like "show me everything this person is responsible for" — a single graph traversal across entity types.

6. **Operational tables remain relational** — CRUD operations (creating a vendor, updating a policy, recording a test result) use standard relational tables. The graph layer is populated via triggers or application-layer sync and serves analysis/visualization queries.

7. **Edge weights for priority** — The `weight` column on `graph_edges` supports weighted traversal. A `mitigated_by` edge with weight 0.3 (partial mitigation) differs from weight 1.0 (full mitigation), enabling risk-weighted impact analysis.

8. **Graph visualization as feature** — The compliance graph IS the product's compliance map visualization. Nodes are entities (controls, evidence, vendors, risks), edges are relationships. This model directly supports a force-directed graph UI showing a tenant's entire compliance posture.
