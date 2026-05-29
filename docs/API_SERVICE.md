# API Service — Detailed Design

## Overview

The API service (`az-rotator-api`) is the sole REST interface between external callers and platform state. Every request arrives through APIM with dual-auth already performed — the API re-verifies both tokens and enforces fine-grained authorization before touching the database.

**Deployment**: Single pod in the `az-rotator-api` namespace (MVP), listening on `:8080`.

**Responsibilities**:
- Endpoint routing and request validation
- Dual-auth re-verification (defense in depth)
- Per-tenant authorization via `tenant_members` lookup
- All CRUD operations against PostgreSQL
- Audit logging for every state-changing action
- Health and readiness probes for Kubernetes

---

## Authorization Model Summary

Two layers determine what a caller can do:

| Layer | Source | Scope | Checked By |
|-------|--------|-------|-----------|
| **App Roles** | JWT `roles` claim (Azure AD App Registration) | Platform-wide | APIM (gate) + API (re-verify) |
| **Tenant Membership** | `tenant_members` table (PostgreSQL) | Per-tenant | API service |

**App Roles** (defined on App Registration manifest):

| Role | Effect |
|------|--------|
| `Platform.Admin` | Global access — bypasses per-tenant checks, can manage all tenants |
| `Platform.User` | Standard access — must have a `tenant_members` row for the target tenant |

**Tenant Roles** (stored in `tenant_members.role`):

| Role | Can Do |
|------|--------|
| `owner` | Everything — manage members, delete tenant, all admin + operator actions |
| `admin` | Approve/reject workers, manage policies, create jobs, manage secrets, view audit |
| `operator` | Create jobs, view workers/jobs/secrets/audit (cannot approve workers or manage members) |
| `viewer` | Read-only access to tenant resources |

**Workers** have no App Role and no `tenant_members` row. Their authorization is based solely on `APPROVED` status in the `workers` table for the target tenant.

---

## Middleware Chain

Every request passes through these middleware layers in order:

```
Request arrives (from APIM via internal LB)
    │
    ▼
┌─────────────────────────────────┐
│ 1. Request ID & Logging         │  Generate trace ID, structured logger
├─────────────────────────────────┤
│ 2. Gateway Auth Verification    │  Validate X-Gateway-Auth JWT (APIM MI)
├─────────────────────────────────┤
│ 3. Client Auth Verification     │  Validate Authorization JWT (caller)
├─────────────────────────────────┤
│ 4. Anti-Tampering Check         │  Compare JWT claims vs injected headers
├─────────────────────────────────┤
│ 5. Caller Context Injection     │  Set caller OID, type, roles in ctx
├─────────────────────────────────┤
│ 6. Tenant Resolution            │  Resolve tenant ID, set RLS session var
├─────────────────────────────────┤
│ 7. Authorization                │  Check permission for this endpoint
├─────────────────────────────────┤
│ 8. Request Validation           │  Validate body, path params, query
├─────────────────────────────────┤
│ 9. Handler                      │  Business logic
├─────────────────────────────────┤
│ 10. Audit Logging               │  Write to audit_logs (post-handler)
└─────────────────────────────────┘
```

### Middleware Details

**1. Request ID & Logging** — Generates a unique request ID (UUID), attaches a structured logger to the context with metadata (request ID, method, path, client IP). All downstream logs include these fields.

**2. Gateway Auth Verification** — Parses `X-Gateway-Auth` header, validates the JWT (signature, issuer, audience, expiry), confirms `oid` matches the known APIM Managed Identity object ID. Rejects with `401` if invalid.

**3. Client Auth Verification** — Parses `Authorization: Bearer` header, validates the client JWT against Azure AD. Extracts `oid`, `tid`, `roles`, `preferred_username` claims. Rejects with `401` if invalid.

**4. Anti-Tampering Check** — Compares JWT `oid` against `X-Caller-OID` header. If they differ, APIM-injected headers were tampered with in transit. Rejects with `401`.

**5. Caller Context Injection** — Packages verified identity info into request context: caller OID, caller type (user/worker), roles, email, Azure AD tenant ID.

**6. Tenant Resolution** — Reads `X-Tenant-ID` header. For tenant-scoped endpoints, sets PostgreSQL session variable `SET app.current_tenant_id = '{uuid}'` to activate RLS. For tenant-creation endpoints, skips RLS.

**7. Authorization** — Calls the authorization function for the matched endpoint (see endpoint table below). This is where `Platform.Admin` bypass, `tenant_members` lookup, or worker status check happens.

