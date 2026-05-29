# Rotation Policies — Detailed Design

## Overview

A **rotation policy** is a reusable configuration template that governs how secrets under its control are rotated. It defines the schedule, retry behavior, validation stance, and notification preferences — everything except *what* to rotate (that's the secret registration) and *how* to rotate it (that's the plugin).

Policies are **tenant-scoped**: each tenant creates and manages their own. Multiple secrets can share one policy, and changing the policy cascades to all secrets using it on their next rotation cycle.

---

## Why Policies Are Separate From Secrets

| If policy fields lived on secret_registrations... | With a separate policy table... |
|---|---|
| Every secret duplicates retry/interval/validation config | One policy, N secrets — DRY |
| Changing "all prod secrets rotate every 30 days" means N updates | Change one policy row, all secrets pick it up |
| No way to enforce organizational standards | Tenant admin creates approved policies, operators assign them |
| Hard to report "which secrets use aggressive vs relaxed settings" | Query by policy_id |

Policies are the control lever for tenant admins. Secrets are the targets. The scheduler evaluates both together at runtime.

---

## Policy Schema

From [DATABASE.md](DATABASE.md) — the `rotation_policies` table:

| Column | Type | Default | What It Controls |
|--------|------|---------|-----------------|
| `rotation_interval` | `INTERVAL` | — (required) | How often the scheduler creates rotation jobs |
| `secret_lifetime` | `INTERVAL` | — (required) | How long the new credential should be valid when created |
| `max_retry_attempts` | `INTEGER` | `3` | How many times a failed job is retried before dead-letter |
| `retry_backoff_base` | `INTERVAL` | `5 minutes` | Base interval for exponential backoff between retries |
| `validation_required` | `BOOLEAN` | `true` | Whether the worker must validate the new credential post-rotation |
| `notification_on_failure` | `BOOLEAN` | `true` | Notify tenant contacts when a rotation fails |
| `notification_on_success` | `BOOLEAN` | `false` | Notify tenant contacts on success (noisy, usually off) |
| `metadata` | `JSONB` | `{}` | Freeform: tags, compliance labels, custom config |

### Field Details

#### `rotation_interval`

The core field. Drives the `next_rotation_at` computation on secrets:

```
secret.next_rotation_at = NOW() + policy.rotation_interval
```

This is evaluated:
- On secret creation (initial schedule)
- On successful rotation completion (reset the timer)
- On policy update if interval changes (optionally recompute — see "Policy Updates" below)

**Valid values**: Any PostgreSQL interval — `7 days`, `30 days`, `90 days`, `12 hours`, `6 hours`. The API accepts shorthand (`7d`, `30d`, `1h`) and normalizes to interval format.

**Minimum**: Enforced in API validation. Recommended floor: `1 hour` (prevents accidental tight loops). Configurable per deployment.

**Maximum**: No hard max, but compliance may require ceiling (e.g., "no policy > 90 days").

---

#### `secret_lifetime`

How long the newly created credential should be valid. Sent to the worker in the Service Bus message so the plugin knows what expiry to set when creating the new credential.

```
Example: App Registration client secret
  secret_lifetime = 55 days
  rotation_interval = 45 days
  buffer = 55 - 45 = 10 days
```

The worker passes this value to the target resource API when creating the credential (e.g., `endDateTime` on a Microsoft Graph `addPassword` call, or `VALID UNTIL` on a PostgreSQL `ALTER ROLE`).

**Relationship to `rotation_interval`**: The `secret_lifetime` should always be **greater than** `rotation_interval`. The difference is your safety buffer — the overlap period where both old and new credentials are valid (for dual-credential resources), or the grace period for retry if rotation fails (for single-credential resources).

**API validation**: The API enforces `secret_lifetime > rotation_interval` on create and update. If violated → `400 Bad Request` with a message explaining the buffer requirement.

```
secret_lifetime - rotation_interval = buffer

Examples:
  55 days - 45 days = 10-day buffer  ✓
  30 days - 30 days = 0-day buffer   ✗ (rejected — no safety margin)
  20 days - 30 days = negative       ✗ (rejected — secret expires before rotation)
```

---

#### `max_retry_attempts`

Controls the threshold at which the Dead-Letter Promoter gives up:

