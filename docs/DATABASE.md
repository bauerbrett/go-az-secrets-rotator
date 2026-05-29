# Database Design — PostgreSQL

## Overview

All platform state lives in a single **Azure Database for PostgreSQL Flexible Server** instance accessed via private endpoint. The database is the source of truth for tenants, workers, rotation jobs, policies, and audit history.

Three services access the database directly:
- **API Service** — reads/writes for all CRUD operations triggered by user or worker requests
- **Scheduler** — reads jobs ready to publish, updates publish status, reads worker health
- **System Tasks** — CronJobs for cleanup, reconciliation, and retry promotion

---

## Design Principles

- **Shared schema, row-level tenant isolation** — all tenants share the same tables; every row has a `tenant_id` foreign key. No schema-per-tenant complexity at MVP.
- **UUID primary keys** — globally unique, no sequential leakage, safe to expose in APIs.
- **Timestamps everywhere** — `created_at`, `updated_at` on all tables for audit/debugging.
- **Soft deletes where appropriate** — workers and tenants are deactivated, not deleted, to preserve audit history.
- **Immutable audit log** — append-only, never updated or deleted by application code (only by retention CronJob).
- **Explicit state machines** — status columns use constrained enums with well-defined transitions.

---

## Entity Relationship Overview

```
┌─────────────┐       ┌────────────────┐
│   tenants   │──1:N──│ tenant_members │ (users ↔ tenants + roles)
└──────┬──────┘       └────────────────┘
       │
       ├──1:1──┌──────────────┐
       │       │   workers    │──1:N── worker_heartbeats
       │       └──────────────┘
       │
       ├──1:N──┌────────────┐
       │       │ key_vaults  │
       │       └──────┬─────┘
       │              │ 1:N
       │              ▼
       ├──1:N──┌──────────────────────┐
       │       │ secret_registrations  │
       │       └──────────┬───────────┘
       │                  │ 1:N
       │                  ▼
       ├──1:N──┌──────────────────┐
       │       │  rotation_jobs   │
       │       └──────────────────┘
       │
       ├──1:N──┌──────────────────┐
       │       │ rotation_policies │
       │       └──────────────────┘
       │
       └──1:N──┌──────────────┐
               │  audit_logs  │
               └──────────────┘
```

**Key relationships**:
- Tenant has exactly **one worker** (1:1) — a tenant represents an environment where the worker has private network access or been whitelisted
- Tenant has **many key vaults** — vaults are the destinations for rotated secrets. Can also have just one if thats all that is needed.
- Each secret references **one key vault** — where the new secret version is written after rotation
- No `job_leases` table — one worker per tenant means no competition for jobs

---

## Tables

### tenants

A **platform tenant** represents a logical grouping — a team, project, or environment — that owns workers, secrets, and policies. This is **not** an Azure AD tenant. Multiple platform tenants can exist within the same Azure AD organization (e.g. `team-platform`, `team-data`, `team-infra` all under `contoso.com`).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, default `gen_random_uuid()` | Unique platform tenant identifier |
| `name` | `VARCHAR(255)` | NOT NULL, UNIQUE | Human-readable name (e.g. `team-platform-prod`) |
| `slug` | `VARCHAR(100)` | NOT NULL, UNIQUE | URL-safe identifier (e.g. `team-platform-prod`) |
| `azure_ad_tenant_id` | `VARCHAR(36)` | NOT NULL | Azure AD org this tenant belongs to (guards who can join) |
| `description` | `TEXT` | | What this tenant is for |
| `status` | `VARCHAR(20)` | NOT NULL, default `'ACTIVE'` | `ACTIVE`, `SUSPENDED`, `DEACTIVATED` |
| `contact_email` | `VARCHAR(255)` | | Primary contact for notifications |
| `metadata` | `JSONB` | default `'{}'` | Freeform tags, labels, config |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |
| `deactivated_at` | `TIMESTAMPTZ` | | Set when status → DEACTIVATED |

**Indexes**:
- `UNIQUE(name)` — no duplicate tenant names
- `UNIQUE(slug)` — used in URLs and API paths
- `idx_tenants_aad_tid` on `(azure_ad_tenant_id)` — find all platform tenants for an AAD org (not unique — many:1)
- `idx_tenants_status` on `(status)` — filter active tenants

