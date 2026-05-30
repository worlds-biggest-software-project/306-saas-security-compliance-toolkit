# SaaS Security Compliance Toolkit — Phased Development Plan

> Project: 306-saas-security-compliance-toolkit · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-1.md`. The database design adopts **Data Model 1 (Entity-Centric Normalized Relational, OSCAL-aligned)** as the canonical schema, augmented with the immutable, append-only `audit_log` concept from Data Model 2 to guarantee SOC 2 Type II evidence of operating effectiveness over a period.

---

## Product Summary

An AI-native, open-source compliance platform that automates evidence collection, control monitoring, and audit preparation for SaaS companies pursuing **SOC 2 Type II**, **ISO/IEC 27001:2022**, and **GDPR** (MVP), with HIPAA, PCI DSS, and FedRAMP planned. It targets engineering-led seed/Series A/B teams under-served by $10K–$80K+/year incumbents (Vanta, Drata, Secureframe), and differentiates on three axes: (1) **self-hosting / air-gapped deployment** that no commercial incumbent offers; (2) **agentic AI** for natural-language automation building, policy generation, and auditor Q&A; and (3) **OSCAL-native** cross-framework control mapping ("map once, apply everywhere").

**Primary personas:** CTO needing SOC 2 Type II to close enterprise deals; security engineer automating continuous control monitoring; compliance officer running a multi-framework programme.

**Deployment model:** Self-hostable via Docker Compose (MVP) and Kubernetes (later), plus an optional managed multi-tenant cloud. The schema is multi-tenant from day one (`tenant_id` everywhere + Postgres RLS).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | **Python 3.12** | Compliance logic is LLM-heavy (policy generation, gap analysis, auditor Q&A) and integration-heavy. Python has the strongest SDK coverage for cloud providers (`boto3`, `google-cloud-*`, `azure-sdk`), the best LLM tooling, and clean async support. |
| API framework | **FastAPI** | Native Pydantic v2 validation maps directly to JSON Schema (Draft 2020-12) for evidence payloads, and auto-generates the **OpenAPI 3.1** spec that `standards.md` identifies as table stakes for every competitor's developer portal. Async-first for fan-out integration calls. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic** | Mature, supports the 33-table normalized model, Postgres-specific features (JSONB, partitioning, RLS, generated columns) used by Data Model 1, and deterministic migrations required for an auditable schema. |
| Database | **PostgreSQL 16** | Data Model 1 requires JSONB, `GENERATED ALWAYS AS` columns, partitioned tables, `INET` types, and Row-Level Security for tenant isolation. SQLite cannot express these. Single Postgres also serves as the job-state store. |
| Task queue | **Celery + Redis** (broker + result backend) | Evidence collection runs on cron-like schedules (`schedule_cron` per integration test), webhooks enqueue async work, and LLM calls are long-running. Celery Beat drives scheduled `test_results` collection. Redis doubles as a cache and rate-limit store. |
| Object storage | **S3-compatible (MinIO self-hosted / AWS S3 cloud)** | Evidence files and exported audit packages need durable, hashable blob storage. MinIO keeps the self-hosted/air-gapped story intact (no external dependency). |
| Secrets / credential vault | **Pluggable: env-encrypted (Fernet) by default, HashiCorp Vault adapter** | Integration credentials (`credentials_vault_ref`) must never sit in plaintext in `integrations`. A pluggable interface keeps self-hosting simple while allowing enterprise Vault. |
| LLM provider | **Pluggable via a `LLMProvider` interface; LiteLLM under the hood** | Self-hosting and data-sovereignty buyers need a choice of OpenAI, Anthropic, Azure OpenAI, or a local Ollama model. LiteLLM gives a single call surface across all of them. |
| Frontend | **Next.js 15 (App Router) + React + TypeScript + Tailwind + shadcn/ui** | The product is dashboard-centric (control matrix, evidence library, audit portal). Next.js gives SSR for the auditor portal, a typed client generated from the OpenAPI spec, and a polished component system. |
| Auth (app users) | **OIDC + OAuth 2.0** (Authlib server-side) issuing JWTs (RFC 7519) | `standards.md` mandates OAuth 2.0 (RFC 6749), JWT, and OIDC Core for SSO with Okta/Azure AD/Google Workspace and for the auditor portal. |
| Auth (API clients) | **OAuth 2.0 client-credentials + bearer API keys**; metadata at `.well-known/oauth-authorization-server` (RFC 8414) | Matches Vanta (OAuth client-credentials) and Drata/Secureframe (bearer key) patterns so integrators feel at home. |
| Standards I/O | **OSCAL 1.x JSON** (catalog, profile, assessment-results) | Cross-framework mapping and export are the core value prop; OSCAL is the only machine-readable standard for it and is mandatory for FedRAMP later. |
| AI tool surface | **MCP server** exposing controls, gaps, evidence, risk | `standards.md` flags compliance as an early MCP candidate; lets GRC copilots query posture in natural language. |
| Containerisation | **Docker + Docker Compose (MVP), Helm chart (later)** | Self-hosting is a primary differentiator; Compose is the turnkey path the research calls for. |
| Testing | **pytest + pytest-asyncio + httpx + testcontainers** (backend); **Vitest + Playwright** (frontend) | testcontainers spins real Postgres/Redis/MinIO for integration tests; Playwright covers the auditor-portal E2E flow. |
| Code quality | **Ruff (lint+format), mypy (strict), pre-commit** (Python); **ESLint + Prettier + tsc** (TS) | Standard, fast, deterministic gates suitable for an auditable codebase. |
| Package mgmt | **uv** (Python), **pnpm** (frontend) | Fast, lockfile-deterministic installs. |
| Licence | **AGPLv3 open-core** | Matches Comp AI's proven model; copyleft protects the open core while permitting a commercial managed tier. |

### Project Structure

```
saas-security-compliance-toolkit/
├── README.md
├── LICENSE                       # AGPLv3
├── docker-compose.yml            # postgres, redis, minio, api, worker, beat, web
├── docker-compose.dev.yml
├── .env.example
├── pyproject.toml                # uv-managed
├── alembic.ini
├── Makefile                      # make up / migrate / seed / test / lint
├── backend/
│   ├── app/
│   │   ├── main.py               # FastAPI app factory, router mounting
│   │   ├── config.py             # pydantic-settings Settings
│   │   ├── db/
│   │   │   ├── base.py           # SQLAlchemy declarative base, session
│   │   │   ├── rls.py            # set app.current_tenant_id per request
│   │   │   └── models/           # one module per domain (controls, evidence, …)
│   │   ├── schemas/              # Pydantic request/response models (OpenAPI 3.1)
│   │   ├── api/
│   │   │   ├── deps.py           # auth, tenant, pagination dependencies
│   │   │   └── v1/               # routers: controls, evidence, policies, …
│   │   ├── core/
│   │   │   ├── auth/             # OAuth2/OIDC, JWT, API keys, RBAC
│   │   │   ├── security/         # password hashing, Fernet vault, signing
│   │   │   └── audit_log.py      # append-only audit-log writer
│   │   ├── services/            # business logic (control_service, scoring, …)
│   │   ├── integrations/
│   │   │   ├── base.py           # Integration + IntegrationTest protocols
│   │   │   ├── registry.py
│   │   │   └── providers/        # aws/, github/, okta/, gcp/, azure/, slack/
│   │   ├── ai/
│   │   │   ├── provider.py       # LLMProvider interface + LiteLLM impl
│   │   │   ├── policy_gen.py
│   │   │   ├── gap_analysis.py
│   │   │   ├── auditor_qa.py     # RAG over evidence
│   │   │   └── automation_builder.py
│   │   ├── oscal/                # import/export (catalog, profile, results)
│   │   ├── mcp/                  # MCP server exposing posture
│   │   ├── workers/
│   │   │   ├── celery_app.py
│   │   │   ├── beat_schedule.py
│   │   │   └── tasks/            # collect_evidence, run_test, sync_integration
│   │   └── seed/                 # framework + control catalog seed data
│   ├── alembic/versions/
│   └── tests/
│       ├── unit/
│       ├── integration/
│       ├── e2e/
│       └── fixtures/             # OSCAL samples, mock API responses
├── frontend/
│   ├── package.json
│   ├── app/                      # Next.js App Router
│   │   ├── (dashboard)/          # control matrix, evidence, policies, risk
│   │   ├── (audit-portal)/       # external auditor views
│   │   └── api-client/           # generated from OpenAPI spec
│   ├── components/               # shadcn/ui-based
│   └── tests/                    # Vitest + Playwright
├── catalogs/                     # committed OSCAL catalogs (SOC2, ISO27001, GDPR)
└── docs/
    ├── openapi.json              # generated, committed for diff review
    └── self-hosting.md