```
If job.attempt_number >= policy.max_retry_attempts → DEAD_LETTERED
If job.attempt_number <  policy.max_retry_attempts → eligible for retry
```

When the scheduler creates a job, it copies this value into `rotation_jobs.max_attempts` — so the job carries the retry budget with it even if the policy changes later.

**Why copy into the job?** A policy update mid-flight shouldn't change the retry count for an already-running job. The job was created under the old rules; it finishes under those rules.

---

#### `retry_backoff_base`

The Retry Promoter uses this for exponential backoff:

```
wait_time = retry_backoff_base × 2^(attempt_number - 1)

Example with base = 5 minutes:
  Attempt 1 fails → wait 5 min  → retry (attempt 2)
  Attempt 2 fails → wait 10 min → retry (attempt 3)
  Attempt 3 fails → wait 20 min → dead-letter (max reached)
```

**Design choice**: The base comes from the policy (via the job's associated `policy_id`), not copied into the job. The Retry Promoter joins to `rotation_policies` at evaluation time. This means if the admin increases backoff on the policy, it takes effect for all FAILED jobs waiting to retry — which is desirable (maybe they increased it because retries were too aggressive).

---

#### `validation_required`

Sent to the worker in the Service Bus message body:

```json
{
  "policy": {
    "validation_required": true
  }
}
```

If `true`: Worker must verify the new credential works (e.g., connect to PostgreSQL with the new password, call Microsoft Graph with the new client secret). If validation fails → job reports FAILED.

If `false`: Worker rotates and reports COMPLETED immediately. Useful for dev/test secrets where speed matters more than confidence.

---

#### `notification_on_failure` / `notification_on_success`

Controls whether the platform sends alerts (future: email, webhook, Teams channel) when:
- A rotation hits DEAD_LETTERED (failure notification)
- A rotation reaches COMPLETED (success notification)

Both are per-policy — some policies are "critical" (always notify on failure), some are "noisy low-priority" (suppress).

**MVP**: Notification delivery is a future feature. The flags are stored now so policies are forward-compatible. In the meantime, observability dashboards surface failures.

---

## Policy Assignment

### One Policy Per Secret

Each `secret_registrations` row has a `policy_id` FK:

```
secret_registrations.policy_id → rotation_policies.id
```

A secret can have `policy_id = NULL` — this means **no automated rotation**. The secret is registered (tracked, can be manually rotated) but the scheduler will never auto-create jobs for it because `next_rotation_at` will be NULL.

### Assignment Flow

```
1. Tenant admin creates policy "prod-30-day"
   POST /api/tenants/{tid}/policies
   { "name": "prod-30-day", "rotation_interval": "30d" }

2. Tenant admin or operator registers a secret with the policy
   POST /api/tenants/{tid}/secrets
   { "name": "prod-db-password", "policy_id": "<policy-uuid>", ... }
   → API sets next_rotation_at = NOW() + 30 days

3. Or assigns a policy to an existing unscheduled secret
   PATCH /api/tenants/{tid}/secrets/{sid}
   { "policy_id": "<policy-uuid>" }
   → API sets next_rotation_at = NOW() + 30 days
```

### Reassignment

A secret can be moved from one policy to another:

```
PATCH /api/tenants/{tid}/secrets/{sid}
{ "policy_id": "<new-policy-uuid>" }
```

**What happens**:
1. `policy_id` updated to new policy
2. `next_rotation_at` recomputed: `NOW() + new_policy.rotation_interval`
3. If there's an in-flight job (PENDING/PUBLISHED/RUNNING) — it continues under the old max_attempts (already copied). No disruption.
4. Audit log: `secret.policy_changed`

### Unassignment

Setting `policy_id = NULL` pauses automated rotation:

```
PATCH /api/tenants/{tid}/secrets/{sid}
{ "policy_id": null }
```

- `next_rotation_at` set to NULL
- Scheduler ignores this secret (no `next_rotation_at` to evaluate)
- Manual rotation still works (`POST /api/tenants/{tid}/secrets/{sid}/rotate`)

---

## Policy Evaluation in the Scheduler

The scheduler's Job Creation loop directly uses the policy:

```sql
SELECT sr.*, rp.rotation_interval, rp.max_retry_attempts, rp.validation_required
FROM secret_registrations sr
JOIN rotation_policies rp ON sr.policy_id = rp.id
WHERE sr.status = 'ACTIVE'
  AND sr.policy_id IS NOT NULL           -- has an assigned policy
  AND sr.next_rotation_at <= NOW()       -- rotation is due
  AND NOT EXISTS (...)                    -- no active job already
```

When creating the job row:
- `rotation_jobs.max_attempts` = `rp.max_retry_attempts` (snapshot at creation)
- `rotation_jobs.policy_id` = `rp.id` (reference for retry promoter backoff lookup)
- `rotation_jobs.scheduled_at` = `NOW()`

The policy's `rotation_interval` itself isn't needed at job creation time — it was already used when computing `next_rotation_at` on the secret. By the time the scheduler runs, it just checks `next_rotation_at <= NOW()`.

---

## Policy Updates

When a tenant admin updates a policy (`PATCH /api/tenants/{tid}/policies/{pid}`), the question is: **do existing secrets get their `next_rotation_at` recomputed?**

### Design Decision: No Automatic Recomputation

Updating a policy does NOT automatically recompute `next_rotation_at` on all secrets using it.

**Rationale**:
- If admin changes interval from 30 days to 7 days, mass-recomputing could create a thundering herd (dozens of secrets all suddenly "due")
- The next natural rotation cycle will use the new interval to reset `next_rotation_at`
- Admin can manually trigger rotation for urgent secrets

**How it plays out**:

```
Secret was: next_rotation_at = June 15 (based on old 30-day policy)
Admin changes policy to 7 days on May 20

- Secret still scheduled for June 15 (no recompute)
- June 15: job runs, succeeds
- API resets: next_rotation_at = June 15 + 7 days = June 22
- From now on, 7-day cadence
```

If the admin wants immediate effect, they can:
1. Manually trigger rotation on critical secrets
2. Or use a bulk API (future) to recompute `next_rotation_at` for all secrets on a policy

### What DOES Update Immediately

| Field Changed | Effect On Existing Secrets | Effect On In-Flight Jobs |
|---------------|---------------------------|--------------------------|
| `rotation_interval` | Next cycle uses new interval | None (current job finishes normally) |
| `max_retry_attempts` | Next job created gets new max | None (current job has snapshot) |
| `retry_backoff_base` | Immediate — Retry Promoter reads live | Retry timing changes for FAILED jobs |
| `validation_required` | Next job's message carries new value | None (current job already received message) |
| `notification_on_failure` | Immediate for next failure event | — |
| `notification_on_success` | Immediate for next success event | — |

---

## Policy Deletion

A policy cannot be deleted while secrets reference it:

```
DELETE /api/tenants/{tid}/policies/{pid}

→ Check: SELECT count(*) FROM secret_registrations WHERE policy_id = {pid} AND status = 'ACTIVE'
→ If > 0: 409 Conflict { "error": "policy_in_use", "count": N }
→ If = 0: Delete + 204 No Content
```

The admin must first reassign or unassign all secrets using the policy.

This prevents orphaned secrets with a dangling `policy_id` FK.

---

## Common Policy Patterns

### Standard Rotation Tiers

Most tenants will create 2–4 policies that cover their needs:

| Policy Name | Interval | Lifetime | Retries | Backoff Base | Validation | Use Case |
|-------------|----------|----------|---------|--------------|------------|----------|
| `critical-7d` | 7 days | 14 days | 5 | 2 min | ✓ | High-security: payment credentials, admin secrets |
| `prod-30d` | 30 days | 45 days | 3 | 5 min | ✓ | Standard production secrets |
| `prod-45d` | 45 days | 55 days | 3 | 5 min | ✓ | Longer-lived resources (10-day buffer) |
| `dev-7d` | 7 days | 14 days | 2 | 1 min | ✗ | Dev/test — fast, no validation overhead |
| `compliance-90d` | 90 days | 120 days | 3 | 10 min | ✓ | Externally mandated rotation (SOC2, PCI) |

### Overlap Rotation (Dual-Credential Resources)

For resources that support two simultaneous credentials (e.g., Azure AD app registrations, storage account keys):

```
Policy secret_lifetime: 55 days
Policy rotation_interval: 45 days
Buffer: 55 - 45 = 10 days of overlap

Day 0:  v1 created (worker sets expiry = 55 days, expires day 55)
Day 45: v2 created (worker sets expiry = 55 days, expires day 100), v1 still valid 10 more days
Day 90: v3 created (worker sets expiry = 55 days, expires day 145), v2 still valid 10 more days
```

The policy sets both `rotation_interval = 45 days` and `secret_lifetime = 55 days`. The worker uses `secret_lifetime` to set the expiry on the new credential. The **plugin** handles the overlap mechanics (adds new credential, doesn't remove old one — it expires naturally at the lifetime boundary).

### Atomic Swap (Single-Credential Resources)

For resources with only one valid credential (e.g., PostgreSQL password):

```
Policy rotation_interval: 30 days
validation_required: true  ← critical for atomic swap

Day 0:  password set
Day 30: new password generated → applied to PostgreSQL → written to Key Vault → validated
         (consumers must pick up new value from Key Vault immediately)
```

Here, `validation_required = true` is essential — the worker must verify the new password works before reporting success, because the old password is already dead.

---

## Policy & Secrets Relationship Diagram

```
┌────────────────────────────────────────────────────────────────┐
│ Tenant: team-platform-prod                                      │
│                                                                  │
│  ┌──────────────────────────┐                                   │
│  │ Policy: prod-30d          │                                   │
│  │  interval: 30 days        │                                   │
│  │  retries: 3               │                                   │
│  │  validation: true         │◄───────┐                         │
│  └──────────────────────────┘        │                          │
│                                       │ policy_id                │
│  ┌──────────────────────────┐        │                          │
│  │ Policy: critical-7d       │        │                          │
│  │  interval: 7 days         │◄──┐    │                          │
│  │  retries: 5               │   │    │                          │
│  │  validation: true         │   │    │                          │
│  └──────────────────────────┘   │    │                          │
│                                  │    │                          │
│  ┌────────────────────┐         │    │   ┌────────────────────┐ │
│  │ Secret: payment-api │─────────┘    │   │ Secret: prod-db-pw │──┘
│  │ next_rot: May 27    │             │   │ next_rot: June 19   │ │
│  └────────────────────┘              │   └────────────────────┘ │
│                                       │                          │
│  ┌────────────────────┐              │                          │
│  │ Secret: cache-redis │──────────────┘                          │
│  │ next_rot: June 19   │                                        │
│  └────────────────────┘                                         │
│                                                                  │
│  ┌────────────────────┐                                         │
│  │ Secret: dev-token   │──── policy_id: NULL (manual only)       │
│  │ next_rot: NULL      │                                        │
│  └────────────────────┘                                         │
└────────────────────────────────────────────────────────────────┘
```

---

## Compliance Reporting

Policies enable compliance-oriented queries without understanding individual secrets:

### Overdue Secrets

```sql
-- Secrets that should have rotated but haven't (job might be stuck or worker down)
SELECT sr.name, sr.next_rotation_at, rp.name AS policy_name,
       NOW() - sr.next_rotation_at AS overdue_by
FROM secret_registrations sr
JOIN rotation_policies rp ON sr.policy_id = rp.id
WHERE sr.status = 'ACTIVE'
  AND sr.next_rotation_at < NOW()
ORDER BY overdue_by DESC;
```

### Rotation Coverage

```sql
-- Secrets without a policy (not automatically rotated)
SELECT sr.name, sr.tenant_id
FROM secret_registrations sr
WHERE sr.policy_id IS NULL
  AND sr.status = 'ACTIVE';
```

### Policy Distribution

```sql
-- How many secrets per policy (helps identify over-used or unused policies)
SELECT rp.name, rp.rotation_interval, count(sr.id) AS secret_count
FROM rotation_policies rp
LEFT JOIN secret_registrations sr ON sr.policy_id = rp.id AND sr.status = 'ACTIVE'
WHERE rp.tenant_id = :tenant_id
GROUP BY rp.id
ORDER BY secret_count DESC;
```

### Compliance Summary

```sql
-- Per-tenant: are all secrets within their rotation window?
SELECT
  t.name AS tenant_name,
  count(*) FILTER (WHERE sr.next_rotation_at > NOW()) AS on_schedule,
  count(*) FILTER (WHERE sr.next_rotation_at <= NOW()) AS overdue,
  count(*) FILTER (WHERE sr.policy_id IS NULL) AS unmanaged
FROM secret_registrations sr
JOIN tenants t ON sr.tenant_id = t.id
WHERE sr.status = 'ACTIVE'
GROUP BY t.id;
```

These queries power the tenant dashboard (future) and can be exposed via API endpoints for compliance tooling.

---

## API Endpoints

From [API_SERVICE.md](API_SERVICE.md):

| Method | Path | Min Role | Description |
|--------|------|----------|-------------|
| GET | `/api/tenants/{tid}/policies` | `operator` | List all policies for the tenant |
| GET | `/api/tenants/{tid}/policies/{pid}` | `operator` | Get single policy with secret count |
| POST | `/api/tenants/{tid}/policies` | `admin` | Create a new policy |
| PATCH | `/api/tenants/{tid}/policies/{pid}` | `admin` | Update policy fields |
| DELETE | `/api/tenants/{tid}/policies/{pid}` | `admin` | Delete (if no secrets reference it) |

### Create Policy — Request/Response

**Request**:
```json
POST /api/tenants/{tid}/policies
{
  "name": "prod-30d",
  "rotation_interval": "30d",
  "secret_lifetime": "45d",
  "max_retry_attempts": 3,
  "retry_backoff_base": "5m",
  "validation_required": true,
  "notification_on_failure": true,
  "notification_on_success": false,
  "metadata": {
    "compliance": "SOC2",
    "team": "platform"
  }
}
```

**Response** (`201 Created`):
```json
{
  "data": {
    "id": "a1b2c3d4-...",
    "tenant_id": "t1e2n3...",
    "name": "prod-30d",
    "rotation_interval": "30 days",
    "secret_lifetime": "45 days",
    "max_retry_attempts": 3,
    "retry_backoff_base": "5 minutes",
    "validation_required": true,
    "notification_on_failure": true,
    "notification_on_success": false,
    "metadata": { "compliance": "SOC2", "team": "platform" },
    "created_at": "2026-05-20T14:30:00Z",
    "updated_at": "2026-05-20T14:30:00Z"
  }
}
```

### Update Policy — Partial Update

```json
PATCH /api/tenants/{tid}/policies/{pid}
{
  "rotation_interval": "14d",
  "secret_lifetime": "21d",
  "max_retry_attempts": 5
}
```

Only fields included in the body are updated. Omitted fields remain unchanged.

### List Policies — With Secret Count

```json
GET /api/tenants/{tid}/policies

{
  "data": [
    {
      "id": "a1b2c3d4-...",
      "name": "prod-30d",
      "rotation_interval": "30 days",
      "max_retry_attempts": 3,
      "validation_required": true,
      "secret_count": 12,
      "created_at": "2026-05-20T14:30:00Z"
    },
    {
      "id": "b2c3d4e5-...",
      "name": "critical-7d",
      "rotation_interval": "7 days",
      "max_retry_attempts": 5,
      "validation_required": true,
      "secret_count": 3,
      "created_at": "2026-05-18T09:00:00Z"
    }
  ],
  "meta": { "total": 2 }
}
```

The `secret_count` field is computed at query time — helps admins see which policies are in use.

---

## Validation Rules

The API enforces these constraints on policy creation and update:

| Field | Rule | Error |
|-------|------|-------|
| `name` | Required, 1–255 chars, unique within tenant | `400` / `409 Conflict` |
| `rotation_interval` | Required, minimum `1h`, parseable as duration | `400` |
| `secret_lifetime` | Required, must be > `rotation_interval`, parseable as duration | `400` |
| `max_retry_attempts` | 1–10 range | `400` |
| `retry_backoff_base` | Minimum `30s`, parseable as duration | `400` |
| `validation_required` | Boolean | `400` |

### Interval Parsing

The API accepts multiple formats and normalizes:

| Input | Stored As | Notes |
|-------|-----------|-------|
| `"7d"` | `7 days` | Days shorthand |
| `"30d"` | `30 days` | |
| `"12h"` | `12:00:00` | Hours shorthand |
| `"90d"` | `90 days` | |
| `"1h"` | `01:00:00` | Minimum useful interval |
| `"30 days"` | `30 days` | Already PostgreSQL format |

---

## Global Default Policy

Every tenant gets a **global default policy** seeded on tenant creation. This provides sensible defaults so tenants can start registering secrets immediately without first creating a custom policy.

### Default Values

| Field | Value | Rationale |
|-------|-------|----------|
| `name` | `default` | Well-known name, always exists |
| `rotation_interval` | `80 days` | Rotate well before the 90-day lifetime expires |
| `secret_lifetime` | `90 days` | Industry-standard credential lifetime (Azure AD default) |
| `max_retry_attempts` | `3` | Standard retry budget |
| `retry_backoff_base` | `5 minutes` | Balanced — not too aggressive, not too slow |
| `validation_required` | `true` | Safety first |
| `notification_on_failure` | `true` | Always know when rotation fails |
| `notification_on_success` | `false` | Not noisy by default |

**Buffer**: 90 - 80 = **10 days** of safety margin. If rotation fails on day 80 and retries take time, there's still a 10-day window before the credential expires.

### How It's Created

When a tenant is created (`POST /api/tenants`), the API auto-inserts the default policy:

```sql
INSERT INTO rotation_policies (
  tenant_id, name, rotation_interval, secret_lifetime,
  max_retry_attempts, retry_backoff_base,
  validation_required, notification_on_failure, notification_on_success
) VALUES (
  :new_tenant_id, 'default', '80 days', '90 days',
  3, '5 minutes',
  true, true, false
);
```

### Usage

- If a secret is registered without a `policy_id`, it gets assigned the tenant's `default` policy automatically
- The API resolves this by looking up `rotation_policies WHERE tenant_id = :tid AND name = 'default'`
- Tenant admins can update the default policy's values (e.g., change interval to 60 days) — it's a normal policy row, just pre-created
- Tenant admins **cannot delete** the default policy (API returns `403` — must always exist)
- Tenants can create additional custom policies for secrets that need different schedules

### Why 90/80?

- **90-day lifetime** aligns with Azure AD's default app registration credential expiry and many compliance frameworks (SOC2, CIS benchmarks recommend ≤ 90 days)
- **80-day rotation** gives a 10-day buffer — enough headroom for retries and weekends
- Tenants who need tighter rotation create custom policies (`critical-7d`, `prod-30d`, etc.)
- Tenants whose resources have shorter lifetimes override with custom policies

### Interaction with CLI

A future CLI tool can trigger rotations on any secret without knowing its policy:

```bash
# Rotates immediately — policy controls lifetime of the new credential
az-rotator rotate --tenant team-platform-prod --secret prod-db-password

# List policies for a tenant
az-rotator policies list --tenant team-platform-prod

# Create a tighter policy
az-rotator policies create --tenant team-platform-prod \
  --name critical-7d --interval 7d --lifetime 14d --retries 5
```

The control plane APIs are designed to be CLI-friendly — RESTful, predictable, stateless.

---

## Policy Lifecycle Summary

```
┌─────────────┐     ┌───────────────────┐     ┌──────────────────┐
│ Admin creates│────▶│ Secrets assigned  │────▶│ Scheduler uses   │
│ policy       │     │ (policy_id FK)     │     │ interval + retry │
└─────────────┘     └───────────────────┘     └──────────────────┘
                            │                          │
                            │                          │
                    ┌───────▼──────────┐      ┌───────▼──────────┐
                    │ next_rotation_at │      │ Job inherits     │
                    │ = NOW() + interval│      │ max_attempts     │
                    └──────────────────┘      └──────────────────┘
                                                       │
                                               ┌───────▼──────────┐
                                               │ On COMPLETED:    │
                                               │ reset next_rot   │
                                               │ = NOW() + interval│
                                               └──────────────────┘
```

The policy is a **living configuration** — changes to retry/notification settings take effect immediately for future evaluation. The `rotation_interval` takes effect on the next successful rotation (when `next_rotation_at` is recomputed). The `max_retry_attempts` is snapshotted into the job at creation time for consistency.