**Key point**: `azure_ad_tenant_id` is **not** unique. It's a guard rail — when a user or worker presents a JWT, the platform verifies their `tid` matches the platform tenant's `azure_ad_tenant_id`. This prevents users from one Azure AD org joining another org's tenants.

**Status transitions**:
- `ACTIVE` → `SUSPENDED` (admin action, blocks new jobs)
- `SUSPENDED` → `ACTIVE` (admin restores)
- `ACTIVE` or `SUSPENDED` → `DEACTIVATED` (soft delete, irreversible unless manually restored)

---

### tenant_members

Maps **users** to **platform tenants** with a role. A user (identified by their Azure AD OID) can be a member of multiple platform tenants with different roles. This is how the platform knows which tenants a user can manage.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, default `gen_random_uuid()` | Membership record ID |
| `tenant_id` | `UUID` | FK → tenants(id), NOT NULL | Platform tenant |
| `user_oid` | `VARCHAR(36)` | NOT NULL | Azure AD object ID of the user |
| `user_email` | `VARCHAR(255)` | | UPN / email (denormalized for display, from `preferred_username`) |
| `role` | `VARCHAR(30)` | NOT NULL | `owner`, `admin`, `operator`, `viewer` |
| `invited_by` | `VARCHAR(36)` | | OID of user who added this member |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |

**Indexes**:
- `UNIQUE(tenant_id, user_oid)` — a user has one role per tenant
- `idx_members_user` on `(user_oid)` — find all tenants a user belongs to
- `idx_members_tenant_role` on `(tenant_id, role)` — list admins for a tenant

**Roles**:
| Role | Permissions |
|------|-------------|
| `owner` | Full control: manage members, delete tenant, all admin + operator actions |
| `admin` | Approve workers, manage policies, create jobs, view audit |
| `operator` | Create jobs, view workers/jobs/audit (cannot approve workers or manage members) |
| `viewer` | Read-only access to jobs, workers, audit |

**How authorization works**:
1. User's JWT arrives at the API with `oid`, `tid`, and `roles` claims (verified by APIM + backend dual-auth)
2. If `roles` contains `Platform.Admin` → authorize immediately (global admin, skip tenant check)
3. Otherwise (`Platform.User`): look up `tenant_members` WHERE `user_oid = {oid}` AND `tenant_id = {requested-tenant}`
4. If no row → `403 Forbidden` (user is not a member of this tenant)
5. If row exists → check `role` against the required permission for the endpoint
6. Additional guard: verify user's JWT `tid` matches the tenant's `azure_ad_tenant_id` (cross-org protection)

**Azure AD App Roles** (defined on the App Registration, assigned via Enterprise App):
| Role | Effect |
|------|--------|
| `Platform.Admin` | Global access — bypasses per-tenant checks, can manage all tenants |
| `Platform.User` | Standard access — per-tenant permissions determined by this table |

> Users without an App Role in their JWT are rejected at the APIM layer before reaching the API.

---

### workers

A worker agent registered with the platform. Each tenant has exactly **one worker** — a tenant represents an environment (e.g. a VNET, subscription, or project) where the worker has private network access to the resources it rotates secrets for.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, default `gen_random_uuid()` | Platform worker ID |
| `tenant_id` | `UUID` | FK → tenants(id), NOT NULL, **UNIQUE** | Owning tenant (one worker per tenant) |
| `azure_oid` | `VARCHAR(36)` | NOT NULL | Managed Identity object ID (from JWT `oid`) |
| `name` | `VARCHAR(255)` | NOT NULL | Worker display name (e.g. `worker-prod-01`) |
| `status` | `VARCHAR(20)` | NOT NULL, default `'PENDING_APPROVAL'` | See status machine below |
| `region` | `VARCHAR(50)` | | Azure region where worker runs |
| `capabilities` | `JSONB` | default `'[]'` | Plugin types this worker supports |
| `metadata` | `JSONB` | default `'{}'` | Tags, version info, deployment context |
| `approved_by` | `VARCHAR(36)` | | OID of user who approved |
| `approved_at` | `TIMESTAMPTZ` | | When approval occurred |
| `last_heartbeat_at` | `TIMESTAMPTZ` | | Last successful heartbeat (denormalized for fast queries) |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |
| `deactivated_at` | `TIMESTAMPTZ` | | Set when status → DEACTIVATED |