```

---

## Phase 1: Foundation — Project Skeleton, Multi-Tenant Core, Auth

### Purpose
Establish the runnable scaffold every later phase depends on: containerised Postgres/Redis/MinIO, the FastAPI app factory, SQLAlchemy/Alembic wiring, the multi-tenant organisation tables with Row-Level Security, and the authentication/authorisation layer. After this phase a developer can register a tenant, create users, log in, obtain a JWT, and call a health-checked, OpenAPI-documented API with enforced tenant isolation.

### Tasks

#### 1.1 — Containerised dev environment & app factory

**What**: Docker Compose stack and a FastAPI application factory with config, logging, and health checks.

**Design**:
- `docker-compose.yml` services: `postgres:16`, `redis:7`, `minio`, `api` (uvicorn), `worker` (celery), `beat` (celery beat), `web` (Next.js). `api` depends_on healthy `postgres`/`redis`.
- `app/config.py` using `pydantic-settings`:
```python
class Settings(BaseSettings):
    database_url: PostgresDsn
    redis_url: RedisDsn
    s3_endpoint: str; s3_access_key: str; s3_secret_key: str; s3_bucket: str = "evidence"
    jwt_secret: str; jwt_algorithm: str = "HS256"; jwt_ttl_seconds: int = 3600
    fernet_key: str                       # 32-byte urlsafe base64
    llm_provider: str = "anthropic"       # anthropic|openai|azure|ollama
    llm_model: str = "claude-sonnet"
    environment: Literal["dev","prod"] = "dev"
    model_config = SettingsConfigDict(env_file=".env")
