# Scheduler Service — Detailed Design

## Overview

The Scheduler is a **long-running Go service** deployed as a Kubernetes Deployment in the `az-rotator` namespace. It is the bridge between database state and Service Bus — its sole job is to find rotation work that needs doing and deliver it to workers.

Two distinct responsibilities:
1. **Job Creation** — scan `secret_registrations` for secrets due for rotation, create `rotation_jobs` records
2. **Job Publishing** — find PENDING jobs, validate worker health, publish messages to the tenant's Service Bus topic

The scheduler never rotates secrets. It never talks to workers directly. It reads the database, writes to the database, and publishes to Service Bus.

---

## Why a Separate Service (Not Part of the API)

| Concern | Embedded in API | Separate Scheduler |
|---------|----------------|-------------------|
| Polling loops | Complicates API lifecycle | Dedicated process, clean shutdown |
| Scaling | API scales for traffic; scheduler needs exactly 1 | Independent scaling decisions |
| Failure blast radius | Scheduler bug takes down API | Scheduler failure doesn't affect user requests |
| Resource profile | API is bursty; scheduler is steady | Each sized for its workload |
| Deployment cadence | Tied to API releases | Can release independently |

The scheduler is a control-loop service — it polls, evaluates, acts, sleeps, repeats. This pattern doesn't belong inside a request-driven API.

---

## Architecture Position

```
                                 ┌─────────────────┐
                                 │   PostgreSQL     │
                                 │                  │
                                 │ secret_registrations
                                 │ rotation_jobs    │
                                 │ workers          │
                                 └────────┬─────────┘
                                          │
                          reads/writes     │
                                          │
                                 ┌────────▼─────────┐
                                 │   Scheduler       │
                                 │   (1 replica)     │
                                 │                   │
                                 │ ┌───────────────┐ │
                                 │ │ Job Creator   │ │  ← finds due secrets
                                 │ └───────────────┘ │
                                 │ ┌───────────────┐ │
                                 │ │ Job Publisher  │ │  ← publishes to SB
                                 │ └───────────────┘ │
                                 └────────┬─────────┘
                                          │
                          publishes        │
                                          │
                                 ┌────────▼─────────┐
                                 │  Service Bus      │
                                 │  (topic per       │
                                 │   tenant)         │
                                 └──────────────────┘
```

---

## Job Creation Loop

The first responsibility: identify secrets whose rotation is due and create job records.

### Polling Strategy