**8. Request Validation** — Validates path parameters (UUIDs are valid), query parameters (pagination bounds), and request body (required fields, field lengths, enum values).

---

## Endpoint Catalog

### Notation

- **Caller**: `user` or `worker`
- **Min Role**: minimum `tenant_members` role required (for users). `Platform.Admin` always bypasses.
- **Tenant-scoped**: whether the endpoint requires tenant context and RLS

---

### Tenants

| Method | Path | Caller | Min Role | Description |
|--------|------|--------|----------|-------------|
| POST | `/api/tenants` | user | `Platform.Admin` | Create a new tenant |
| GET | `/api/tenants` | user | *(none — returns only user's tenants)* | List tenants the caller belongs to |
| GET | `/api/tenants/{tenantId}` | user | `viewer` | Get tenant details |
| PATCH | `/api/tenants/{tenantId}` | user | `owner` | Update tenant (name, description, metadata) |
| DELETE | `/api/tenants/{tenantId}` | user | `owner` | Deactivate tenant (soft delete) |

#### POST /api/tenants — Create Tenant

**Authorization**: `Platform.Admin` only. Regular users cannot self-create tenants — a platform admin provisions them.

**What it does**:
1. Validate request body (name, slug, optional description/metadata)
2. Extract caller's `tid` from JWT → set as the tenant's `azure_ad_tenant_id`
3. Insert into `tenants` (status = `ACTIVE`)
4. Insert into `tenant_members` (caller becomes `owner`)
5. Audit log: `tenant.created`
6. Return tenant object with ID

**Request body**:
- `name` (required) — human-readable tenant name
- `slug` (required) — URL-safe identifier, validated: lowercase alphanumeric + hyphens
- `description` (required) — what this tenant is for
- `contact_email` (required) — notification target
- `metadata` (optional) — freeform key-value tags

**Response**: `201 Created` with full tenant object + caller's membership

---

#### GET /api/tenants — List My Tenants

**Authorization**: Any authenticated user. Returns only tenants where the caller has a `tenant_members` row. `Platform.Admin` sees all tenants.

**What it does**:
1. Query `tenant_members` WHERE `user_oid = {caller-oid}` → get list of tenant IDs
2. Query `tenants` WHERE `id IN (...)` AND `status != 'DEACTIVATED'`
3. Return list with caller's role in each

**Query params**: `?status=ACTIVE` (optional filter), `?page=1&per_page=20`

**Response**: `200 OK` with paginated list of tenants + caller's role in each

---

#### GET /api/tenants/{tenantId} — Get Tenant

**Authorization**: User must be a member (`viewer` or above), or `Platform.Admin`.

**What it does**:
1. Look up tenant by ID
2. Return tenant details

**Response**: `200 OK` with tenant object

---

#### PATCH /api/tenants/{tenantId} — Update Tenant

**Authorization**: `owner` or `Platform.Admin`.

**What it does**:
1. Validate update fields (name, description, contact_email, metadata)
2. Update tenant record
3. Audit log: `tenant.updated` with before/after diff

**Response**: `200 OK` with updated tenant

---

#### DELETE /api/tenants/{tenantId} — Deactivate Tenant

**Authorization**: `owner` or `Platform.Admin`.

**What it does**:
1. Set tenant status → `DEACTIVATED`, set `deactivated_at`
2. Cancel all PENDING/PUBLISHED jobs for this tenant
3. Audit log: `tenant.deactivated`

**Response**: `204 No Content`

---

### Tenant Members

| Method | Path | Caller | Min Role | Description |
|--------|------|--------|----------|-------------|
| GET | `/api/tenants/{tenantId}/members` | user | `admin` | List all members of a tenant |
| POST | `/api/tenants/{tenantId}/members` | user | `admin` | Add a member to the tenant |
| PATCH | `/api/tenants/{tenantId}/members/{memberId}` | user | `owner` | Change a member's role |
| DELETE | `/api/tenants/{tenantId}/members/{memberId}` | user | `admin` | Remove a member |

#### POST /api/tenants/{tenantId}/members — Add Member

**Authorization**: `admin` or `owner`. Cannot assign a role higher than your own (admin cannot create an owner).

**What it does**:
1. Validate request (user_oid or user_email required, role required)
2. Verify the target user's Azure AD `tid` matches the tenant's `azure_ad_tenant_id` (can't add users from a different org)
3. Insert into `tenant_members`
4. Audit log: `member.added`

**Request body**:
- `user_oid` (required) — Azure AD object ID of user to add
- `user_email` (optional) — display hint (denormalized)
- `role` (required) — `admin`, `operator`, or `viewer` (owner must use `PATCH` to transfer ownership)

**Response**: `201 Created` with membership record

---

#### PATCH /api/tenants/{tenantId}/members/{memberId} — Change Role

**Authorization**: `owner` only. Cannot demote yourself below `owner` unless another owner exists.

**What it does**:
1. Validate new role
2. Update `tenant_members.role`
3. Audit log: `member.role_changed` with old → new

**Response**: `200 OK` with updated membership

---

#### DELETE /api/tenants/{tenantId}/members/{memberId} — Remove Member

**Authorization**: `admin` or `owner`. Cannot remove the last owner (prevents orphaned tenants). Cannot remove yourself (use a different endpoint or transfer ownership first).

**What it does**:
1. Validate constraints (not last owner, not self)
2. Delete from `tenant_members`
3. Audit log: `member.removed`

**Response**: `204 No Content`

---

### Workers

| Method | Path | Caller | Min Role | Description |
|--------|------|--------|----------|-------------|
| POST | `/api/workers/register` | worker | *(worker auth)* | Register a new worker |
| POST | `/api/workers/heartbeat` | worker | *(worker auth)* | Report heartbeat |
| GET | `/api/tenants/{tenantId}/workers` | user | `operator` | List workers for a tenant |
| GET | `/api/tenants/{tenantId}/workers/{workerId}` | user | `operator` | Get worker details |
| POST | `/api/tenants/{tenantId}/workers/{workerId}/approve` | user | `admin` | Approve a pending worker |
| POST | `/api/tenants/{tenantId}/workers/{workerId}/reject` | user | `admin` | Reject a pending worker |
| POST | `/api/tenants/{tenantId}/workers/{workerId}/suspend` | user | `admin` | Suspend an approved worker |
| POST | `/api/tenants/{tenantId}/workers/{workerId}/restore` | user | `admin` | Restore a suspended worker |
| DELETE | `/api/tenants/{tenantId}/workers/{workerId}` | user | `admin` | Deactivate a worker |

#### POST /api/workers/register — Worker Registration

**Authorization**: Worker auth (valid MI JWT, no roles). The worker's JWT `tid` must match the `azure_ad_tenant_id` of the tenant specified in `X-Tenant-ID`.

**What it does**:
1. Extract worker OID from JWT, tenant ID from `X-Tenant-ID` header
2. Verify tenant exists and is `ACTIVE`
3. Verify worker's JWT `tid` matches tenant's `azure_ad_tenant_id`
4. Check for existing registration (same OID + tenant) — if exists and not `DEACTIVATED`, return conflict
5. Insert into `workers` (status = `PENDING_APPROVAL`)
6. Audit log: `worker.registered`
7. Return registration ID and status

**Request body**:
- `name` (required) — display name for the worker
- `capabilities` (optional) — array of plugin types this worker supports (e.g. `["keyvault", "postgresql"]`)
- `region` (optional) — Azure region
- `metadata` (optional) — version info, tags

**Response**: `201 Created` with worker record (status = `PENDING_APPROVAL`)

---

#### POST /api/workers/heartbeat — Worker Heartbeat

**Authorization**: Worker auth. Worker must be in `APPROVED` status.

**What it does**:
1. Look up worker by OID + tenant ID
2. Verify worker status = `APPROVED`
3. Update `workers.last_heartbeat_at`
4. Insert into `worker_heartbeats` (historical record)
5. Return acknowledgment with any pending commands (future: config updates)

**Request body**:
- `worker_version` (optional) — current software version
- `active_jobs` (optional) — number of jobs currently processing
- `cpu_usage` (optional) — reported CPU load
- `memory_usage` (optional) — reported memory usage
- `metadata` (optional) — additional health info

**Response**: `200 OK` with `{ "ack": true, "server_time": "..." }`

---

#### POST /api/tenants/{tenantId}/workers/{workerId}/approve — Approve Worker

**Authorization**: `admin` or `owner` in the tenant.

**What it does**:
1. Verify worker exists, belongs to tenant, and is in `PENDING_APPROVAL` status
2. Update worker status → `APPROVED`, set `approved_by` and `approved_at`
3. Create Service Bus subscription for the worker on the tenant's topic
4. Audit log: `worker.approved`

**Response**: `200 OK` with updated worker record

---

#### POST /api/tenants/{tenantId}/workers/{workerId}/reject — Reject Worker

**Authorization**: `admin` or `owner`.

**What it does**:
1. Verify worker is in `PENDING_APPROVAL`
2. Update status → `REJECTED`
3. Audit log: `worker.rejected`

**Response**: `200 OK` with updated worker record

---

#### POST /api/tenants/{tenantId}/workers/{workerId}/suspend — Suspend Worker

**Authorization**: `admin` or `owner`.

**What it does**:
1. Verify worker is `APPROVED`
2. Update status → `SUSPENDED`
3. Any in-flight jobs for this tenant will fail (worker won't be processing)
4. Audit log: `worker.suspended`

**Response**: `200 OK` with updated worker record

---

#### POST /api/tenants/{tenantId}/workers/{workerId}/restore — Restore Worker

**Authorization**: `admin` or `owner`.

**What it does**:
1. Verify worker is `SUSPENDED`
2. Update status → `APPROVED`
3. Audit log: `worker.restored`

**Response**: `200 OK` with updated worker record

---

#### DELETE /api/tenants/{tenantId}/workers/{workerId} — Deactivate Worker

**Authorization**: `admin` or `owner`.

**What it does**:
1. Update status → `DEACTIVATED`, set `deactivated_at`
2. Remove Service Bus subscription
3. Audit log: `worker.deactivated`

**Response**: `204 No Content`

---

### Secret Registrations

| Method | Path | Caller | Min Role | Description |
|--------|------|--------|----------|-------------|
| GET | `/api/tenants/{tenantId}/secrets` | user | `operator` | List registered secrets |
| GET | `/api/tenants/{tenantId}/secrets/{secretId}` | user | `operator` | Get secret details |
| POST | `/api/tenants/{tenantId}/secrets` | user | `admin` | Register a new secret |
| PATCH | `/api/tenants/{tenantId}/secrets/{secretId}` | user | `admin` | Update secret config |
| DELETE | `/api/tenants/{tenantId}/secrets/{secretId}` | user | `admin` | Deactivate a secret |
| POST | `/api/tenants/{tenantId}/secrets/{secretId}/pause` | user | `admin` | Pause rotation |
| POST | `/api/tenants/{tenantId}/secrets/{secretId}/resume` | user | `admin` | Resume rotation |

#### POST /api/tenants/{tenantId}/secrets — Register Secret

**Authorization**: `admin` or `owner`.

**What it does**:
1. Validate request (name, secret_type, resource_id, vault_id required)
2. Verify no existing active registration for same resource_id in tenant
3. Verify `vault_id` exists and belongs to tenant
4. If `policy_id` provided, verify policy exists in tenant
5. Insert into `secret_registrations` (status = `ACTIVE`)
6. Compute `next_rotation_at` from policy interval
7. Audit log: `secret.registered`

**Request body**:
- `name` (required) — human-readable name
- `secret_type` (required) — plugin type (`keyvault`, `app_registration`, `postgresql`, `github_pat`, etc.)
- `resource_id` (required) — Azure resource ID or external identifier of the secret source
- `vault_id` (required) — which registered Key Vault the rotated secret value is written to
- `secret_name_in_vault` (required) — the secret name within the Key Vault (e.g. `prod-db-password`)
- `policy_id` (optional) — rotation policy to assign
- `config` (optional) — plugin-specific configuration (connection info, etc.)
- `metadata` (optional) — tags

**Response**: `201 Created` with secret registration record

> **Key Vault authorization**: The tenant admin must ensure the worker's MI has `Key Vault Secrets Officer` (or equivalent) RBAC on the target vault. The platform does not assign this — it only tracks which vault a secret targets. If the worker lacks permission, the rotation will fail and be reported back via the status endpoint.

---

#### PATCH /api/tenants/{tenantId}/secrets/{secretId} — Update Secret

**Authorization**: `admin` or `owner`.

**What it does**:
1. Validate update fields
2. If changing policy, recompute `next_rotation_at`
3. Update record
4. Audit log: `secret.updated`

**Response**: `200 OK` with updated record

---

### Key Vaults

| Method | Path | Caller | Min Role | Description |
|--------|------|--------|----------|-------------|
| GET | `/api/tenants/{tenantId}/vaults` | user | `operator` | List registered key vaults |
| GET | `/api/tenants/{tenantId}/vaults/{vaultId}` | user | `operator` | Get vault details |
| POST | `/api/tenants/{tenantId}/vaults` | user | `admin` | Register a key vault |
| PATCH | `/api/tenants/{tenantId}/vaults/{vaultId}` | user | `admin` | Update vault metadata |
| DELETE | `/api/tenants/{tenantId}/vaults/{vaultId}` | user | `admin` | Remove a vault registration |

Key vaults are the **destination** where workers write rotated secret values. Each secret registration references which vault it targets. The tenant admin/owner is responsible for granting the worker's Managed Identity the appropriate RBAC role (e.g. `Key Vault Secrets Officer`) on the Azure Key Vault resource before secrets can be rotated.

#### POST /api/tenants/{tenantId}/vaults — Register Vault

**Authorization**: `admin` or `owner`.

**What it does**:
1. Validate request (vault_uri required, must be a valid Key Vault URI)
2. Verify no duplicate vault_uri within tenant
3. Insert into `key_vaults`
4. Audit log: `vault.registered`

**Request body**:
- `name` (required) — human-readable name (e.g. `prod-secrets-vault`)
- `vault_uri` (required) — Azure Key Vault URI (e.g. `https://my-vault.vault.azure.net/`)
- `description` (optional) — what this vault is used for
- `metadata` (optional) — tags

**Response**: `201 Created` with vault record

> **Important**: Registering a vault does NOT grant the worker access. The tenant admin must separately assign RBAC on the Azure Key Vault resource to the worker's Managed Identity. The platform tracks this relationship but does not perform the RBAC assignment — that's an Azure control plane operation owned by the tenant.

---

### Rotation Jobs

| Method | Path | Caller | Min Role | Description |
|--------|------|--------|----------|-------------|
| GET | `/api/tenants/{tenantId}/jobs` | user | `operator` | List rotation jobs |
| GET | `/api/tenants/{tenantId}/jobs/{jobId}` | user | `operator` | Get job details |
| POST | `/api/tenants/{tenantId}/jobs` | user | `admin` | Create a manual rotation job |
| POST | `/api/tenants/{tenantId}/secrets/{secretId}/rotate` | user | `admin` | Trigger immediate rotation (convenience) |
| POST | `/api/rotations/{jobId}/status` | worker | *(worker auth)* | Report job status update |

#### POST /api/tenants/{tenantId}/jobs — Create Manual Job

**Authorization**: `admin` or above.

**What it does**:
1. Validate request (secret_id required)
2. Verify secret exists, is `ACTIVE`, belongs to tenant
3. Verify secret's key vault is registered and worker has been noted as authorized
4. Check no existing PENDING/PUBLISHED/RUNNING job for same secret (prevent duplicates)
5. Insert into `rotation_jobs` (status = `PENDING`, scheduled_at = NOW)
6. Audit log: `job.created`

**Request body**:
- `secret_id` (required) — which secret to rotate
- `priority` (optional, default 0) — higher = more urgent
- `reason` (optional) — why this manual rotation was triggered

**Response**: `201 Created` with job record

---

#### POST /api/tenants/{tenantId}/secrets/{secretId}/rotate — Trigger Rotation

**Authorization**: `admin` or above.

**What it does**: Convenience shorthand — creates a manual rotation job for a specific secret without requiring `secret_id` in the body. Identical logic to `POST /api/tenants/{tid}/jobs` with `secret_id` pre-filled from the URL.

1. Validate secret exists, is ACTIVE, belongs to tenant
2. Check no existing PENDING/PUBLISHED/RUNNING job for this secret
3. Insert into `rotation_jobs` (status = `PENDING`, scheduled_at = NOW, priority = 10)
4. Audit log: `job.created` (source: `manual`)

**Request body** (optional):
- `priority` (optional, default 10 — elevated over scheduled jobs)
- `reason` (optional) — why this rotation was triggered

**Response**: `201 Created` with job record

> **CLI usage**: A future `az-rotator` CLI tool would call this endpoint to trigger ad-hoc rotations:
> ```bash
> az-rotator rotate --tenant team-platform-prod --secret prod-db-password
> # → POST /api/tenants/{tid}/secrets/{sid}/rotate
> ```

---

#### GET /api/tenants/{tenantId}/jobs — List Jobs

**Authorization**: `operator` or above.

**What it does**:
1. Query `rotation_jobs` for tenant with filters
2. Return paginated list

**Query params**:
- `?status=PENDING,RUNNING,COMPLETED` — filter by status (comma-separated)
- `?secret_id={uuid}` — filter by secret
- `?worker_id={uuid}` — filter by assigned worker
- `?since=2026-01-01T00:00:00Z` — jobs created after this time
- `?page=1&per_page=50`

**Response**: `200 OK` with paginated job list

---

#### POST /api/rotations/{jobId}/status — Report Job Status

**Authorization**: Worker auth. Worker must be the tenant's approved worker and the job must belong to that tenant.

**What it does**:
1. Verify worker is `APPROVED` for the job's tenant
2. Validate status transition is legal (see state machine)
3. Update `rotation_jobs.status`, set `worker_id` if not already set
4. If RUNNING: set `started_at`
5. If COMPLETED: set `completed_at`, update `secret_registrations.last_rotated_at` and compute `next_rotation_at`
6. If FAILED: set `error_message`
7. Audit log: `job.status_updated`

**Request body**:
- `status` (required) — new status: `RUNNING`, `VALIDATING`, `COMPLETED`, `FAILED`
- `error_message` (optional) — if FAILED, what went wrong
- `result` (optional) — rotation result details (new expiry, validation info)

**Response**: `200 OK` with updated job record

> **No lease required**: Since there is exactly one worker per tenant, there's no competition for jobs. The scheduler publishes to the tenant's Service Bus topic and the tenant's single worker subscribes. No lease-based coordination needed.

---

### Rotation Policies

| Method | Path | Caller | Min Role | Description |
|--------|------|--------|----------|-------------|
| GET | `/api/tenants/{tenantId}/policies` | user | `operator` | List policies |
| GET | `/api/tenants/{tenantId}/policies/{policyId}` | user | `operator` | Get policy details |
| POST | `/api/tenants/{tenantId}/policies` | user | `admin` | Create a policy |
| PATCH | `/api/tenants/{tenantId}/policies/{policyId}` | user | `admin` | Update a policy |
| DELETE | `/api/tenants/{tenantId}/policies/{policyId}` | user | `admin` | Delete a policy |

#### POST /api/tenants/{tenantId}/policies — Create Policy

**Authorization**: `admin` or `owner`.

**What it does**:
1. Validate request (name, rotation_interval required)
2. Verify name is unique within tenant
3. Insert into `rotation_policies`
4. Audit log: `policy.created`

**Request body**:
- `name` (required) — policy display name
- `rotation_interval` (required) — e.g. `"30d"`, `"7d"`, `"1h"` (parsed to PostgreSQL interval)
- `max_retry_attempts` (optional, default 3)
- `retry_backoff_base` (optional, default `"5m"`)
- `validation_required` (optional, default true)
- `notification_on_failure` (optional, default true)
- `notification_on_success` (optional, default false)
- `metadata` (optional)

**Response**: `201 Created` with policy record

---

#### DELETE /api/tenants/{tenantId}/policies/{policyId} — Delete Policy

**Authorization**: `admin` or `owner`.

**What it does**:
1. Check if any active secrets reference this policy
2. If yes → return `409 Conflict` (must reassign secrets first)
3. If no → delete policy
4. Audit log: `policy.deleted`

**Response**: `204 No Content` or `409 Conflict`

---

### Audit Logs

| Method | Path | Caller | Min Role | Description |
|--------|------|--------|----------|-------------|
| GET | `/api/tenants/{tenantId}/audit` | user | `operator` | Query audit logs |

#### GET /api/tenants/{tenantId}/audit — Query Audit Logs

**Authorization**: `operator` or above.

**What it does**:
1. Query `audit_logs` for the tenant with filters
2. Return paginated results (newest first)

**Query params**:
- `?action=worker.approved,job.created` — filter by action type
- `?actor_oid={uuid}` — filter by who did it
- `?resource_type=worker` — filter by resource type
- `?resource_id={uuid}` — filter by specific resource
- `?since=2026-01-01T00:00:00Z` — logs after this time
- `?until=2026-02-01T00:00:00Z` — logs before this time
- `?page=1&per_page=100`

**Response**: `200 OK` with paginated audit log entries

---

### Health & Readiness

| Method | Path | Caller | Auth | Description |
|--------|------|--------|------|-------------|
| GET | `/healthz` | k8s | none | Liveness probe |
| GET | `/readyz` | k8s | none | Readiness probe |

These are **not** exposed through APIM — they're internal-only, hit by Kubernetes probes.

#### GET /healthz — Liveness

Returns `200 OK` if the process is alive. Does not check dependencies (a DB outage should not cause pod restarts).

#### GET /readyz — Readiness

Returns `200 OK` if the service can handle traffic:
- PostgreSQL connection pool has at least one healthy connection
- JWKS keys have been fetched at least once (can validate tokens)

Returns `503 Service Unavailable` if not ready (k8s removes pod from service endpoints).

---

## Response Envelope

All responses use a consistent JSON structure:

**Success (single resource)**:
```
{
  "data": { ... },
  "meta": {
    "request_id": "uuid"
  }
}
```

**Success (list)**:
```
{
  "data": [ ... ],
  "meta": {
    "request_id": "uuid",
    "page": 1,
    "per_page": 50,
    "total": 142
  }
}
```

**Error**:
```
{
  "error": {
    "code": "FORBIDDEN",
    "message": "You do not have admin access to this tenant",
    "details": {}
  },
  "meta": {
    "request_id": "uuid"
  }
}
```

### Standard Error Codes

| HTTP Status | Code | When |
|-------------|------|------|
| 400 | `VALIDATION_ERROR` | Request body or params failed validation |
| 401 | `UNAUTHORIZED` | JWT invalid, expired, or gateway auth failed |
| 403 | `FORBIDDEN` | Caller lacks required role or membership |
| 404 | `NOT_FOUND` | Resource doesn't exist (or caller can't see it) |
| 409 | `CONFLICT` | Duplicate resource or illegal state transition |
| 422 | `UNPROCESSABLE` | Valid syntax but semantically wrong (e.g. assigning deleted policy) |
| 429 | `RATE_LIMITED` | Too many requests (should be caught at APIM, but defense in depth) |
| 500 | `INTERNAL_ERROR` | Unexpected server error |

---

## Authorization Decision Matrix

Complete mapping of caller type × endpoint → authorization check:

### User Endpoints

| Endpoint | Platform.Admin | Platform.User (with tenant role) | Platform.User (no membership) |
|----------|----------------|-----------------------------------|-------------------------------|
| Create tenant | ✅ | ❌ 403 | ❌ 403 |
| List my tenants | ✅ (sees all) | ✅ (sees own) | ✅ (empty list) |
| Get tenant | ✅ | viewer+ | ❌ 403 |
| Update tenant | ✅ | owner | ❌ 403 |
| Delete tenant | ✅ | owner | ❌ 403 |
| List members | ✅ | admin+ | ❌ 403 |
| Add member | ✅ | admin+ | ❌ 403 |
| Change member role | ✅ | owner | ❌ 403 |
| Remove member | ✅ | admin+ | ❌ 403 |
| List workers | ✅ | operator+ | ❌ 403 |
| Approve worker | ✅ | admin+ | ❌ 403 |
| Reject worker | ✅ | admin+ | ❌ 403 |
| Suspend worker | ✅ | admin+ | ❌ 403 |
| Restore worker | ✅ | admin+ | ❌ 403 |
| Deactivate worker | ✅ | admin+ | ❌ 403 |
| List secrets | ✅ | operator+ | ❌ 403 |
| Register secret | ✅ | admin+ | ❌ 403 |
| Update secret | ✅ | admin+ | ❌ 403 |
| Deactivate secret | ✅ | admin+ | ❌ 403 |
| List jobs | ✅ | operator+ | ❌ 403 |
| Create job | ✅ | operator+ | ❌ 403 |
| List policies | ✅ | operator+ | ❌ 403 |
| Create policy | ✅ | admin+ | ❌ 403 |
| Update policy | ✅ | admin+ | ❌ 403 |
| Delete policy | ✅ | admin+ | ❌ 403 |
| View audit logs | ✅ | operator+ | ❌ 403 |

### Worker Endpoints

| Endpoint | Worker (APPROVED) | Worker (PENDING/SUSPENDED) | Worker (wrong tenant) |
|----------|-------------------|----------------------------|----------------------|
| Register | ✅ (if no existing active registration) | N/A (not registered yet) | ❌ 403 (tid mismatch) |
| Heartbeat | ✅ | ❌ 403 | ❌ 403 |
| Report status | ✅ (job must belong to worker's tenant) | ❌ 403 | ❌ 403 |

---

## Go Project Structure

```
cmd/
  api/
    main.go                     ← Entry point: config, DI, server start

internal/
  api/
    server.go                   ← HTTP server setup, route registration
    routes.go                   ← Route definitions, middleware wiring
    middleware/
      request_id.go             ← Request ID generation
      gateway_auth.go           ← X-Gateway-Auth JWT verification
      client_auth.go            ← Authorization JWT verification
      anti_tamper.go            ← Header vs claim comparison
      caller_context.go         ← Inject caller identity into context
      tenant_resolver.go        ← Resolve tenant, set RLS session var
      authorization.go          ← Permission check dispatcher
      request_logger.go         ← Structured request/response logging

    handler/
      tenants.go                ← Tenant CRUD handlers
      members.go                ← Tenant membership handlers
      workers.go                ← Worker registration, approval, lifecycle
      vaults.go                 ← Key Vault registration handlers
      secrets.go                ← Secret registration handlers
      jobs.go                   ← Job creation, listing
      rotations.go              ← Status reporting (worker)
      policies.go               ← Policy CRUD handlers
      audit.go                  ← Audit log query handler
      health.go                 ← /healthz and /readyz

    request/
      tenants.go                ← Request structs + validation for tenant endpoints
      members.go                ← Request structs for member endpoints
      workers.go                ← Request structs for worker endpoints
      vaults.go                 ← Request structs for vault endpoints
      secrets.go                ← Request structs for secret endpoints
      jobs.go                   ← Request structs for job endpoints
      policies.go               ← Request structs for policy endpoints

    response/
      envelope.go               ← Standard response envelope (data, meta, error)
      pagination.go             ← Pagination helpers

  auth/
    jwt.go                      ← JWT parsing + validation (Azure AD JWKS)
    claims.go                   ← Claim extraction helpers
    roles.go                    ← Role constants, hierarchy comparison

  authz/
    checker.go                  ← Authorization logic (Platform.Admin bypass, tenant_members check)
    permissions.go              ← Endpoint → required role mapping

  domain/
    tenant.go                   ← Tenant domain types
    member.go                   ← Membership domain types
    worker.go                   ← Worker domain types + status machine
    vault.go                    ← Key Vault registration types
    secret.go                   ← Secret registration types
    job.go                      ← Job types + status machine
    policy.go                   ← Policy types
    audit.go                    ← Audit log types

  store/
    postgres.go                 ← Connection pool, RLS session var helper
    tenants.go                  ← Tenant queries
    members.go                  ← Membership queries
    workers.go                  ← Worker queries
    vaults.go                   ← Key Vault queries
    secrets.go                  ← Secret registration queries
    jobs.go                     ← Job queries
    heartbeats.go               ← Heartbeat queries
    audit.go                    ← Audit log insert + query
    policies.go                 ← Policy queries

  audit/
    logger.go                   ← Audit log writer (wraps store + adds context)

  config/
    config.go                   ← Configuration loading (env vars, validation)
```

---

## Configuration

All configuration via environment variables (12-factor):

| Variable | Required | Description |
|----------|----------|-------------|
| `PORT` | no (default `8080`) | HTTP listen port |
| `DATABASE_URL` | yes | PostgreSQL connection string (Azure AD auth) |
| `AZURE_AD_TENANT_ID` | yes | Azure AD tenant for token validation |
| `AZURE_AD_APP_ID` | yes | Platform App Registration client ID (audience) |
| `APIM_MI_OID` | yes | APIM Managed Identity object ID (for gateway auth verification) |
| `JWKS_URL` | no (derived) | Azure AD JWKS endpoint (auto-derived from tenant ID) |
| `DB_MAX_OPEN_CONNS` | no (default `25`) | Connection pool size |
| `DB_MAX_IDLE_CONNS` | no (default `5`) | Idle connections |

| `LOG_LEVEL` | no (default `info`) | Structured log level |
| `LOG_FORMAT` | no (default `json`) | Log output format |

---

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Auth re-verification at API | Yes (defense in depth) | Even though APIM validates, API re-checks independently — no single point of trust |
| RLS session variable | Per-transaction | Set at start of each request, not per-connection — safe with connection pooling |
| One worker per tenant | No lease/competition | A tenant = an environment where the worker has private access or whitelisted. Scheduler publishes to topic, worker subscribes — no coordination needed |
| Tenant creation = Platform.Admin | Elevated privilege | Prevents sprawl — tenants are provisioned by platform admins, not self-service |
| Key Vault as explicit entity | `key_vaults` table + FK from secrets | Clear ownership model — admin registers vaults, grants worker RBAC externally, secrets reference vaults |
| Audit as middleware | Post-handler hook | Handler returns what happened, middleware writes the audit log — keeps handlers clean |
| 404 for unauthorized resources | Return 404 instead of 403 when resource exists but caller can't see it | Prevents resource enumeration (attacker can't distinguish "doesn't exist" from "no access") |
| Pagination | Offset-based (page/per_page) | Simple for MVP; cursor-based pagination can be added later for large datasets |
| Slug validation | Lowercase alphanumeric + hyphens, 3-100 chars | URL-safe, human-readable, no special chars |