```
- `app/main.py`: `create_app()` mounts `/api/v1`, adds CORS, request-id middleware, exception handlers mapping domain errors to RFC 7807 problem+json, and `GET /healthz` returning `{"status":"ok","db":bool,"redis":bool}`.

**Testing**:
- `Unit: Settings loads from env with defaults applied`
- `Unit: missing required env (database_url) → ValidationError naming the field`
- `Integration (testcontainers): GET /healthz with live db+redis → 200, db:true, redis:true`
- `Integration: GET /healthz with redis down → 200, redis:false (degraded, not crashed)`

#### 1.2 — Database base, sessions, and Row-Level Security

**What**: SQLAlchemy declarative base, async session factory, Alembic baseline, and per-request RLS tenant scoping.

**Design**:
- `app/db/base.py`: async engine, `async_session` factory, `Base = DeclarativeBase` with shared `id: Mapped[UUID]` mixin (`server_default=text("gen_random_uuid()")`) and `created_at/updated_at` `TimestampMixin`.
- `app/db/rls.py`: a FastAPI dependency that, after authenticating the request, executes `SET LOCAL app.current_tenant_id = :tid` on the session so the RLS policies from Data Model 1 apply automatically.
- Alembic baseline migration enables `pgcrypto` (for `gen_random_uuid()`), creates `tenants`, `users`, `teams`, `team_members` exactly as Data Model 1 specifies, plus the `current_setting('app.current_tenant_id')` RLS policies on tenant-scoped tables.

**Testing**:
- `Unit: TimestampMixin sets created_at/updated_at on insert/update`
- `Integration (testcontainers): query evidence as tenant A returns zero of tenant B's rows (RLS enforced)`
- `Integration: session without app.current_tenant_id set → RLS blocks reads on protected tables`
- `Migration test: alembic upgrade head then downgrade base runs clean on empty db`

#### 1.3 — Authentication & RBAC

**What**: User registration/login issuing JWTs, OAuth 2.0 client-credentials for machine clients, API keys, and role-based access control.

**Design**:
- Password hashing with `argon2`. JWT (RFC 7519) claims: `sub` (user id), `tid` (tenant id), `role`, `scope`, `exp`, `iat`.
- Roles (stored on `users.role` / `team_members.role`): `owner`, `admin`, `member`, `auditor` (read-only, scoped to assigned audits), `system`.
- API keys: random 32-byte tokens stored hashed (`api_keys` table: `id, tenant_id, name, hashed_key, scopes JSONB, last_used_at, expires_at, revoked`).
- OAuth metadata endpoint `GET /.well-known/oauth-authorization-server` (RFC 8414) advertising token endpoint + supported scopes.
- Scopes: `controls:read`, `evidence:write`, `audits:read`, etc. `require_scope("evidence:write")` dependency.
- Endpoints:
  - `POST /api/v1/auth/register` → creates tenant + owner user (self-host onboarding).
  - `POST /api/v1/auth/login` → `{access_token, token_type, expires_in}`.
  - `POST /api/v1/auth/token` → OAuth client-credentials grant (RFC 6749).
  - `POST /api/v1/api-keys` / `DELETE /api/v1/api-keys/{id}`.

**Testing**:
- `Unit: valid password verifies; wrong password fails; argon2 hash never equals plaintext`
- `Unit: JWT encode→decode round-trips claims; expired token → 401`
- `Integration (mocked): login with correct creds → 200 + token; wrong creds → 401`
- `Integration: member role calls admin-only endpoint → 403`
- `Integration: auditor role reads assigned audit → 200; reads unassigned audit → 403`
- `Integration: request with revoked API key → 401`
- `Integration: GET /.well-known/oauth-authorization-server → 200 with token_endpoint`

#### 1.4 — Append-only audit log

**What**: A tamper-evident, partitioned `audit_log` writer invoked on every state-changing operation.

**Design**:
- `audit_log` table per Data Model 1 (partitioned by `RANGE (created_at)`), with a monthly partition-creation maintenance task.
- `core/audit_log.py`: `record(action, resource_type, resource_id, changes: dict, actor)` — never updated or deleted (enforced by a `BEFORE UPDATE/DELETE` trigger raising an exception). Stores a `prev_hash`/`row_hash` chain (hash of `prev_hash + canonical_json(row)`) for tamper evidence, borrowing the audit-first principle from Data Model 2.
- Wired via a SQLAlchemy `after_flush` hook on tracked models.

**Testing**:
- `Unit: record() computes row_hash = sha256(prev_hash + canonical_json); chain links correctly`
- `Integration: UPDATE on audit_log row raises (trigger blocks mutation)`
- `Integration: creating a policy writes one audit_log row with action='created' and the diff`
- `Integration: tamper a middle row → chain verification detects the break`

---

## Phase 2: Frameworks, Controls & Cross-Framework Mapping

### Purpose
Build the compliance backbone: the OSCAL-aligned frameworks/controls schema, seed catalogs for SOC 2, ISO 27001:2022, and GDPR, the cross-framework `control_mappings` engine ("map once, apply everywhere"), and per-tenant `control_objectives` tracking. After this phase a tenant can browse a control library, see which controls overlap across frameworks, and set/track implementation status — the heart of the product.