```
Every 60 seconds (configurable via SCHEDULER_CREATE_INTERVAL):
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. Query due secrets:                                        │
│                                                              │
│    SELECT sr.*, kv.vault_uri, kv.id AS vault_id,             │
│           rp.max_retry_attempts, rp.validation_required      │
│    FROM secret_registrations sr                               │
│    JOIN key_vaults kv ON sr.vault_id = kv.id                 │
│    JOIN rotation_policies rp ON sr.policy_id = rp.id         │
│    WHERE sr.status = 'ACTIVE'                                │
│      AND sr.next_rotation_at <= NOW()                        │
│      AND NOT EXISTS (                                        │
│        SELECT 1 FROM rotation_jobs rj                        │
│        WHERE rj.secret_id = sr.id                            │
│        AND rj.status IN ('PENDING','PUBLISHED','RUNNING',    │
│                          'VALIDATING','RETRYING')             │
│      )                                                       │
│    ORDER BY sr.next_rotation_at ASC                           │
│    LIMIT 200                                                 │
│                                                              │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. For each due secret:                                      │
│    a. Check tenant is ACTIVE (skip suspended/deactivated)    │
│    b. Check worker is APPROVED (no point creating a job      │
│       if no worker can pick it up)                           │
│    c. INSERT rotation_jobs:                                  │
│       - status = 'PENDING'                                   │
│       - attempt_number = 1                                   │
│       - max_attempts = policy.max_retry_attempts             │
│       - scheduled_at = NOW()                                 │
│       - priority = 0 (normal)                                │
│    d. Audit log: job.created                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Why NOT EXISTS?

The `NOT EXISTS` subquery prevents creating duplicate jobs for the same secret when an active job already exists. Without this:
- Secret `A` is due for rotation
- Scheduler creates job #1 (PENDING)
- 60 seconds later, secret `A` still shows `next_rotation_at <= NOW()` (hasn't been rotated yet)
- Scheduler would create job #2 — duplicate

The subquery ensures only one active job per secret at any time.

### When next_rotation_at Updates

`next_rotation_at` is only updated when a job reaches `COMPLETED`:
- Worker reports success → API sets `secret_registrations.next_rotation_at = NOW() + policy.rotation_interval`
- Until completion, the due date stays in the past — but the `NOT EXISTS` guard prevents re-scheduling

---

## Job Publishing Loop

The second responsibility: take PENDING jobs and publish them to Service Bus.

### Polling Strategy

```
Every 30 seconds (configurable via SCHEDULER_PUBLISH_INTERVAL):
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. Query publishable jobs:                                   │
│                                                              │
│    SELECT rj.*,                                              │
│           sr.name AS secret_name,                            │
│           sr.secret_type, sr.resource_id, sr.config,         │
│           sr.secret_name_in_vault,                           │
│           kv.vault_uri,                                      │
│           rp.validation_required, rp.max_retry_attempts,     │
│           t.slug AS tenant_slug,                             │
│           w.status AS worker_status                           │
│    FROM rotation_jobs rj                                     │
│    JOIN secret_registrations sr ON rj.secret_id = sr.id      │
│    JOIN key_vaults kv ON sr.vault_id = kv.id                 │
│    JOIN rotation_policies rp ON rj.policy_id = rp.id         │
│    JOIN tenants t ON rj.tenant_id = t.id                     │
│    JOIN workers w ON w.tenant_id = rj.tenant_id              │
│    WHERE rj.status = 'PENDING'                               │
│      AND rj.scheduled_at <= NOW()                            │
│      AND t.status = 'ACTIVE'                                 │
│      AND w.status = 'APPROVED'                               │
│    ORDER BY rj.priority DESC, rj.scheduled_at ASC            │
│    LIMIT 100                                                 │
│                                                              │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Group jobs by tenant (for batch publishing):              │
│    map[tenantSlug] → []job                                   │
│                                                              │
│ 3. For each tenant batch:                                    │
│    a. Resolve topic: "tenant-{slug}-rotations"               │
│    b. Build messages (see Message Assembly below)             │
│    c. Publish batch to topic                                 │
│    d. For each successfully published job:                    │
│       → UPDATE status = 'PUBLISHED'                          │
│       → SET published_at = NOW()                             │
│    e. Audit log: job.published (per job)                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Worker Health Gate

The `JOIN workers w ... AND w.status = 'APPROVED'` clause ensures we never publish to a tenant whose worker is dead:

- If worker is `SUSPENDED` → jobs stay PENDING until worker recovers
- If worker is `PENDING_APPROVAL` → jobs stay PENDING until approval
- If worker is `DEACTIVATED` → jobs stay PENDING (and eventually get caught by reconciler)

This avoids publishing messages to a topic where nothing is listening, which would just waste the message TTL and result in DLQ entries.

### Message Assembly

For each job, the scheduler builds a Service Bus message:

```go
msg := &azservicebus.Message{
    MessageID:   &job.ID,                    // UUID deduplication
    ContentType: to.Ptr("application/json"),
    Subject:     to.Ptr("rotation-job"),
    Body:        marshaledBody,               // full job payload JSON
    TimeToLive:  to.Ptr(1 * time.Hour),
    ApplicationProperties: map[string]interface{}{
        "tenant_id":      job.TenantID,
        "secret_id":      job.SecretID,
        "job_id":         job.ID,
        "secret_type":    secret.SecretType,
        "attempt_number": job.AttemptNumber,
        "priority":       job.Priority,
    },
}
```

The message body includes everything the worker needs (see [SERVICE_BUS.md](SERVICE_BUS.md) for full schema).

### Batch Publishing

Jobs targeting the same tenant are published in a single batch:
- Service Bus supports batches up to 256KB or a configurable max message count
- Reduces round trips to Service Bus (1 call per tenant instead of 1 per job)
- If a batch exceeds size limits, it's split into smaller batches automatically

```go
batch, err := sender.NewMessageBatch(ctx, nil)
for _, msg := range messages {
    if err := batch.AddMessage(msg, nil); err != nil {
        // Batch full — send current batch, start new one
        sender.SendMessageBatch(ctx, batch, nil)
        batch, _ = sender.NewMessageBatch(ctx, nil)
        batch.AddMessage(msg, nil)
    }
}
// Send remaining
sender.SendMessageBatch(ctx, batch, nil)
```

---

## Publish Guarantees & Failure Handling

### At-Least-Once Delivery

The scheduler ensures every PENDING job eventually gets published:

1. Job stays `PENDING` until publish succeeds AND DB update commits
2. If scheduler crashes after publishing but before DB update → job stays PENDING → re-published next cycle
3. Service Bus deduplication (10-minute window on `MessageId = job UUID`) prevents the worker from seeing duplicates

### Publish Failures

| Failure | Scheduler Action | Recovery |
|---------|-----------------|----------|
| Service Bus unreachable | Log error, skip this cycle | Retry next polling interval |
| Topic doesn't exist | Log error, skip tenant's jobs | Indicates tenant topic was deleted — admin investigation |
| Auth failure (MI issue) | Log error, skip all publishes | Alert fires, pod identity issue |
| Partial batch failure | Commit successful ones, retry rest | Partially published → rest next cycle |
| DB update failure after publish | Job stays PENDING, re-published | SB dedup protects against double-delivery |

### Idempotent Publishing

Even if a job is published twice (crash recovery), the system handles it safely:
- Service Bus deduplication drops the duplicate within 10 minutes
- Even without dedup, the worker reports status to API — second message for an already RUNNING/COMPLETED job would be completed immediately by the worker (checks current status on the API)
- The `MessageId = job.ID` ensures the same logical job maps to the same message identity

---

## Leader Election

The scheduler runs as a **single replica** in production. If scaling beyond 1 for availability:

### Why Single Replica is Fine (MVP)

- Scheduler going down for 1–2 minutes (pod restart) just delays publishing
- Jobs accumulate as PENDING, published on recovery
- No data loss — just latency
- Kubernetes Deployment with `replicas: 1` and `maxSurge: 1` handles rolling restarts

### Future: Multi-Replica with Lease

If HA is needed later:

```
Option A: Kubernetes Lease
- Use client-go's leaderelection package
- Only the leader runs creation/publish loops
- Follower sits idle, takes over if leader dies
- Failover time: ~15 seconds

Option B: PostgreSQL Advisory Lock
- pg_try_advisory_lock(scheduler_lock_id)
- Only one instance acquires the lock
- Simpler than Kubernetes lease for DB-heavy services
- Lock released on disconnect
```

**Decision**: Single replica for MVP. The scheduler is stateless (all state in DB) — fast restart is sufficient. Add leader election only if the 30–60 second restart gap becomes unacceptable.

---

## Configuration

All configuration via environment variables (12-factor):

| Variable | Default | Description |
|----------|---------|-------------|
| `SCHEDULER_CREATE_INTERVAL` | `60s` | How often to scan for due secrets |
| `SCHEDULER_PUBLISH_INTERVAL` | `30s` | How often to publish PENDING jobs |
| `SCHEDULER_CREATE_BATCH_SIZE` | `200` | Max secrets to process per creation cycle |
| `SCHEDULER_PUBLISH_BATCH_SIZE` | `100` | Max jobs to publish per cycle |
| `SCHEDULER_PUBLISH_STALE_THRESHOLD` | `1h` | How long before PUBLISHED is considered stale |
| `SCHEDULER_WORKER_HEALTH_GRACE` | `5m` | Heartbeat grace period to consider worker healthy |
| `DATABASE_URL` | — | PostgreSQL connection string |
| `SERVICEBUS_NAMESPACE` | — | Service Bus fully qualified namespace |
| `SERVICEBUS_DEDUP_WINDOW` | `10m` | Message deduplication window for topics |
| `LOG_LEVEL` | `info` | Logging verbosity |
| `OTEL_EXPORTER_ENDPOINT` | — | OpenTelemetry collector endpoint |

---

## Startup & Shutdown

### Startup Sequence

```
1. Parse configuration (env vars)
2. Connect to PostgreSQL (retry with backoff, max 30s)
3. Create Service Bus client (namespace-level, Managed Identity)
4. Start health/readiness HTTP server (:8081)
   - /healthz → returns 200 if process alive
   - /readyz  → returns 200 if DB connected + SB client ready
5. Start Job Creation loop (goroutine)
6. Start Job Publishing loop (goroutine)
7. Block on OS signal (SIGTERM/SIGINT)
```

### Graceful Shutdown

```
1. Receive SIGTERM (Kubernetes pod termination)
2. Stop accepting new loop iterations
3. Wait for current in-flight publish batch to complete (max 30s)
4. Close Service Bus client
5. Close database connection pool
6. Exit 0
```

The Kubernetes `terminationGracePeriodSeconds` is set to 60 seconds — more than enough for the scheduler to drain.