**Indexes**:
- `UNIQUE(tenant_id)` — one worker per tenant
- `idx_workers_azure_oid` on `(azure_oid)` — lookup by MI OID during auth
- `idx_workers_heartbeat` on `(status, last_heartbeat_at)` — find stale workers

**Status transitions**:
```
PENDING_APPROVAL → APPROVED        (admin approves)
PENDING_APPROVAL → REJECTED        (admin rejects)
APPROVED         → SUSPENDED       (admin suspends, heartbeat timeout, or manual)
APPROVED         → DEACTIVATED     (admin removes)
SUSPENDED        → APPROVED        (admin restores)
SUSPENDED        → DEACTIVATED     (admin removes)
REJECTED         → DEACTIVATED     (cleanup)
```

---

### key_vaults

Azure Key Vaults registered to a tenant. These are the **destinations** where workers write rotated secret values. The tenant admin is responsible for granting the worker's Managed Identity RBAC access (e.g. `Key Vault Secrets Officer`) on the actual Azure Key Vault resource.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, default `gen_random_uuid()` | Vault registration ID |
| `tenant_id` | `UUID` | FK → tenants(id), NOT NULL | Owning tenant |
| `name` | `VARCHAR(255)` | NOT NULL | Human-readable name (e.g. `prod-secrets-vault`) |
| `vault_uri` | `TEXT` | NOT NULL | Azure Key Vault URI (e.g. `https://my-vault.vault.azure.net/`) |
| `description` | `TEXT` | | What this vault is used for |
| `metadata` | `JSONB` | default `'{}'` | Tags, labels |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |

**Indexes**:
- `UNIQUE(tenant_id, vault_uri)` — no duplicate vault URIs within a tenant
- `idx_vaults_tenant` on `(tenant_id)` — list vaults for a tenant

> **Authorization model**: The platform does NOT manage Azure RBAC assignments. When an admin registers a vault here, they must separately assign the worker’s MI the `Key Vault Secrets Officer` role on that vault in Azure. If they don't, rotation jobs targeting secrets in that vault will fail with a permission error reported by the worker.

---

### secret_registrations

A secret that the platform tracks for rotation. Belongs to a tenant. Each secret references a **key vault** where the rotated value is written. This is **metadata only** — the platform never stores the actual secret value.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, default `gen_random_uuid()` | Secret registration ID |
| `tenant_id` | `UUID` | FK → tenants(id), NOT NULL | Owning tenant |
| `vault_id` | `UUID` | FK → key_vaults(id), NOT NULL | Key Vault where rotated value is stored |
| `name` | `VARCHAR(255)` | NOT NULL | Human-readable name (e.g. `prod-db-password`) |
| `secret_name_in_vault` | `VARCHAR(255)` | NOT NULL | Secret name within the Key Vault |
| `secret_type` | `VARCHAR(50)` | NOT NULL | Plugin type: `keyvault`, `app_registration`, `postgresql`, `github_pat`, etc. |
| `resource_id` | `TEXT` | NOT NULL | Azure resource ID or external identifier of the secret source |
| `policy_id` | `UUID` | FK → rotation_policies(id) | Assigned rotation policy |
| `status` | `VARCHAR(20)` | NOT NULL, default `'ACTIVE'` | `ACTIVE`, `PAUSED`, `DEACTIVATED` |
| `last_rotated_at` | `TIMESTAMPTZ` | | Last successful rotation timestamp |
| `next_rotation_at` | `TIMESTAMPTZ` | | Scheduled next rotation (computed from policy) |
| `config` | `JSONB` | default `'{}'` | Plugin-specific configuration (connection info, key name, etc.) |
| `metadata` | `JSONB` | default `'{}'` | Tags, labels |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |

**Indexes**:
- `UNIQUE(tenant_id, resource_id)` — one registration per resource per tenant
- `UNIQUE(vault_id, secret_name_in_vault)` — no duplicate secret names in the same vault
- `idx_secrets_next_rotation` on `(status, next_rotation_at)` — scheduler finds due secrets
- `idx_secrets_tenant` on `(tenant_id, status)` — list secrets for a tenant
- `idx_secrets_vault` on `(vault_id)` — list secrets in a vault

---

### rotation_policies

Defines how often and how secrets should be rotated. Can be shared across multiple secret registrations within a tenant.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, default `gen_random_uuid()` | Policy ID |
| `tenant_id` | `UUID` | FK → tenants(id), NOT NULL | Owning tenant |
| `name` | `VARCHAR(255)` | NOT NULL | Policy display name |
| `rotation_interval` | `INTERVAL` | NOT NULL | How often to rotate (e.g. `30 days`, `7 days`) |
| `secret_lifetime` | `INTERVAL` | NOT NULL | How long the new credential should be valid (e.g. `55 days`) |
| `max_retry_attempts` | `INTEGER` | NOT NULL, default `3` | Retries before dead-letter |
| `retry_backoff_base` | `INTERVAL` | NOT NULL, default `'5 minutes'` | Base interval for exponential backoff |
| `notification_on_failure` | `BOOLEAN` | NOT NULL, default `true` | Notify tenant on rotation failure |
| `notification_on_success` | `BOOLEAN` | NOT NULL, default `false` | Notify tenant on success |
| `validation_required` | `BOOLEAN` | NOT NULL, default `true` | Require post-rotation validation step |
| `metadata` | `JSONB` | default `'{}'` | Additional policy config |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |

**Indexes**:
- `UNIQUE(tenant_id, name)` — unique policy names within a tenant
- `idx_policies_tenant` on `(tenant_id)` — list policies for a tenant

---

### rotation_jobs

An individual rotation job for a specific secret. Created by the scheduler (based on policy schedule) or manually by a user. Represents one attempt to rotate one secret.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, default `gen_random_uuid()` | Job ID |
| `tenant_id` | `UUID` | FK → tenants(id), NOT NULL | Owning tenant |
| `secret_id` | `UUID` | FK → secret_registrations(id), NOT NULL | Target secret |
| `worker_id` | `UUID` | FK → workers(id) | Worker that processed this job (set when worker picks it up) |
| `policy_id` | `UUID` | FK → rotation_policies(id) | Policy that triggered this job |
| `status` | `VARCHAR(20)` | NOT NULL, default `'PENDING'` | See state machine below |
| `priority` | `INTEGER` | NOT NULL, default `0` | Higher = more urgent |
| `attempt_number` | `INTEGER` | NOT NULL, default `1` | Which attempt this is (1 = first, 2+ = retry) |
| `max_attempts` | `INTEGER` | NOT NULL, default `3` | Max attempts before dead-letter |
| `scheduled_at` | `TIMESTAMPTZ` | NOT NULL | When this job should execute |
| `started_at` | `TIMESTAMPTZ` | | When a worker started processing |
| `completed_at` | `TIMESTAMPTZ` | | When processing finished (success or final failure) |
| `published_at` | `TIMESTAMPTZ` | | When published to Service Bus |
| `error_message` | `TEXT` | | Last error from worker |
| `result` | `JSONB` | | Rotation result details (new expiry, validation status) |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | |

**Indexes**:
- `idx_jobs_publish_ready` on `(status, scheduled_at)` WHERE `status = 'PENDING'` — scheduler finds jobs to publish
- `idx_jobs_tenant_status` on `(tenant_id, status)` — list jobs for a tenant by state
- `idx_jobs_secret` on `(secret_id, status)` — find active jobs for a secret (prevent duplicate scheduling)
- `idx_jobs_worker` on `(worker_id, status)` — jobs assigned to a worker

**Status state machine**:
```
PENDING          → PUBLISHED        (scheduler publishes to Service Bus)
PUBLISHED        → RUNNING          (worker picks up message, reports RUNNING)
RUNNING          → VALIDATING       (worker rotated, now validating)
VALIDATING       → COMPLETED        (validation passed)
VALIDATING       → FAILED           (validation failed)
RUNNING          → FAILED           (rotation error)
FAILED           → RETRYING         (system task promotes for retry)
RETRYING         → PENDING          (new attempt created, attempt_number++)
FAILED           → DEAD_LETTERED    (max attempts exceeded)
PUBLISHED        → PENDING          (message expired / worker unresponsive — recycle)
```