### Tasks

#### 2.1 — Framework & control schema + catalog seeding

**What**: Implement `frameworks`, `control_families`, `controls`, `control_objectives` and seed three framework catalogs.

**Design**:
- Tables exactly as Data Model 1. `controls.code` examples: SOC 2 `CC6.1`, ISO `A.8.1`, GDPR `Art.30`.
- Seed data lives as committed OSCAL JSON in `catalogs/` and is loaded by `app/seed/load_catalogs.py`:
  - **SOC 2**: AICPA TSC Common Criteria CC1–CC9 (+ optional A/PI/C/P categories) — modelled as `frameworks` row (authority `AICPA`) with `control_families` per category.
  - **ISO 27001:2022**: 93 Annex A controls in 4 themes (Organisational, People, Physical, Technological).
  - **GDPR**: key articles as controls (Art. 30 RoPA, Art. 32 security, Art. 33/34 breach notification, Art. 17 erasure).
- Seeding is **per-tenant**: on tenant creation, copy the canonical catalog rows into the tenant's `frameworks`/`controls`. A `catalog_version` is stamped for upgrade tracking.

**Testing**:
- `Unit: OSCAL catalog JSON → controls rows mapping preserves code, title, family`
- `Integration: seeding a new tenant creates 9 SOC2 CC families, 93 ISO controls`
- `Integration: re-running seed is idempotent (UNIQUE(framework_id, code) prevents dupes)`
- `Fixture: catalogs/soc2.json validates against OSCAL catalog JSON Schema`

#### 2.2 — Control CRUD & objective tracking API

**What**: REST endpoints to list/read controls and to set tenant implementation status.

**Design**:
- `GET /api/v1/frameworks` ; `GET /api/v1/frameworks/{id}/controls?family=&status=`
- `GET /api/v1/controls/{id}` → control + current objective + linked evidence count.
- `PATCH /api/v1/controls/{id}/objective` body `{status, owner_id, implementation_notes, due_date}`; `status ∈ {not_started,in_progress,implemented,not_applicable}`; sets `control_objectives` row (UNIQUE per tenant+control).
- Objective `test_result ∈ {pass,fail,partial,not_tested}` is set by the integration engine (Phase 4), not by users.

**Testing**:
- `Unit: status transition not_started→implemented allowed; invalid status → 422`
- `Integration: PATCH objective writes control_objectives row + audit_log entry`
- `Integration: list controls filtered by status=failing returns only failing`
- `Integration: tenant A cannot PATCH tenant B's control objective (RLS) → 404`

#### 2.3 — Cross-framework mapping engine

**What**: Populate and query `control_mappings` so evidence/status flows across frameworks.

**Design**:
- `control_mappings` per Data Model 1: `mapping_type ∈ {equivalent,partial,related,superset,subset}`, `confidence REAL`.
- Seed authoritative mappings from CSA CCM crosswalks and NIST SP 800-53 cross-references where available; mark inferred mappings with `confidence < 1.0` and `source='inferred'`.
- `GET /api/v1/controls/{id}/mappings` → mapped controls grouped by target framework.
- Service `propagate_status(control_id)`: when a control's objective becomes `implemented`, surface candidate auto-implementation for `equivalent` mappings (flagged for human confirmation, never auto-committed for high-stakes mappings — see legal note in features.md).

**Testing**:
- `Unit: propagate_status returns equivalent mappings only, excludes related/partial`
- `Integration: GET mappings for SOC2 CC6.1 returns mapped ISO A.8.x controls`
- `Unit: self-mapping (source==target) rejected → 422`
- `Integration: seeding mappings is idempotent (UNIQUE source,target)`

---

## Phase 3: Evidence Management & Manual Collection

### Purpose
Implement the evidence lifecycle: typed evidence with versioning, hashing, S3-backed file storage, the evidence-to-control junction enabling one evidence item to satisfy controls across frameworks, review workflow, and expiry tracking. After this phase teams can upload, tag, search, and approve evidence and trace any control to its evidence chain — satisfying the auditor traceability requirement.

### Tasks

#### 3.1 — Evidence storage, versioning & hashing

**What**: `evidence`, `evidence_types`, `evidence_versions`, `evidence_controls` tables with S3-backed uploads.

**Design**:
- Tables per Data Model 1. `status ∈ {pending_review,approved,rejected,expired,superseded}`.
- Upload flow: client requests `POST /api/v1/evidence` (metadata) → API returns a presigned MinIO/S3 PUT URL → client uploads → API computes/stores `file_hash` (SHA-256) on a confirm call. New uploads to existing evidence create an `evidence_versions` row and supersede the prior one.
- `expires_at` derived from `evidence_types.retention_days`. A Celery Beat job marks `approved`→`expired` past `expires_at` and opens a remediation task (Phase 6).

**Testing**:
- `Unit: file_hash = sha256(content); mismatch on confirm → 409`
- `Integration: upload→confirm sets status=pending_review, version 1`
- `Integration: second upload creates version 2, prior version superseded`
- `Integration (testcontainers MinIO): presigned PUT stores object retrievable by key`

#### 3.2 — Evidence ↔ control linking, tagging & search

**What**: Link evidence to controls (across frameworks) and full-text/tag search.

