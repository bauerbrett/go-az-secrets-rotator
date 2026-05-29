# Service Bus — Detailed Design

## Overview

Azure Service Bus is the **job distribution mechanism** between the control plane (scheduler) and workers. The scheduler publishes rotation job messages to a tenant's topic; the tenant's worker subscribes and processes them.

**Namespace**: One shared Service Bus namespace (Standard tier) for all tenants.

**Model**: Topic-per-tenant with a single subscription per topic (one worker per tenant).

---

## Why Service Bus (Not Direct API Polling)

| Concern | Direct Polling | Service Bus |
|---------|---------------|-------------|
| Worker coupling | Worker must poll API repeatedly | Worker subscribes, messages pushed |
| Offline handling | Missed jobs if worker is down | Messages queue until worker returns |
| Retry | Platform must re-push manually | Built-in retry + dead-letter |
| Ordering | Custom implementation | Session-based ordering available |
| Backpressure | Worker overwhelmed if many jobs | Worker controls receive rate (prefetch) |
| Auth | Worker needs API access constantly | Worker needs only Service Bus Receiver role |

The key advantage: if a worker is restarting, scaling, or temporarily offline, messages accumulate safely in the subscription. When the worker comes back, it drains the queue — no jobs lost, no coordination needed.

---

## Topology

```
                        ┌─────────────────────────────────────────┐
                        │   Service Bus Namespace                  │
                        │   (az-rotator-servicebus)                │
                        │                                         │
                        │  ┌──────────────────────────────────┐   │
                        │  │ Topic: tenant-{slug}-rotations    │   │
                        │  │                                    │   │
 Scheduler ──publish──▶ │  │  ┌────────────────────────────┐   │   │
                        │  │  │ Subscription: worker        │   │   │  ◀──receive── Worker
                        │  │  │  (max delivery: 3)          │   │   │
                        │  │  │  (lock duration: 5 min)     │   │   │
                        │  │  │  (TTL: 1 hour)              │   │   │
                        │  │  └────────────────────────────┘   │   │
                        │  │                                    │   │
                        │  │  ┌────────────────────────────┐   │   │
                        │  │  │ Dead-Letter Queue (DLQ)     │   │   │
                        │  │  │  (failed after max delivery)│   │   │
                        │  │  └────────────────────────────┘   │   │
                        │  └──────────────────────────────────┘   │
                        │                                         │
                        │  ┌──────────────────────────────────┐   │
                        │  │ Topic: tenant-{slug}-rotations    │   │  (one per tenant)
                        │  │  ...                              │   │
                        │  └──────────────────────────────────┘   │
                        └─────────────────────────────────────────┘
```

### Naming Convention

| Resource | Name Pattern | Example |
|----------|-------------|---------|
| Namespace | `az-rotator-servicebus` | `az-rotator-servicebus` |
| Topic | `tenant-{slug}-rotations` | `tenant-team-platform-prod-rotations` |
| Subscription | `worker` | `worker` (always the same — one per topic) |

Topics are created **on tenant creation** and deleted (or disabled) **on tenant deactivation**.

Subscriptions are created **on worker approval** and deleted **on worker deactivation**.

---

## Message Schema

Every message published to a tenant's topic represents a single rotation job.

### Message Properties (Broker Properties)

| Property | Value | Purpose |
|----------|-------|---------|
| `MessageId` | Job UUID (`rotation_jobs.id`) | Deduplication — same job won't be processed twice |
| `ContentType` | `application/json` | Message body format |
| `Subject` | `rotation-job` | Message type discriminator (future: other message types) |
| `TimeToLive` | 1 hour (configurable) | Message expires if not picked up — system task recycles the job |
| `SessionId` | *(not used)* | No ordering needed — each job is independent |

### Custom Properties (Application Properties)

| Property | Type | Value |
|----------|------|-------|
| `tenant_id` | string | Platform tenant UUID |
| `secret_id` | string | Secret registration UUID |
| `job_id` | string | Rotation job UUID |
| `secret_type` | string | Plugin type (e.g. `postgresql`, `app_registration`) |
| `attempt_number` | int | Which attempt this is (1, 2, 3...) |
| `priority` | int | Job priority (higher = more urgent) |

These properties allow the worker to inspect the message metadata **without deserializing the body** — useful for filtering, logging, and routing decisions.