**Who transitions what**:
| Transition | Triggered By |
|------------|-------------|
| PENDING → PUBLISHED | Scheduler |
| PUBLISHED → RUNNING | Worker (via API status report) |
| RUNNING → VALIDATING | Worker |
| VALIDATING → COMPLETED | Worker |
| * → FAILED | Worker (reports failure) |
| FAILED → RETRYING | System Task (retry CronJob) |
| RETRYING → PENDING | System Task (creates new attempt) |
| FAILED → DEAD_LETTERED | System Task (max attempts reached) |
| PUBLISHED → PENDING | System Task (message timeout reconciliation) |

> **No lease table**: Since each tenant has exactly one worker, there’s no competition. The scheduler publishes to the tenant’s Service Bus topic, the worker subscribes and picks up messages. If the worker fails or goes offline, the system task detects unprocessed jobs (stuck in PUBLISHED) via timeout and recycles them.

---

### worker_heartbeats

Historical heartbeat records from workers. The `last_heartbeat_at` on the `workers` table is denormalized for fast lookups, but this table keeps the full history for diagnostics.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, default `gen_random_uuid()` | Heartbeat record ID |
| `worker_id` | `UUID` | FK → workers(id), NOT NULL | Reporting worker |
| `tenant_id` | `UUID` | FK → tenants(id), NOT NULL | For tenant-scoped queries |
| `reported_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | When heartbeat was received |
| `worker_version` | `VARCHAR(50)` | | Worker software version |
| `active_jobs` | `INTEGER` | default `0` | Jobs currently being processed |
| `cpu_usage` | `REAL` | | Optional: worker CPU load |
| `memory_usage` | `REAL` | | Optional: worker memory usage |
| `metadata` | `JSONB` | default `'{}'` | Additional health info |

**Indexes**:
- `idx_heartbeats_worker_time` on `(worker_id, reported_at DESC)` — latest heartbeat per worker
- Partition by time (monthly) for easy cleanup

**Retention**: System task deletes heartbeats older than configured retention (e.g. 30 days).

---

### audit_logs

Immutable, append-only audit trail. Every significant action is logged here. Never updated or deleted by application code.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, default `gen_random_uuid()` | Log entry ID |
| `tenant_id` | `UUID` | FK → tenants(id), NOT NULL | Tenant context |
| `actor_oid` | `VARCHAR(36)` | NOT NULL | Azure AD OID of the actor (user or worker) |
| `actor_type` | `VARCHAR(10)` | NOT NULL | `user` or `worker` |
| `action` | `VARCHAR(100)` | NOT NULL | What happened (e.g. `worker.registered`, `job.created`, `rotation.completed`) |
| `resource_type` | `VARCHAR(50)` | NOT NULL | Entity type (e.g. `worker`, `job`, `policy`, `secret`) |
| `resource_id` | `UUID` | | ID of the affected resource |
| `details` | `JSONB` | default `'{}'` | Action-specific details (before/after state, error messages) |
| `gateway_oid` | `VARCHAR(36)` | | APIM MI OID that forwarded the request |
| `ip_address` | `INET` | | Client IP (from APIM forwarded headers) |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, default `NOW()` | Immutable timestamp |

**Indexes**:
- `idx_audit_tenant_time` on `(tenant_id, created_at DESC)` — tenant audit log view
- `idx_audit_actor` on `(actor_oid, created_at DESC)` — what did this user/worker do
- `idx_audit_resource` on `(resource_type, resource_id, created_at DESC)` — history of a resource
- `idx_audit_action` on `(action, created_at DESC)` — find all events of a type
- Partition by time (monthly) for retention management

**Retention**: System task drops partitions older than configured retention (e.g. 90 days, 1 year — tenant configurable).

---

## Tenant Isolation Strategy

**Row-level isolation** — every table includes `tenant_id`. All queries from the API and scheduler include `tenant_id` in their WHERE clauses.

### Enforcement Layers

1. **Application layer** — every query includes `tenant_id` extracted from the verified JWT. Handlers never construct queries without tenant scope.
2. **PostgreSQL Row-Level Security (RLS)** — as a safety net, RLS policies on all tables enforce `tenant_id` matches the current session variable. Even if application code has a bug, the DB rejects cross-tenant reads.
3. **API layer** — the dual-auth middleware sets the tenant context; handlers cannot override it.

### RLS Design

Each service connects with a dedicated database role. At connection time (or per-transaction), the application sets a session variable:

```
SET app.current_tenant_id = '{tenant-uuid}';
```

RLS policies on each table:
```
CREATE POLICY tenant_isolation ON workers
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