**Design**:
- `POST /api/v1/evidence/{id}/controls` body `{control_ids:[...], satisfaction_level}` → `evidence_controls` rows; one item may satisfy many controls in many frameworks.
- Search: Postgres `tsvector` over `title/description` + tag filter + `GET /api/v1/evidence?control_id=&source=&status=&q=`.
- `GET /api/v1/controls/{id}/evidence` → full evidence chain for an auditor.

**Testing**:
- `Unit: linking evidence to a control in another framework allowed`
- `Integration: search q='backup' returns evidence with 'backup' in title; ranked`
- `Integration: control evidence chain returns all linked approved evidence`

#### 3.3 — Review workflow

**What**: Approve/reject evidence with reviewer attribution.

**Design**:
- `POST /api/v1/evidence/{id}/review` body `{decision: approved|rejected, notes}` (requires `admin`/`owner`). Sets `reviewed_by`, `reviewed_at`, writes audit_log.
- Rejected evidence cannot be linked to satisfy a control's `test_result=pass`.

**Testing**:
- `Integration: member cannot review → 403; admin approves → status=approved`
- `Integration: review writes audit_log with decision in changes`
- `Unit: rejected evidence excluded from control satisfaction calc`

---

## Phase 4: Integrations & Automated Evidence Collection

### Purpose
Build the integration framework and the continuous, scheduled control-testing engine that automatically files evidence — the capability that distinguishes "alert-only" tools from real compliance automation. After this phase, connecting AWS/GitHub/Okta runs hourly checks (e.g. "IAM MFA enabled", "branch protection on") that update control `test_result` and produce evidence with zero human effort.

### Tasks

#### 4.1 — Integration framework & credential vault

**What**: `integrations` table, pluggable provider registry, and encrypted credential storage.

**Design**:
- `integrations`, `integration_tests`, `test_results` per Data Model 1.
- `integrations/base.py`:
```python
class IntegrationProvider(Protocol):
    provider: str
    def validate_credentials(self, cfg: dict) -> bool: ...
    def list_tests(self) -> list[TestDefinition]: ...

class IntegrationTest(Protocol):
    test_code: str; title: str; control_codes: list[str]
    def run(self, ctx: IntegrationContext) -> TestResult: ...
```
- `TestResult`: `{status: pass|fail|error|warning, resource_id, resource_type, details: dict}`.
- Credentials encrypted at rest (Fernet) and referenced by `credentials_vault_ref`; never returned by the API.
- `POST /api/v1/integrations` (provider, config, credentials) → `validate_credentials()` → status `connected|error`.

**Testing**:
- `Unit: Fernet round-trips credentials; ciphertext != plaintext`
- `Unit: registry resolves 'aws' → AWSProvider; unknown → error`
- `Integration (mocked boto3): connect AWS with valid creds → status=connected`
- `Integration: GET integration never includes decrypted credentials`

#### 4.2 — Provider implementations (AWS, GitHub, Okta)

**What**: Three reference providers with representative control tests.

**Design**:
- **AWS** (`boto3`, mocked with `moto` in tests): `aws.iam.mfa_enabled` (→ SOC 2 CC6.1, ISO A.8.x), `aws.s3.public_access_blocked`, `aws.cloudtrail.enabled` (→ CC7.x).
- **GitHub** (`PyGithub`): `github.branch_protection`, `github.2fa_required` (→ CC6.x).
- **Okta**: `okta.mfa_policy_enforced`, `okta.password_policy` (→ CC6.1).
- Each test maps `control_codes` → resolves to the tenant's `controls.id` and writes a `test_results` row; a passing test auto-creates `evidence` (source=provider) linked via `evidence_controls`, and updates `control_objectives.test_result`.

**Testing**:
- `Integration (moto): IAM user without MFA → test status=fail`
- `Integration (moto): all IAM users MFA → status=pass + evidence row created`
- `Integration (mocked PyGithub): repo lacking branch protection → fail`
- `Unit: test_code maps to correct control codes across SOC2+ISO`

#### 4.3 — Scheduled collection & webhook ingestion

**What**: Celery Beat scheduler running tests on `schedule_cron`, plus a custom-evidence webhook/API.

**Design**:
- `workers/tasks/run_test.py`: executes one `integration_test`, persists `test_results`, updates objective status, and (on transition fail→pass or pass→fail) enqueues notification + remediation logic.
- `beat_schedule.py` dynamically schedules active `integration_tests` by their cron.
- `POST /api/v1/evidence/webhook` (signed, HMAC-SHA256 header `X-Signature`) for custom external evidence submission, validated against a published **JSON Schema (Draft 2020-12)**.

**Testing**:
- `Integration: run_test for failing check sets objective.test_result=fail`
- `Integration: fail→pass transition closes the linked remediation task`
- `Integration (mocked): webhook with valid signature → evidence created, 202`
- `Integration: webhook with invalid signature → 401, no evidence written`
- `Integration: webhook payload failing JSON Schema → 422 with field path`

---

## Phase 5: Policies, Vendor Risk & Risk Register