### Message Body

```
{
  "job_id": "uuid",
  "tenant_id": "uuid",
  "secret": {
    "id": "uuid",
    "name": "prod-db-password",
    "secret_type": "postgresql",
    "resource_id": "/subscriptions/.../resourceGroups/.../providers/Microsoft.DBforPostgreSQL/flexibleServers/prod-db",
    "config": {
      "host": "prod-db.postgres.database.azure.com",
      "port": 5432,
      "username": "app_user",
      "database": "appdb"
    }
  },
  "vault": {
    "id": "uuid",
    "vault_uri": "https://prod-secrets.vault.azure.net/",
    "secret_name": "prod-db-password"
  },
  "policy": {
    "validation_required": true,
    "max_retry_attempts": 3,
    "secret_lifetime": "55 days"
  },
  "attempt_number": 1,
  "scheduled_at": "2026-05-18T10:00:00Z",
  "published_at": "2026-05-18T10:00:05Z"
}
```

**Design principle**: The message body contains **everything the worker needs** to execute the rotation without calling back to the API for job details. This minimizes round trips and allows the worker to operate even if the API is briefly unreachable.

### What's Included

| Field | Why |
|-------|-----|
| `secret.resource_id` | Worker knows which resource to rotate |
| `secret.secret_type` | Worker selects the correct rotation plugin |
| `secret.config` | Plugin-specific connection/config details |
| `vault.vault_uri` | Worker knows where to store the new secret |
| `vault.secret_name` | Worker knows the secret name to write in Key Vault |
| `policy.validation_required` | Worker knows if it must validate post-rotation |
| `attempt_number` | Worker can log/alert differently on retries |

### What's NOT Included

- Actual secret values (never in messages)
- Worker credentials (worker uses its own MI)
- Tenant member info (irrelevant to rotation)

---

## Lifecycle Events

### Topic Created — On Tenant Creation

When a `Platform.Admin` creates a new tenant (`POST /api/tenants`), the API service creates the Service Bus topic:

1. API creates tenant record in PostgreSQL
2. API calls Service Bus management SDK → create topic `tenant-{slug}-rotations`
3. Topic settings: `default_message_ttl = 1h`, `max_size = 1GB`, `enable_partitioning = false`

If topic creation fails, the tenant creation is rolled back (or retried).

### Subscription Created — On Worker Approval

When an admin approves a worker (`POST /api/tenants/{tid}/workers/{wid}/approve`), the API service creates the subscription:

1. API updates worker status → `APPROVED`
2. API calls Service Bus management SDK → create subscription `worker` on topic `tenant-{slug}-rotations`
3. Subscription settings:
   - `max_delivery_count = 3` — after 3 failed deliveries, message goes to DLQ
   - `lock_duration = PT5M` — worker has 5 minutes to process before lock expires
   - `default_message_ttl = PT1H` — inherited from topic
   - `dead_lettering_on_message_expiration = true` — expired messages go to DLQ
   - `enable_dead_lettering_on_filter_evaluation_exceptions = true`

### Subscription Deleted — On Worker Deactivation

When a worker is deactivated (`DELETE /api/tenants/{tid}/workers/{wid}`):

1. API updates worker status → `DEACTIVATED`
2. API calls Service Bus management SDK → delete subscription `worker`
3. Any unprocessed messages in the subscription are lost (expected — tenant is decommissioning the worker)

### Topic Deleted — On Tenant Deactivation

When a tenant is deactivated (`DELETE /api/tenants/{tid}`):

1. API updates tenant status → `DEACTIVATED`
2. API calls Service Bus management SDK → delete topic `tenant-{slug}-rotations`
3. All messages and DLQ entries for this tenant are permanently removed

---

## Publishing (Scheduler → Service Bus)

The scheduler is the sole publisher. It runs on a polling loop:

### Scheduler Publish Loop

```
Every 30 seconds (configurable):
    │
    ▼
┌─────────────────────────────────────────────────┐
│ 1. Query: rotation_jobs WHERE status = 'PENDING'│
│    AND scheduled_at <= NOW()                    │
│    ORDER BY priority DESC, scheduled_at ASC     │
│    LIMIT 100                                    │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│ 2. For each job:                                │
│    a. Load secret + vault + policy details      │
│    b. Build message body + properties           │
│    c. Resolve topic name from tenant slug       │
│    d. Publish message to topic                  │
│    e. UPDATE job: status → PUBLISHED,           │
│       published_at = NOW()                      │
└─────────────────────────────────────────────────┘
```