This means even a query without an explicit `WHERE tenant_id = ...` will only return rows for the current tenant.

### When RLS Applies

| Service | RLS Behavior |
|---------|-------------|
| **API** | Always enabled — sets tenant from JWT on every request |
| **Scheduler** | Bypasses RLS (uses a superuser role) — needs to read across all tenants to find due jobs |
| **System Tasks** | Bypasses RLS — needs cross-tenant access for reconciliation and cleanup |

The scheduler and system tasks use a separate DB role (`az_rotator_system`) that bypasses RLS. The API uses `az_rotator_api` which has RLS enforced.

---

## Access Patterns by Service

### API Service (az-rotator-api)

| Operation | Tables | Pattern |
|-----------|--------|---------|
| Register tenant | `tenants`, `tenant_members` | INSERT tenant, INSERT owner membership |
| Get tenant | `tenants` | SELECT by id (RLS + membership check) |
| Add member | `tenant_members`, `audit_logs` | INSERT membership, INSERT audit |
| Remove member | `tenant_members`, `audit_logs` | DELETE membership, INSERT audit |
| List members | `tenant_members` | SELECT by tenant_id |
| List my tenants | `tenant_members` | SELECT by user_oid (returns tenant IDs user belongs to) |
| Register vault | `key_vaults`, `audit_logs` | INSERT vault, INSERT audit |
| List vaults | `key_vaults` | SELECT by tenant_id |
| Register worker | `workers`, `audit_logs` | INSERT worker (PENDING_APPROVAL), INSERT audit |
| Approve worker | `workers`, `audit_logs` | UPDATE status → APPROVED, INSERT audit |
| List workers | `workers` | SELECT by tenant_id + status |
| Create job (manual) | `rotation_jobs`, `audit_logs` | INSERT job (PENDING), INSERT audit |
| Report job status | `rotation_jobs`, `audit_logs` | UPDATE job status, INSERT audit |
| Worker heartbeat | `workers`, `worker_heartbeats` | UPDATE workers.last_heartbeat_at, INSERT heartbeat |
| List jobs | `rotation_jobs` | SELECT by tenant_id, paginated |
| Get audit logs | `audit_logs` | SELECT by tenant_id, paginated, filtered |
| CRUD policies | `rotation_policies` | SELECT/INSERT/UPDATE by tenant_id |
| CRUD secrets | `secret_registrations` | SELECT/INSERT/UPDATE by tenant_id |

### Scheduler (az-rotator-scheduler)

| Operation | Tables | Pattern |
|-----------|--------|---------|
| Find due secrets | `secret_registrations`, `rotation_policies` | SELECT WHERE next_rotation_at <= NOW() AND status = 'ACTIVE' |
| Create scheduled job | `rotation_jobs` | INSERT (status = PENDING) |
| Find publishable jobs | `rotation_jobs` | SELECT WHERE status = 'PENDING' AND scheduled_at <= NOW() |
| Mark job published | `rotation_jobs` | UPDATE status → PUBLISHED, set published_at |
| Find stale workers | `workers` | SELECT WHERE last_heartbeat_at < threshold |
| Mark worker suspended | `workers` | UPDATE status → SUSPENDED |

### System Tasks (az-rotator-system)