### Purpose
Round out the GRC core that auditors expect: versioned policies with attestation, vendor risk questionnaires with scoring, and an ISO 27005-aligned risk register with computed inherent/residual scores. After this phase a tenant has the full non-integration evidence surface needed for a SOC 2 / ISO 27001 audit. These tasks can be built in parallel with Phase 4 once Phase 2 is complete.

### Tasks

#### 5.1 — Policy library with versioning & attestation

**What**: `policies`, `policy_versions`, `policy_acknowledgements`, `policy_controls`.

**Design**:
- Tables per Data Model 1. `status ∈ {draft,in_review,approved,published,archived}`. Markdown + rendered HTML stored per version.
- `POST /api/v1/policies`, `POST /api/v1/policies/{id}/versions`, `POST /api/v1/policies/{id}/approve`, `POST /api/v1/policies/{id}/acknowledge` (records `user_id`, `ip_address`, `acknowledged_at`).
- `next_review_date` derived from `review_frequency_days`; Beat job opens a remediation task when overdue.

**Testing**:
- `Integration: publish requires an approved version → else 409`
- `Integration: acknowledge records ip + timestamp; duplicate ack → idempotent (UNIQUE)`
- `Unit: next_review_date = published_at + review_frequency_days`
- `Integration: policy_controls links policy to controls; appears in control's evidence view`

#### 5.2 — Vendor risk management

**What**: `vendors`, `vendor_assessments`, `vendor_questionnaire_responses`, `vendor_controls`.

**Design**:
- Tables per Data Model 1. Ship CAIQ v4-style questionnaire templates (`standards.md`).
- Scoring: `overall_risk_score` = weighted sum of `risk_flag` responses; maps to `risk_level ∈ {critical,high,medium,low}`.
- `POST /api/v1/vendors`, `POST /api/v1/vendors/{id}/assessments`, `PUT /assessments/{id}/responses`.

**Testing**:
- `Unit: scoring 3 flagged of 10 weighted → expected score → risk_level=medium`
- `Integration: completing assessment updates vendor risk_level + next_assessment_date`
- `Unit: questionnaire response failing template schema → 422`

#### 5.3 — Risk register (ISO 27005)

**What**: `risk_register`, `risk_controls` with generated risk scores.

**Design**:
- Tables per Data Model 1; `inherent_risk_score`/`residual_risk_score` are `GENERATED ALWAYS AS (likelihood*impact)`. `treatment ∈ {accept,mitigate,transfer,avoid}`.
- `GET /api/v1/risks?sort=inherent_risk_score.desc`; link mitigating controls via `risk_controls`.

**Testing**:
- `Unit: likelihood=4, impact=5 → inherent_risk_score=20 (DB-computed)`
- `Integration: residual scores null until residual_likelihood/impact set`
- `Integration: risks sorted by inherent score desc`
- `Unit: likelihood outside 1..5 → CHECK constraint violation → 422`

---

## Phase 6: Remediation, Compliance Scoring & Dashboards

### Purpose
Turn raw control/evidence/test data into actionable signal: a remediation task tracker fed automatically by failing tests, expired evidence, and audit findings; a continuous compliance score with historical trendline; and the dashboard/control-matrix UI. After this phase the product is demonstrably usable end-to-end through a web UI.

### Tasks

#### 6.1 — Remediation tracker

**What**: `remediation_tasks`, `task_comments` auto-created from failures.

**Design**:
- Tables per Data Model 1. `source_type ∈ {test_failure,audit_finding,risk_treatment,vendor_finding,manual}`, `source_id` points back. Auto-creation deduplicates on `(source_type, source_id, control_id, status!=resolved)`.
- `POST /api/v1/tasks`, `PATCH /api/v1/tasks/{id}` (assign, status, due_date), `POST /tasks/{id}/comments`.

**Testing**:
- `Integration: test fail auto-creates one open task; repeated fails don't duplicate`
- `Integration: fail→pass auto-resolves the task with resolved_at set`
- `Integration: expired evidence opens a remediation task`

#### 6.2 — Compliance scoring & trend analysis

**What**: A per-framework score and a daily-snapshot trendline.

**Design**:
- `compliance_snapshots(id, tenant_id, framework_id, score REAL, passing INT, failing INT, needs_evidence INT, captured_at)` (additive table).
- Score = `implemented_and_passing_controls / applicable_controls`, weighted by control criticality. Beat job snapshots daily.
- `GET /api/v1/frameworks/{id}/score` (current) and `?trend=90d` (series). Predictive readiness date via linear regression on the recent slope.

**Testing**:
- `Unit: 80/100 applicable controls passing → score 0.80`
- `Unit: not_applicable controls excluded from denominator`
- `Integration: daily job writes one snapshot per framework`
- `Unit: trend slope projects readiness date; flat/negative slope → null`

#### 6.3 — Dashboard & control matrix UI

**What**: Next.js dashboard: control matrix (passing/failing/needs-evidence), evidence library, score trendline, task board.

**Design**:
- Typed API client generated from `docs/openapi.json`. Control matrix is a virtualised grid: rows=controls, status chips (green/red/amber) per Data Model 1's status enums. Evidence library with tag/search. Recharts trendline. Task kanban by status.
- Server Components for read-heavy views; auth via httpOnly JWT cookie.