### Health Probes

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8081
  initialDelaySeconds: 5
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /readyz
    port: 8081
  initialDelaySeconds: 10
  periodSeconds: 5
```

The scheduler doesn't receive traffic from a Service, but probes ensure Kubernetes restarts it if something goes wrong.

---

## Observability

### Metrics (OpenTelemetry)

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `scheduler.create.secrets_scanned` | Counter | — | Total secrets evaluated per cycle |
| `scheduler.create.jobs_created` | Counter | `tenant_id` | Jobs created per tenant |
| `scheduler.create.skipped_no_worker` | Counter | `tenant_id` | Secrets skipped because worker unhealthy |
| `scheduler.publish.jobs_published` | Counter | `tenant_id` | Jobs published to Service Bus |
| `scheduler.publish.batch_size` | Histogram | `tenant_id` | Messages per batch |
| `scheduler.publish.latency_ms` | Histogram | `tenant_id` | Time to publish a batch |
| `scheduler.publish.errors` | Counter | `tenant_id`, `error_type` | Publish failures |
| `scheduler.loop.duration_ms` | Histogram | `loop` (`create`/`publish`) | How long each loop iteration takes |
| `scheduler.loop.last_success` | Gauge | `loop` | Unix timestamp of last successful run |

### Structured Logging

Every log line includes:
- `service: scheduler`
- `loop: create | publish`
- `tenant_id` (when processing tenant-specific work)
- `job_id` (when publishing a specific job)
- `duration_ms` (for timed operations)

Example log lines:
```json
{"level":"info","service":"scheduler","loop":"create","msg":"cycle complete","secrets_scanned":45,"jobs_created":3,"duration_ms":120}
{"level":"info","service":"scheduler","loop":"publish","msg":"batch published","tenant_id":"uuid","topic":"tenant-team-platform-prod-rotations","count":2,"duration_ms":85}
{"level":"warn","service":"scheduler","loop":"publish","msg":"skipped tenant - worker not healthy","tenant_id":"uuid","worker_status":"SUSPENDED"}
```

### Alerts

| Condition | Severity | Action |
|-----------|----------|--------|
| Publish loop hasn't succeeded in 5 min | Warning | Check scheduler pod logs |
| Publish errors > 10 in 5 min | Critical | Service Bus connectivity issue |
| Create loop found due secrets but no healthy worker | Warning | Tenant notification |
| Scheduler pod restarting (CrashLoopBackOff) | Critical | Investigate DB/SB connectivity |

---

## Database Access Pattern

The scheduler uses the `az_rotator_scheduler` database role:

```sql
-- Role definition
CREATE ROLE az_rotator_scheduler LOGIN PASSWORD '...';

-- Grants
GRANT SELECT ON secret_registrations TO az_rotator_scheduler;
GRANT SELECT ON rotation_policies TO az_rotator_scheduler;
GRANT SELECT ON key_vaults TO az_rotator_scheduler;
GRANT SELECT ON tenants TO az_rotator_scheduler;
GRANT SELECT ON workers TO az_rotator_scheduler;
GRANT SELECT, INSERT, UPDATE ON rotation_jobs TO az_rotator_scheduler;
GRANT INSERT ON audit_logs TO az_rotator_scheduler;
```

**RLS**: The scheduler operates **cross-tenant** — it processes all tenants' secrets in a single loop. It either:
- Bypasses RLS (like system tasks), or
- Uses a role that isn't subject to tenant-scoped RLS policies

Since the scheduler only reads secrets/workers/tenants (never exposes them via API), cross-tenant access is safe.

### Connection Pool

```
Min connections: 2
Max connections: 10
Idle timeout: 5 minutes
Max lifetime: 30 minutes
```

The scheduler doesn't need many connections — it runs two loops sequentially (not concurrent queries per loop). A small pool handles the steady load.

---

## Service Bus Access Pattern

The scheduler uses **Azure Service Bus Data Sender** role at the namespace level:

```
Scope: /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ServiceBus/namespaces/az-rotator-servicebus
Role: Azure Service Bus Data Sender
Principal: Scheduler Managed Identity (Workload Identity)
```

This grants permission to publish messages to any topic in the namespace. The scheduler doesn't need Receiver access — it never reads from subscriptions or DLQs.

### Client Management

```go
// One client per namespace (shared across all topics)
client, err := azservicebus.NewClient(namespace, credential, nil)

// One sender per topic (cached, reused across cycles)
senderCache := map[string]*azservicebus.Sender{}