| Operation | Tables | Pattern |
|-----------|--------|---------|
| Find stale published jobs | `rotation_jobs` | SELECT WHERE status = 'PUBLISHED' AND published_at < timeout_threshold |
| Recycle stale jobs | `rotation_jobs` | UPDATE status → PENDING (reset for re-publish) |
| Promote failed jobs for retry | `rotation_jobs` | SELECT WHERE status = 'FAILED' AND attempt < max, UPDATE → RETRYING then new PENDING |
| Dead-letter exhausted jobs | `rotation_jobs` | UPDATE → DEAD_LETTERED WHERE attempts >= max |
| Cleanup old heartbeats | `worker_heartbeats` | DELETE WHERE reported_at < retention |
| Cleanup old audit logs | `audit_logs` | DROP partition or DELETE WHERE created_at < retention |

---

## Migration Strategy

**Tool**: `golang-migrate` (file-based SQL migrations, runs at service startup or as init container).

**Naming convention**: `YYYYMMDDHHMMSS_description.up.sql` / `YYYYMMDDHHMMSS_description.down.sql`

**Deployment**:
- Migrations run as a Kubernetes **init container** on the API pod before the main process starts
- The init container uses the same DB credentials (via Workload Identity)
- Only the API pod runs migrations; Scheduler and System Tasks assume schema is ready
- Schema version tracked in `schema_migrations` table (standard golang-migrate behavior)

**Initial migration** creates all tables, indexes, RLS policies, and database roles.

---

## Connection Strategy

### Roles

| Role | Used By | RLS | Permissions |
|------|---------|-----|-------------|
| `az_rotator_api` | API Service | Enforced | SELECT, INSERT, UPDATE on all tables |
| `az_rotator_system` | Scheduler, System Tasks | Bypassed | SELECT, INSERT, UPDATE, DELETE on all tables |
| `az_rotator_migrations` | Init container | Bypassed | DDL (CREATE, ALTER, DROP) + all DML |

All roles authenticate via **Azure AD authentication** (Workload Identity token exchange for PostgreSQL) — no passwords stored anywhere.

### Connection Pooling

- Application-level pooling via Go's `database/sql` with `max_open_conns` and `max_idle_conns` tuned per service
- **PgBouncer** as a sidecar or managed layer is optional for MVP but recommended at scale
- Connection idle timeout set to avoid stale connections through the private endpoint

### Private Endpoint

- PostgreSQL is not publicly accessible
- Private endpoint lives in the AKS subnet
- DNS resolution via Azure Private DNS Zone (`privatelink.postgres.database.azure.com`)
- Cilium network policies restrict DB access to only the API, Scheduler, and System Task pods

---

## Partitioning

Two tables benefit from time-based partitioning for retention and performance:

| Table | Partition Key | Granularity | Retention |
|-------|--------------|-------------|-----------|
| `audit_logs` | `created_at` | Monthly | Configurable (default 90 days) |
| `worker_heartbeats` | `reported_at` | Monthly | 30 days |

Partitioning allows the system task to drop old partitions efficiently (`DROP TABLE partition_name`) instead of slow bulk DELETEs.

---

## Backup & Recovery

- **Azure-managed backups**: Point-in-time restore up to 35 days (Flexible Server default)
- **Geo-redundant backup**: Enabled for disaster recovery across regions
- **Logical backups**: Optional `pg_dump` CronJob for compliance/export (stored in Azure Blob)
- **RTO**: < 1 hour (restore from Azure backup)
- **RPO**: < 5 minutes (continuous WAL archiving)

---

## Performance Considerations

- **Indexes tuned for read patterns** — most queries are filtered by `tenant_id` + `status`, which are covered by composite indexes
- **Partial indexes** — `WHERE status = 'ACTIVE'` or `WHERE status = 'PENDING'` to keep index size small on hot paths
- **JSONB for flexibility** — `metadata`, `config`, `details`, `result` columns use JSONB for schema evolution without migrations
- **Denormalized `last_heartbeat_at`** on workers table avoids joining heartbeats for health checks
- **Denormalized `next_rotation_at`** on secret_registrations avoids joining policies during scheduling query

---

## Open Questions / Future

- **Multi-region**: If we need active-active, consider Cosmos DB or Citus extension for distributed PostgreSQL
- **Read replicas**: If audit log reads become heavy, route read queries to a replica
- **Encryption at rest**: Azure manages this by default (service-managed keys); could upgrade to customer-managed keys (CMK) if required
- **Schema-per-tenant**: Only if a tenant requires physical data isolation (enterprise compliance). Not planned for MVP.