**Testing**:
- `Vitest: status chip renders correct colour per status enum`
- `Vitest: control matrix filters by framework + status`
- `Playwright E2E: login → control matrix loads → open control → see evidence chain`
- `Playwright E2E: upload evidence → appears pending_review in library`

---

## Phase 7: Audit Collaboration & Auditor Portal

### Purpose
Implement the audit workflow incumbents charge most for: audits scoped to a framework/period, auditor evidence requests, evidence submission, findings, and a scoped external auditor portal. After this phase an external auditor can log in, request evidence per control, receive it, and record findings without internal staff exporting spreadsheets.

### Tasks

#### 7.1 — Audit workflow

**What**: `audits`, `audit_requests`, `audit_request_evidence`, `audit_findings`.

**Design**:
- Tables per Data Model 1. `audit_type ∈ {type_1,type_2,recertification,internal}`; request `status` machine `pending→in_progress→submitted→accepted|needs_revision`.
- `POST /api/v1/audits`, `POST /audits/{id}/requests`, `POST /requests/{id}/evidence`, `POST /audits/{id}/findings`. A finding can spawn a `remediation_task`.

**Testing**:
- `Integration: submit request without evidence → 409`
- `Integration: accept request transitions status; needs_revision reopens`
- `Integration: critical finding auto-creates a remediation task`

#### 7.2 — Scoped auditor portal

**What**: External-auditor login scoped strictly to assigned audits, read-only except request actions.

**Design**:
- `auditor` role JWT carries `audit_ids` claim. RLS + endpoint guards ensure an auditor sees only assigned audits' requests/evidence/findings. Next.js `(audit-portal)` route group with minimal UI: request list, evidence viewer, accept/needs-revision actions, finding entry.

**Testing**:
- `Integration: auditor reads assigned audit evidence → 200; other audit → 403`
- `Integration: auditor cannot create controls/policies → 403`
- `Playwright E2E: auditor logs in → opens request → views evidence → marks accepted`

#### 7.3 — Evidence export & OSCAL assessment-results

**What**: Export an audit package as a ZIP plus OSCAL assessment-results JSON.

**Design**:
- `GET /api/v1/audits/{id}/export` → ZIP of evidence files + manifest (control→evidence→hash) + `assessment-results.json` conforming to **OSCAL 1.x assessment-results**. Hashes let an auditor verify integrity against the audit_log chain.

**Testing**:
- `Integration: export ZIP contains every accepted evidence file + manifest`
- `Fixture: emitted assessment-results.json validates against OSCAL JSON Schema`
- `Unit: manifest hashes match stored file_hash values`

---

## Phase 8: AI-Native Automation

### Purpose
Deliver the differentiating agentic layer: natural-language automation building, AI policy generation, gap analysis, and auditor Q&A (RAG over the evidence library). All build on Phases 2–7 data. The `LLMProvider` abstraction keeps self-hosted/local-model deployments viable.

### Tasks

#### 8.1 — LLM provider abstraction

**What**: A single pluggable interface over OpenAI/Anthropic/Azure/Ollama via LiteLLM, with cost/latency logging.

**Design**:
```python
class LLMProvider(Protocol):
    def complete(self, system: str, user: str, *, json_schema: dict | None = None) -> LLMResult: ...
    def embed(self, texts: list[str]) -> list[list[float]]: ...
```
- All AI features go through this; responses requiring structure use `json_schema` (validated before use). Every call logged with token counts and confidence where applicable.

**Testing**:
- `Unit (mock): complete returns parsed JSON when json_schema provided`
- `Unit: schema-invalid LLM output → retried once then surfaced as low-confidence`
- `Integration: provider switch (anthropic→ollama) requires no caller change`

#### 8.2 — AI policy generation & gap analysis

**What**: Generate auditor-ready policies from a short description; produce a prioritised remediation roadmap.

**Design**:
- `POST /api/v1/ai/policies/generate` `{category, company_context}` → draft `policy_version` (status=draft, always human-reviewed). System prompt grounds output in the linked controls' requirements.
- `POST /api/v1/ai/gap-analysis` `{framework_id}` → ranks `not_started/failing` controls by risk×effort, returns roadmap with effort estimates; persists as remediation task drafts.

**Testing**:
- `Integration (mock LLM): generate returns draft tied to category's controls; never auto-published`
- `Unit: gap analysis ranks failing critical controls above passing/low ones`
- `Integration: generated roadmap items map to real control ids`

#### 8.3 — Auditor Q&A (RAG) & automation builder

**What**: Answer narrative auditor questions from evidence; build integration tests from plain English.

**Design**:
- Evidence + policy text embedded (`embed`) into a `pgvector` column; `POST /api/v1/ai/auditor-qa` `{question}` retrieves top-k chunks, synthesises an answer with **inline evidence citations**, and returns a `confidence` score — answers below threshold are `flagged_for_review` (never auto-sent), addressing the high-stakes legal note in features.md.
- `POST /api/v1/ai/automation/build` `{description}` → proposes an `IntegrationTest` definition (provider, test_code, control mapping) for human approval before activation.

**Testing**:
- `Integration (mock): Q&A cites the retrieved evidence ids in the answer`
- `Unit: confidence below threshold → flagged_for_review=true`
- `Integration: automation builder output requires explicit activation (never auto-runs)`