func getSender(tenantSlug string) *azservicebus.Sender {
    topic := fmt.Sprintf("tenant-%s-rotations", tenantSlug)
    if s, ok := senderCache[topic]; ok {
        return s
    }
    s, _ := client.NewSender(topic, nil)
    senderCache[topic] = s
    return s
}
```

Senders are cached per topic and reused across publish cycles. They're lightweight and safe to hold long-term.

---

## Go Project Structure

```
cmd/
  scheduler/
    main.go              ← entry point, config parsing, DI wiring, signal handling

internal/
  scheduler/
    creator.go           ← Job Creation loop logic
    publisher.go         ← Job Publishing loop logic
    message.go           ← Service Bus message assembly
    batch.go             ← Batch publishing helpers
    config.go            ← Configuration struct + env parsing
    health.go            ← /healthz, /readyz handlers
    metrics.go           ← OTel metric definitions + recording

  database/
    queries/
      scheduler.sql      ← SQL queries (used by sqlc or hand-written)
    pool.go              ← Connection pool setup
    models.go            ← Generated/shared DB models

  servicebus/
    client.go            ← Service Bus client initialization
    sender.go            ← Sender cache + publish helpers
```

### Key Interfaces

```go
// internal/scheduler/creator.go

type JobCreator struct {
    db       *database.Pool
    interval time.Duration
    batch    int
    metrics  *Metrics
    logger   *slog.Logger
}

func (c *JobCreator) Run(ctx context.Context) error {
    ticker := time.NewTicker(c.interval)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            if err := c.createDueJobs(ctx); err != nil {
                c.logger.Error("creation cycle failed", "error", err)
            }
        }
    }
}

func (c *JobCreator) createDueJobs(ctx context.Context) error {
    // 1. Query due secrets (NOT EXISTS active jobs)
    // 2. Filter: tenant ACTIVE, worker APPROVED
    // 3. INSERT rotation_jobs (PENDING)
    // 4. Record metrics
}
```

```go
// internal/scheduler/publisher.go

type JobPublisher struct {
    db         *database.Pool
    senders    *servicebus.SenderCache
    interval   time.Duration
    batch      int
    metrics    *Metrics
    logger     *slog.Logger
}

func (p *JobPublisher) Run(ctx context.Context) error {
    ticker := time.NewTicker(p.interval)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            if err := p.publishPendingJobs(ctx); err != nil {
                p.logger.Error("publish cycle failed", "error", err)
            }
        }
    }
}

func (p *JobPublisher) publishPendingJobs(ctx context.Context) error {
    // 1. Query PENDING jobs with tenant/secret/vault/policy/worker details
    // 2. Group by tenant slug
    // 3. For each tenant: build messages, batch publish
    // 4. Update job status → PUBLISHED
    // 5. Record metrics
}
```

```go
// cmd/scheduler/main.go

func main() {
    cfg := scheduler.LoadConfig()

    // Dependencies
    pool := database.NewPool(cfg.DatabaseURL)
    sbClient := servicebus.NewClient(cfg.ServiceBusNamespace)
    senders := servicebus.NewSenderCache(sbClient)

    // Loops
    creator := scheduler.NewJobCreator(pool, cfg, metrics, logger)
    publisher := scheduler.NewJobPublisher(pool, senders, cfg, metrics, logger)

    // Health server
    health := scheduler.NewHealthServer(pool, sbClient)
    go health.ListenAndServe(":8081")

    // Run loops in goroutines
    ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
    defer cancel()

    g, gctx := errgroup.WithContext(ctx)
    g.Go(func() error { return creator.Run(gctx) })
    g.Go(func() error { return publisher.Run(gctx) })

    if err := g.Wait(); err != nil && !errors.Is(err, context.Canceled) {
        logger.Error("scheduler exited with error", "error", err)
        os.Exit(1)
    }
}
```

---

## Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scheduler
  namespace: az-rotator
  labels:
    app: scheduler
    component: control-plane
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: scheduler
  template:
    metadata:
      labels:
        app: scheduler
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: scheduler-sa
      terminationGracePeriodSeconds: 60
      containers:
        - name: scheduler
          image: ghcr.io/org/az-rotator-scheduler:latest
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          env:
            - name: SCHEDULER_CREATE_INTERVAL
              value: "60s"
            - name: SCHEDULER_PUBLISH_INTERVAL
              value: "30s"
            - name: SCHEDULER_CREATE_BATCH_SIZE
              value: "200"
            - name: SCHEDULER_PUBLISH_BATCH_SIZE
              value: "100"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: scheduler-db-credentials
                  key: url
            - name: SERVICEBUS_NAMESPACE
              value: "az-rotator-servicebus.servicebus.windows.net"
            - name: LOG_LEVEL
              value: "info"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8081
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8081
            initialDelaySeconds: 10
            periodSeconds: 5
```