### Publish Guarantees

- **At-least-once**: If the scheduler crashes after publishing but before updating the DB, the job stays PENDING and will be re-published on next cycle. The `MessageId = job UUID` ensures the Service Bus deduplicates within its dedup window.
- **Deduplication window**: 10 minutes (Service Bus topic-level setting). Same `MessageId` within this window is silently dropped.
- **Batch publishing**: The scheduler can batch messages to the same topic for efficiency (up to 256KB per batch or configurable message count).

### Publish Failure Handling

If publishing fails (Service Bus unreachable, auth error, topic doesn't exist):

- Job stays in `PENDING` — scheduler retries on next polling cycle
- Scheduler logs the error with job ID and tenant ID
- Alert fires if publish failures exceed threshold (observability)

---

## Receiving (Worker ← Service Bus)

The worker subscribes to its tenant's topic and processes messages one at a time (or with configurable concurrency).

### Worker Receive Loop

```
Worker starts (after registration + approval):
    │
    ▼
┌─────────────────────────────────────────────────┐
│ 1. Connect to Service Bus subscription          │
│    Topic: tenant-{slug}-rotations               │
│    Subscription: worker                         │
│    Auth: Managed Identity                       │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│ 2. Receive message (blocking, with timeout)     │
│    - PeekLock mode (message invisible to others)│
│    - Lock duration: 5 minutes                   │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│ 3. Report RUNNING to API                        │
│    POST /api/rotations/{jobId}/status           │
│    Body: { "status": "RUNNING" }                │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│ 4. Execute rotation plugin                      │
│    - Generate new secret value                  │
│    - Apply to target resource (resource_id)     │
│    - Write new version to Key Vault             │
│      (vault_uri + secret_name)                  │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│ 5. Validate (if policy.validation_required)     │
│    - Report VALIDATING to API                   │
│    - Test connectivity with new credential      │
│    - If validation fails → report FAILED        │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│ 6. Report completion to API                     │
│    POST /api/rotations/{jobId}/status           │
│    Body: { "status": "COMPLETED", "result": {}} │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│ 7. Complete message (remove from subscription)  │
│    servicebus.CompleteMessage(msg)              │
└─────────────────────────────────────────────────┘
```

### Message Settlement

| Outcome | Action | Effect |
|---------|--------|--------|
| Rotation succeeded | `CompleteMessage` | Message removed from subscription |
| Rotation failed (recoverable) | `AbandonMessage` | Message returns to subscription, delivery count incremented |
| Rotation failed (permanent) | `DeadLetterMessage` with reason | Message moves to DLQ immediately |
| Lock expired (worker too slow or crashed) | Automatic | Message becomes visible again, delivery count incremented |

### PeekLock vs ReceiveAndDelete

We use **PeekLock** mode:
- Message is invisible while the worker processes it
- If worker completes successfully → message removed
- If worker crashes → lock expires → message reappears
- This gives at-least-once delivery without losing messages

---

## Dead-Letter Queue (DLQ)

Each subscription has an automatic DLQ. Messages land here when:

1. **Max delivery count exceeded** — message delivered 3 times, each time abandoned or lock expired
2. **Message expired** — TTL elapsed without being picked up (1 hour default)
3. **Explicitly dead-lettered** — worker calls `DeadLetterMessage` for permanent failures
4. **Filter evaluation exception** — subscription rule error (shouldn't happen with our simple setup)

### DLQ Handling

The **System Tasks** CronJob periodically checks DLQs:

```
Every 5 minutes:
    │
    ▼
┌─────────────────────────────────────────────────┐
│ 1. For each active tenant:                      │
│    a. Peek DLQ messages (non-destructive)       │
│    b. For each DLQ message:                     │
│       - Read job_id from properties             │
│       - Update rotation_jobs.status →           │
│         DEAD_LETTERED (if not already)          │
│       - Log dead-letter reason                  │
│       - Complete the DLQ message (remove it)    │
│    c. Alert if DLQ depth > threshold            │
└─────────────────────────────────────────────────┘
```

This ensures the platform DB state stays in sync with Service Bus state. The audit log captures why the job was dead-lettered.

---

## Timeout & Reconciliation

Since there's no lease table, the system task detects stale jobs by time:

| Scenario | Detection | Resolution |
|----------|-----------|------------|
| Job stuck in PUBLISHED (worker offline) | `published_at < NOW() - 1 hour` | System task resets to PENDING (re-publish on next scheduler cycle) |
| Worker crashed mid-processing | Worker stops heartbeating, job stays RUNNING | System task detects: `started_at < NOW() - 30 min` AND status = RUNNING → mark FAILED |
| Message expired in Service Bus | DLQ receives it with `TTLExpired` reason | DLQ handler marks job DEAD_LETTERED or FAILED |

The timeout thresholds are configurable and should be >= the message TTL to avoid conflicts between Service Bus expiry and system task reconciliation.

---

## Authentication & RBAC

All Service Bus access uses **Managed Identity** — no connection strings or SAS keys.

### Role Assignments

| Principal | Role | Scope | Purpose |
|-----------|------|-------|---------|
| Scheduler (AKS Workload Identity) | `Azure Service Bus Data Sender` | Namespace | Publish messages to any topic |
| API Service (AKS Workload Identity) | `Azure Service Bus Data Owner` | Namespace | Create/delete topics and subscriptions |
| Worker (Container App MI) | `Azure Service Bus Data Receiver` | Specific topic | Receive messages from its tenant's topic only |
| System Tasks (AKS Workload Identity) | `Azure Service Bus Data Receiver` | Namespace | Read DLQ messages for reconciliation |

### Worker Scope Restriction

Workers receive **topic-scoped** RBAC — not namespace-wide. When a worker is approved:

1. API assigns `Azure Service Bus Data Receiver` to the worker's MI, scoped to the specific topic (`tenant-{slug}-rotations`)
2. Worker cannot read messages from other tenants' topics — Azure RBAC enforces this at the infrastructure level

This is a **hard isolation boundary** — even if a worker is compromised, it cannot access other tenants' job messages.

### RBAC Assignment Lifecycle

| Event | RBAC Action |
|-------|-------------|
| Worker approved | Assign `Data Receiver` on topic to worker MI |
| Worker deactivated | Remove `Data Receiver` assignment |
| Worker suspended | Remove `Data Receiver` assignment (re-assign on restore) |
| Worker restored | Re-assign `Data Receiver` on topic |

---

## Configuration

### Service Bus Namespace Settings

| Setting | Value | Rationale |
|---------|-------|-----------|
| Tier | Standard | Topics + subscriptions support (not in Basic) |
| Messaging units | 1 (MVP) | Scale up if throughput requires |
| Duplicate detection | Enabled, 10 min window | Prevents re-publish duplication |
| Geo-replication | Disabled (MVP) | Enable for DR later |

### Topic Settings (Per Tenant)

| Setting | Value | Rationale |
|---------|-------|-----------|
| `default_message_ttl` | 1 hour | Jobs should be picked up within this window |
| `max_size_in_megabytes` | 1024 (1 GB) | More than enough for rotation job messages |
| `requires_duplicate_detection` | true | Dedup window = 10 min |
| `enable_partitioning` | false | Not needed at current scale — single worker per topic |
| `support_ordering` | false | Jobs are independent — no ordering requirement |

### Subscription Settings

| Setting | Value | Rationale |
|---------|-------|-----------|
| `max_delivery_count` | 3 | Retry at Service Bus level before going to DLQ |
| `lock_duration` | PT5M (5 min) | Worker has 5 min to start processing before lock expires |
| `default_message_ttl` | PT1H (1 hour) | Inherited from topic |
| `dead_lettering_on_message_expiration` | true | Expired messages → DLQ for system task to reconcile |
| `enable_dead_lettering_on_filter_evaluation_exceptions` | true | Safety net |
| `auto_delete_on_idle` | disabled | Subscription persists even if worker is offline |

---

## Relationship to Job State Machine

How Service Bus states map to the `rotation_jobs` database state:

```
DB: PENDING
    │
    │  Scheduler publishes message
    ▼
DB: PUBLISHED  ←→  Service Bus: message in subscription (invisible until delivered)
    │
    │  Worker receives (PeekLock)
    │  Worker calls POST /api/rotations/{jobId}/status {status: "RUNNING"}
    ▼
DB: RUNNING  ←→  Service Bus: message locked by worker
    │
    │  Worker completes rotation
    │  Worker calls POST /api/rotations/{jobId}/status {status: "COMPLETED"}
    │  Worker calls CompleteMessage
    ▼
DB: COMPLETED  ←→  Service Bus: message removed
```

**Failure path**:
```
DB: RUNNING
    │
    │  Worker fails
    │  Worker calls POST /api/rotations/{jobId}/status {status: "FAILED"}
    │  Worker calls AbandonMessage (or DeadLetterMessage)
    ▼
DB: FAILED  ←→  Service Bus: message back in sub (or in DLQ)
    │
    │  System Task promotes retry (if attempts < max)
    ▼
DB: PENDING  ←→  (cycle repeats — scheduler re-publishes)
```

---

## Go SDK Usage

The platform uses `github.com/Azure/azure-sdk-for-go/sdk/messaging/azservicebus` for runtime operations and `github.com/Azure/azure-sdk-for-go/sdk/messaging/azservicebus/admin` for management operations.

### Key Interfaces

**Scheduler (publisher)**:
- `admin.Client` — create/delete topics and subscriptions (used by API service too)
- `azservicebus.Client` → `NewSender(topicName)` — publish messages

**Worker (receiver)**:
- `azservicebus.Client` → `NewReceiverForSubscription(topicName, subscriptionName)` — receive messages
- `receiver.ReceiveMessages(ctx, count, options)` — blocking receive
- `receiver.CompleteMessage(ctx, msg)` / `receiver.AbandonMessage(ctx, msg)` / `receiver.DeadLetterMessage(ctx, msg, options)`

**All authentication** via `azidentity.NewDefaultAzureCredential()` — picks up Workload Identity (AKS) or Managed Identity (Container App) automatically.

---

## Observability

### Metrics to Track

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| Messages published per minute | Scheduler logs | < expected rate → scheduler stuck |
| Subscription active message count | Service Bus metrics | > 10 per tenant → worker falling behind |
| DLQ message count | Service Bus metrics | > 0 → rotation failures need attention |
| Publish latency (p99) | Scheduler traces | > 2s → Service Bus connectivity issue |
| Time in PUBLISHED state | DB query (system task) | > 15 min → worker may be offline |
| Worker receive-to-complete latency | Worker traces | > 5 min → slow rotations |

### Diagnostic Logs

Service Bus diagnostic logs sent to Log Analytics:
- `OperationalLogs` — connection issues, throttling, auth failures
- `RuntimeAuditLogs` — detailed send/receive/dead-letter events (PII-filtered)

---

## Security Considerations

| Threat | Mitigation |
|--------|-----------|
| Worker reads another tenant's messages | RBAC scoped to specific topic — Azure enforces |
| Message tampering in transit | TLS (Service Bus enforces transport security) |
| Replay attack (old job re-processed) | `MessageId` deduplication + job state machine (COMPLETED jobs can't transition) |
| Worker processes expired job | Worker checks `published_at` + policy TTL before executing |
| Scheduler publishes to wrong topic | Topic name derived from tenant slug — validated at publish time |
| Namespace-wide compromise | Network rules restrict access to AKS VNET + worker Container App outbound IPs |

---

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| One topic per tenant | Isolation | Hard RBAC boundary between tenants at infra level |
| Single subscription per topic | One worker per tenant | No competition, no fan-out needed |
| Standard tier (not Premium) | Cost | Sufficient for expected throughput; upgrade path exists |
| PeekLock (not ReceiveAndDelete) | Reliability | Don't lose messages if worker crashes mid-processing |
| 5-min lock duration | Balance | Enough time for most rotations; not so long that failed workers block the queue |
| Message contains full job payload | Worker independence | Worker doesn't need to call API for job details — reduces coupling |
| Max delivery = 3 | Retry budget | Aligns with `rotation_policies.max_retry_attempts` default |
| TTL = 1 hour | Staleness bound | Jobs older than 1 hour are likely stale — system task handles |
| No sessions/ordering | Simplicity | Each rotation job is independent; no ordering requirements |
| RBAC per-topic for workers | Least privilege | Worker only sees its own tenant's messages |