#### 8.4 — MCP server

**What**: Expose controls, gaps, evidence, and risk posture over the Model Context Protocol.

**Design**:
- `app/mcp/` server with read tools: `get_control_status`, `list_gaps`, `get_evidence_chain`, `get_risk_posture`. Auth via API key scoped read-only; tenant-scoped.

**Testing**:
- `Integration: MCP get_control_status returns tenant-scoped status only`
- `Integration: MCP tools reject write attempts / out-of-scope keys`

---

## Phase 9: Hardening, Self-Hosting & Release

### Purpose
Make the toolkit production- and audit-grade: security hardening against OWASP ASVS, OpenAPI publication, threat-intel-aware prioritisation, and a turnkey self-hosting/air-gapped deployment story — the headline differentiator over all commercial incumbents.

### Tasks

#### 9.1 — Security hardening (OWASP ASVS L2)

**What**: Verify the platform against OWASP ASVS 4.0.3 Level 2 and feed results into its own CC6 controls.

**Design**:
- Checklist run over auth, session, access control, crypto, error handling. Rate limiting (Redis) per `standards.md` patterns. Dependency scanning in CI. Findings recorded as internal evidence against SOC 2 CC6.

**Testing**:
- `Integration: rate limit exceeded → 429 with Retry-After`
- `Integration: IDOR attempt across tenants → 404/403 (RLS holds)`
- `Security: dependency scan gate fails CI on high-severity CVE`

#### 9.2 — OpenAPI publication & threat-intel prioritisation

**What**: Commit the OpenAPI 3.1 spec; add a CVE/threat feed that re-weights control priority.

**Design**:
- CI step exports `docs/openapi.json` and fails if it drifts from code (spec-as-test). Threat-intel job ingests a CVE feed, correlates affected technologies (from connected integrations) to controls, and bumps `remediation_tasks.priority` — the risk-aware posture no incumbent offers (features.md gap #2).

**Testing**:
- `Integration: openapi.json regenerated matches committed file (else CI fails)`
- `Unit: CVE affecting a connected provider raises related control priority`

#### 9.3 — Self-hosting & air-gapped deployment

**What**: Turnkey Docker Compose + Helm chart + offline catalog/seed bundle.

**Design**:
- `docker-compose.yml` brings up the full stack with one `make up`; `.env.example` documents every variable. Air-gapped mode: local Ollama LLM, MinIO storage, bundled OSCAL catalogs, no outbound calls. `docs/self-hosting.md` covers backup/restore of Postgres + MinIO and the audit_log integrity check.

**Testing**:
- `E2E: fresh `make up` → register → seed → connect mock integration → score renders`
- `E2E (air-gapped): stack runs with no outbound network; AI uses local model`
- `Integration: backup→restore preserves audit_log hash chain integrity`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (auth, multi-tenant, RLS, audit log)  ─── required by everything
    │
Phase 2: Frameworks, Controls & Mapping                   ─── requires P1
    │
    ├── Phase 3: Evidence Management                       ─── requires P2
    │       │
    │       └── Phase 4: Integrations & Auto-Collection    ─── requires P3
    │
    └── Phase 5: Policies, Vendor Risk, Risk Register      ─── requires P2 (parallel with P3/P4)
            │
Phase 6: Remediation, Scoring & Dashboards                ─── requires P3, P4, P5
    │
Phase 7: Audit Collaboration & Auditor Portal             ─── requires P3, P6
    │
Phase 8: AI-Native Automation                             ─── requires P3, P5, P7 (8.1 can start after P2)
    │
Phase 9: Hardening, Self-Hosting & Release                ─── requires all
```

**Parallelism opportunities:**
- **Phase 5** (Policies/Vendor/Risk) runs concurrently with **Phases 3–4** (Evidence/Integrations) once Phase 2 lands — they share only the controls schema.
- Within Phase 4, the three providers (AWS, GitHub, Okta) can be built in parallel against the `IntegrationProvider` protocol.
- **Phase 8.1** (LLM abstraction) can be built any time after Phase 2; the dependent AI features (8.2–8.4) need their respective data domains.
- Frontend work in **Phase 6.3** and **7.2** can proceed against the OpenAPI spec while backend endpoints are finalised.

---

## Definition of Done (per phase)

Every phase is complete only when:

1. All tasks implemented as specified.
2. All unit and integration tests pass; new code has meaningful coverage of happy-path and edge cases.
3. `ruff` lint + format pass; `mypy --strict` passes (backend); `eslint` + `tsc` pass (frontend).
4. `alembic upgrade head` and `downgrade base` run cleanly; new migrations are committed and reversible.
5. Row-Level Security verified for every new tenant-scoped table.
6. New/changed endpoints appear in the regenerated `docs/openapi.json` (OpenAPI 3.1) and the spec-drift CI check passes.
7. `docker compose up` builds and the affected feature works end-to-end (E2E test or documented manual check).
8. Every state-changing operation writes an `audit_log` entry; the hash chain verifies.
9. New configuration options are documented in `.env.example` and `docs/self-hosting.md`.
10. AI features never auto-commit high-stakes artefacts (published policies, sent auditor answers) without human review, per the legal note in `features.md`.
```