### Workload Identity

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: scheduler-sa
  namespace: az-rotator
  annotations:
    azure.workload.identity/client-id: "<scheduler-mi-client-id>"
```

The scheduler's Managed Identity has:
- `Azure Service Bus Data Sender` on the Service Bus namespace
- PostgreSQL login via Entra auth (or connection string in secret)

---

## Interaction with Other Services

| Service | Interaction | Direction |
|---------|-------------|-----------|
| **PostgreSQL** | Read secrets/policies/workers/tenants, write jobs/audit | Scheduler → DB |
| **Service Bus** | Publish rotation messages to tenant topics | Scheduler → SB |
| **API Service** | None — scheduler doesn't call the API | — |
| **Workers** | None — communication is indirect via Service Bus | — |
| **System Tasks** | Handles jobs the scheduler published that went wrong | Complementary |

### Scheduler ↔ System Tasks Boundary

| Concern | Scheduler | System Tasks |
|---------|-----------|--------------|
| Create jobs | ✓ | ✗ |
| Publish jobs | ✓ | ✗ |
| Retry failed jobs | ✗ | ✓ (Retry Promoter resets to PENDING, scheduler re-publishes) |
| Detect stale jobs | ✗ | ✓ (Job Reconciler) |
| Read DLQ | ✗ | ✓ (DLQ Drainer) |
| Mark dead-lettered | ✗ | ✓ (Dead-Letter Promoter) |

The scheduler focuses on the happy path. System tasks handle the failure paths.

---

## Edge Cases & Safeguards

### Secret with No Policy

If `secret_registrations.policy_id` is NULL or the policy doesn't exist:
- `next_rotation_at` will be NULL → scheduler won't pick it up
- The secret is effectively paused until a policy is assigned

### Tenant Gets Suspended Mid-Cycle

If a tenant becomes `SUSPENDED` while its jobs are PENDING:
- Publish loop's `WHERE t.status = 'ACTIVE'` filters them out
- Jobs stay PENDING indefinitely until tenant is reactivated
- No accumulation problem — `NOT EXISTS` prevents re-creation

### Worker Dies After Publish

If the scheduler publishes successfully but the worker is dead:
1. Message sits in subscription until lock_duration expires (5 min)
2. Service Bus redelivers (up to max_delivery_count = 3)
3. After 3 failed deliveries → DLQ
4. DLQ Drainer marks job FAILED
5. Worker Health Monitor marks worker SUSPENDED
6. Future PENDING jobs for that tenant stay PENDING (worker gate)
7. When worker recovers → APPROVED → Retry Promoter makes job PENDING → scheduler publishes again

### Clock Skew

- `scheduled_at <= NOW()` uses database server time (not scheduler pod time)
- All time comparisons happen in PostgreSQL → consistent regardless of pod timezone
- `created_at`, `published_at` also use `NOW()` from DB

### Duplicate Prevention Summary

| Layer | Mechanism |
|-------|-----------|
| DB level | `NOT EXISTS` subquery prevents duplicate job creation |
| Service Bus | `MessageId = job UUID` + dedup window prevents duplicate messages |
| Worker level | Worker checks job status via API before processing (idempotent check) |

---

## Manual Job Trigger

Users can create jobs manually via the API (`POST /api/tenants/{tid}/secrets/{sid}/rotate`). These bypass the creation loop but enter the same flow:

1. API creates `rotation_jobs` with `status = 'PENDING'`, `scheduled_at = NOW()`
2. Scheduler's publish loop picks it up on the next cycle (≤30 seconds)
3. Same publish flow, same guarantees

The scheduler doesn't distinguish between auto-created and manually-created jobs — it publishes anything that's PENDING.

---

## Summary

The scheduler is deliberately simple: two polling loops, each doing one thing:

1. **Create**: Find due secrets → create PENDING jobs
2. **Publish**: Find PENDING jobs → publish to Service Bus → mark PUBLISHED

No complex orchestration, no distributed locks (at MVP), no direct worker coordination. The database is the source of truth. Service Bus is the delivery mechanism. System tasks handle everything that goes wrong after publishing.
