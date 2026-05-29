# System Tasks — Detailed Design

## Overview

System Tasks are **Kubernetes CronJobs** that run in the `az-rotator-system` namespace. They handle background maintenance that no single request triggers — reconciliation, retry promotion, cleanup, and health enforcement.

They are the platform's **janitor**: they detect problems, fix inconsistencies, and keep the system healthy without human intervention.

---

## Why CronJobs (Not Long-Running Services)

| Concern | Long-running service | CronJob |
|---------|---------------------|---------|
| Resource usage | Constantly consuming CPU/memory | Only runs when scheduled |
| Failure recovery | Needs restart logic, health probes | Kubernetes re-runs on next schedule |
| Scaling | Needs leader election if >1 replica | Single execution guaranteed by schedule |
| Observability | Logs mixed with other ops | Each run produces a clean start/finish in logs |
| Complexity | Must handle idle periods | Simple: start → do work → exit |

System tasks don't need to be running 24/7 — they just need to run periodically. CronJobs are the natural fit.

---

## Task Inventory

| Task | Schedule | Purpose | Database Role |
|------|----------|---------|---------------|
| **Job Reconciler** | Every 5 min | Detect stale PUBLISHED/RUNNING jobs, reset or fail them | `az_rotator_system` (bypasses RLS) |
| **Retry Promoter** | Every 5 min | Promote FAILED jobs for retry (if attempts remain) | `az_rotator_system` |
| **Dead-Letter Promoter** | Every 5 min | Mark exhausted jobs as DEAD_LETTERED | `az_rotator_system` |
| **DLQ Drainer** | Every 5 min | Read Service Bus DLQs, sync DB state | `az_rotator_system` |
| **Worker Health Monitor** | Every 2 min | Detect dead workers (no heartbeat), mark SUSPENDED | `az_rotator_system` |
| **Heartbeat Cleanup** | Daily (02:00 UTC) | Delete old heartbeat records beyond retention | `az_rotator_system` |
| **Audit Log Cleanup** | Daily (03:00 UTC) | Drop old audit log partitions beyond retention | `az_rotator_system` |

All tasks use the `az_rotator_system` database role which **bypasses RLS** — they need cross-tenant access to reconcile state across the whole platform.

---

## Task Details

### 1. Job Reconciler

**Schedule**: `*/5 * * * *` (every 5 minutes)

**Problem it solves**: Jobs can get stuck in intermediate states if things go wrong:
- `PUBLISHED` but worker never picked it up (worker offline, message expired, Service Bus issue)
- `RUNNING` but worker crashed and never reported back

**Logic**:

```
┌─────────────────────────────────────────────────────┐
│ Job Reconciler                                       │
│                                                      │
│ Step 1: Find stale PUBLISHED jobs                    │
│   SELECT * FROM rotation_jobs                        │
│   WHERE status = 'PUBLISHED'                         │
│   AND published_at < NOW() - INTERVAL '1 hour'      │
│                                                      │
│   For each:                                          │
│     → UPDATE status = 'PENDING'                      │
│     → Clear published_at                             │
│     → Audit log: job.recycled (reason: publish_stale)│
│                                                      │
│ Step 2: Find stale RUNNING jobs                      │
│   SELECT * FROM rotation_jobs                        │
│   WHERE status = 'RUNNING'                           │
│   AND started_at < NOW() - INTERVAL '30 minutes'    │
│                                                      │
│   For each:                                          │
│     → UPDATE status = 'FAILED'                       │
│     → Set error_message = 'Worker unresponsive'      │
│     → Set completed_at = NOW()                       │
│     → Audit log: job.failed (reason: running_stale)  │
│                                                      │
│ Step 3: Find stale VALIDATING jobs                   │
│   SELECT * FROM rotation_jobs                        │
│   WHERE status = 'VALIDATING'                        │
│   AND updated_at < NOW() - INTERVAL '15 minutes'    │
│                                                      │
│   For each:                                          │
│     → Same as RUNNING (mark FAILED)                  │
└─────────────────────────────────────────────────────┘
```

**Thresholds** (configurable via env vars):

| State | Threshold | Rationale |
|-------|-----------|-----------|
| PUBLISHED → stale | 1 hour | Matches Service Bus message TTL |
| RUNNING → stale | 30 min | Most rotations take < 5 min; 30 min is generous |
| VALIDATING → stale | 15 min | Validation should be fast |

---

### 2. Retry Promoter

**Schedule**: `*/5 * * * *` (every 5 minutes)

**Problem it solves**: When a job fails, it should be retried with exponential backoff — up to the policy's max attempts.

**Logic**:

```
┌─────────────────────────────────────────────────────┐
│ Retry Promoter                                       │
│                                                      │
│ SELECT * FROM rotation_jobs                          │
│ WHERE status = 'FAILED'                              │
│ AND attempt_number < max_attempts                    │
│ AND completed_at < NOW() - backoff_interval          │
│                                                      │
│ For each:                                            │
│   1. Calculate next backoff:                         │
│      base * 2^(attempt_number - 1)                   │
│      e.g. 5min, 10min, 20min                         │
│                                                      │
│   2. If NOW() > completed_at + backoff:              │
│      → UPDATE status = 'RETRYING'                    │
│      → UPDATE attempt_number += 1                    │
│      → UPDATE scheduled_at = NOW()                   │
│      → Clear error_message, started_at, completed_at │
│      → UPDATE status = 'PENDING' (ready for publish) │
│      → Audit log: job.retrying                       │
│                                                      │
│   3. If not yet past backoff window:                 │
│      → Skip (not ready for retry yet)               │
└─────────────────────────────────────────────────────┘
```

**Backoff calculation**:

```
backoff = retry_backoff_base × 2^(attempt_number - 1)

Example with base = 5 minutes:
  Attempt 1 fails → wait 5 min → retry
  Attempt 2 fails → wait 10 min → retry
  Attempt 3 fails → wait 20 min → dead-letter
```

The `retry_backoff_base` comes from the job's `rotation_policies.retry_backoff_base`.

---

### 3. Dead-Letter Promoter

**Schedule**: `*/5 * * * *` (every 5 minutes)

**Problem it solves**: Jobs that have exhausted all retry attempts should be permanently marked as dead-lettered so they stop being processed.

**Logic**:

```
┌─────────────────────────────────────────────────────┐
│ Dead-Letter Promoter                                 │
│                                                      │
│ SELECT * FROM rotation_jobs                          │
│ WHERE status = 'FAILED'                              │
│ AND attempt_number >= max_attempts                   │
│                                                      │
│ For each:                                            │
│   → UPDATE status = 'DEAD_LETTERED'                  │
│   → Audit log: job.dead_lettered                     │
│   → (Future: notify tenant via email/webhook)        │
└─────────────────────────────────────────────────────┘
```

Once dead-lettered, a job is terminal. It requires manual intervention from the tenant admin (investigate the failure, fix the root cause, create a new manual job).

---

### 4. DLQ Drainer

**Schedule**: `*/5 * * * *` (every 5 minutes)

**Problem it solves**: Messages in Service Bus dead-letter queues need to be reconciled with database state. Otherwise the DB thinks a job is still in-flight when it's actually dead in Service Bus.

**Logic**:

```
┌─────────────────────────────────────────────────────┐
│ DLQ Drainer                                          │
│                                                      │
│ For each active tenant:                              │
│   1. Get tenant's topic name                         │
│   2. Connect to DLQ for subscription "worker"        │
│   3. Peek messages (up to 50)                        │
│                                                      │
│   For each DLQ message:                              │
│     a. Extract job_id from application properties    │
│     b. Look up rotation_jobs by job_id               │
│     c. If status not already terminal:               │
│        → UPDATE status = 'FAILED'                    │
│        → Set error_message from DLQ reason           │
│        → Audit log: job.dlq_received                 │
│     d. Complete the DLQ message (remove it)          │
│                                                      │
│   4. Check DLQ depth (active message count)          │
│      If > 5 → log warning (worker may be broken)     │
│      If > 20 → alert                                │
└─────────────────────────────────────────────────────┘
```

**Note**: DLQ messages are read and removed — they're reconciled into the DB and the retry/dead-letter promoters handle the rest.

**Auth**: System Tasks use `Azure Service Bus Data Receiver` at the namespace level (see SERVICE_BUS.md RBAC section).

---

### 5. Worker Health Monitor

**Schedule**: `*/2 * * * *` (every 2 minutes)

**Problem it solves**: If a worker crashes or goes offline without cleanly shutting down, the platform needs to detect it and suspend the worker so no new jobs are published to a dead subscription.

**Logic**:

```
┌─────────────────────────────────────────────────────┐
│ Worker Health Monitor                                │
│                                                      │
│ SELECT * FROM workers                                │
│ WHERE status = 'APPROVED'                            │
│ AND last_heartbeat_at < NOW() - INTERVAL '5 minutes' │
│                                                      │
│ For each:                                            │
│   → UPDATE status = 'SUSPENDED'                      │
│   → Audit log: worker.suspended (reason: heartbeat_  │
│     timeout)                                         │
│   → (Future: notify tenant admin)                    │
│   → Remove Service Bus RBAC (receiver role)          │
│                                                      │
│ Note: Scheduler also checks worker health before     │
│ publishing. But this task is the definitive enforcer.│
└─────────────────────────────────────────────────────┘
```

**Heartbeat threshold**: 5 minutes (configurable). Workers send heartbeats every 60 seconds, so missing 5 consecutive heartbeats = definitely offline.

**What happens to in-flight jobs**: If the worker was mid-rotation when it died:
1. The RUNNING job will be caught by Job Reconciler (30 min stale threshold)
2. The Service Bus message lock expires (5 min), message goes back to subscription
3. But worker is now SUSPENDED → message will go to DLQ after TTL
4. DLQ Drainer marks it FAILED → Retry Promoter re-schedules once worker is restored

---

### 6. Heartbeat Cleanup

**Schedule**: `0 2 * * *` (daily at 02:00 UTC)

**Problem it solves**: The `worker_heartbeats` table grows continuously. Old records have no operational value after a short period.

**Logic**:

```
┌─────────────────────────────────────────────────────┐
│ Heartbeat Cleanup                                    │
│                                                      │
│ DELETE FROM worker_heartbeats                         │
│ WHERE reported_at < NOW() - INTERVAL '30 days'       │
│                                                      │
│ (If partitioned: DROP old partitions instead)        │
│                                                      │
│ Log: "Deleted {count} heartbeat records"             │
└─────────────────────────────────────────────────────┘
```

**Retention**: 30 days (configurable). Enough for diagnostics/trending but not infinite growth.

If partitioned by month, this becomes `DROP TABLE worker_heartbeats_2026_03` — instant, no vacuum needed.

---

### 7. Audit Log Cleanup

**Schedule**: `0 3 * * *` (daily at 03:00 UTC)

**Problem it solves**: Audit logs are append-only and grow indefinitely. Older logs must be purged per retention policy.

**Logic**:

```
┌─────────────────────────────────────────────────────┐
│ Audit Log Cleanup                                    │
│                                                      │
│ Default retention: 90 days                           │
│ (Future: per-tenant configurable retention)          │
│                                                      │
│ If partitioned:                                      │
│   → DROP TABLE audit_logs_2026_02                    │
│   → (drop partitions older than retention)           │
│                                                      │
│ If not partitioned:                                  │
│   → DELETE FROM audit_logs                           │
│     WHERE created_at < NOW() - INTERVAL '90 days'   │
│     LIMIT 10000 (batch delete to avoid long locks)   │
│   → Repeat until 0 rows affected                    │
│                                                      │
│ Log: "Cleaned {count} audit records"                 │
└─────────────────────────────────────────────────────┘
```

**Retention**: 90 days default. Could be made per-tenant in the future (stored on `tenants.metadata` or a separate config).

---

## How DLQ Drainer + Retry Promoter + Dead-Letter Promoter Work Together

These three tasks form a **pipeline that syncs Service Bus state with the database**. The core insight: Service Bus and PostgreSQL are two separate systems, and they can get out of sync. These tasks keep them aligned.

### The Problem

Service Bus has its own retry mechanism (`max_delivery_count = 3`). When a message fails 3 immediate deliveries, Service Bus moves it to the DLQ automatically. But the **database doesn't know this happened** — it still shows the job as `PUBLISHED` or `RUNNING`. Without the DLQ Drainer, the DB and Service Bus would drift apart permanently.

### The Pipeline

```
SERVICE BUS WORLD                          DATABASE WORLD
━━━━━━━━━━━━━━━━━                          ━━━━━━━━━━━━━━
                                           
Message delivered to worker ──────────────── Job status: PUBLISHED/RUNNING
         │                                        │
         │ Worker fails                           │
         │ (AbandonMessage)                       │ (no change yet — worker
         │                                        │  didn't report back)
         │ Service Bus retries 3x                 │
         │ (all fail)                             │
         ▼                                        │
Message lands in DLQ ─────────┐                   │
                              │                   │
                              │  DLQ Drainer      │
                              │  (every 5 min)    │
                              │                   │
                              └─────────────────▶ │
                                                  ▼
                                           Job status: FAILED
                                           attempt_number: still 1
                                                  │
                                                  │  Retry Promoter
                                                  │  (every 5 min, after backoff)
                                                  ▼
                                           Job status: PENDING
                                           attempt_number: 2
                                                  │
                                                  │  Scheduler re-publishes
                                                  ▼
New message in Service Bus ◀━━━━━━━━━━━━━━ Job status: PUBLISHED
         │                                        │
         │  (cycle repeats)                       │
         ...                                      ...
                                                  │
                                           After max attempts exhausted:
                                                  │
                                                  │  Dead-Letter Promoter
                                                  ▼
                                           Job status: DEAD_LETTERED
                                           (terminal — needs human)
```

### Key Point: Service Bus Retries ≠ Application Retries

| | Service Bus retry | Application retry |
|--|-------------------|-------------------|
| **What** | Same message re-delivered immediately | New message published after backoff |
| **Count** | 3 deliveries per message | 3 publish cycles (each with 3 deliveries = 9 total attempts) |
| **Speed** | Instant (message reappears in seconds) | Delayed (5 min, 10 min, 20 min backoff) |
| **Scope** | Transport-level (Service Bus handles it) | Application-level (system tasks handle it) |
| **Tracked where** | Service Bus delivery_count (invisible to DB) | `rotation_jobs.attempt_number` in DB |

Service Bus burning through its 3 deliveries = **one application attempt**. The DLQ Drainer's job is to translate "Service Bus gave up on this message" into "mark this as one FAILED attempt in the DB" so the Retry Promoter can decide whether to try again.

### Walkthrough: A Job That Eventually Succeeds on Attempt 2

```
1. Scheduler publishes job (attempt 1) → Service Bus message
2. Worker receives → fails → AbandonMessage (delivery 1)
3. Worker receives → fails → AbandonMessage (delivery 2)
4. Worker receives → fails → AbandonMessage (delivery 3 = max)
5. Service Bus moves message to DLQ
   ─── DB still thinks job is PUBLISHED ───
6. DLQ Drainer runs → reads message → marks job FAILED in DB (attempt 1)
7. DLQ Drainer removes message from DLQ
   ─── DB and Service Bus now in sync ───
8. Retry Promoter runs → sees FAILED, attempt 1 < max 3, backoff elapsed
   → resets job to PENDING, attempt_number = 2
9. Scheduler publishes again (attempt 2) → NEW Service Bus message
10. Worker receives → succeeds! → reports COMPLETED → CompleteMessage
    Done ✓
```

The DLQ Drainer is the **bridge** between Service Bus and the database. Without it, the DB would never learn that Service Bus gave up, and the Retry Promoter would never get a chance to schedule a new attempt.

---

### CronJob Manifests

Each task is a separate Kubernetes CronJob:

```
Namespace: az-rotator-system

CronJobs:
  ├── job-reconciler         (*/5 * * * *)
  ├── retry-promoter         (*/5 * * * *)
  ├── dead-letter-promoter   (*/5 * * * *)
  ├── dlq-drainer            (*/5 * * * *)
  ├── worker-health-monitor  (*/2 * * * *)
  ├── heartbeat-cleanup      (0 2 * * *)
  └── audit-log-cleanup      (0 3 * * *)
```

### CronJob Properties

| Property | Value | Rationale |
|----------|-------|-----------|
| `concurrencyPolicy` | `Forbid` | Don't overlap — if previous run is still going, skip this one |
| `successfulJobsHistoryLimit` | 3 | Keep last 3 completed runs for troubleshooting |
| `failedJobsHistoryLimit` | 5 | Keep last 5 failures for investigation |
| `activeDeadlineSeconds` | 300 (5 min) | Kill if stuck longer than 5 minutes |
| `restartPolicy` | `OnFailure` | Retry within the same schedule window |

### Single Binary, Multiple Commands

All tasks share a single Go binary with subcommands:

```
az-rotator-system job-reconcile
az-rotator-system retry-promote
az-rotator-system dead-letter-promote
az-rotator-system dlq-drain
az-rotator-system worker-health
az-rotator-system heartbeat-cleanup
az-rotator-system audit-cleanup
```

Each CronJob manifest just calls the binary with a different subcommand. This simplifies deployment — one container image, seven CronJobs.

---

## Consolidated vs Separate: Why Separate CronJobs?

We could run all tasks in a single "janitor" CronJob that does everything every 5 minutes. Why not?

| Separate | Single |
|----------|--------|
| Each task has its own schedule (some need 2 min, some daily) | Everything runs at the slowest common denominator |
| One task failure doesn't block others | One panic kills all maintenance |
| Easy to disable one task without affecting others | All or nothing |
| Clear observability (which task failed?) | Logs mixed together |
| Independent scaling and timeout | One timeout for everything |

The cost is more CronJob manifests, but that's a one-time setup.

---

## How Tasks Interact With Each Other

The tasks form a **pipeline** for handling failures:

```
                                    Happy Path
                                    ━━━━━━━━━━
Job Created → PENDING → PUBLISHED → RUNNING → COMPLETED ✓
                                       │
                                       │ (worker fails)
                                       ▼
                                     FAILED
                                       │
                    ┌──────────────────┤
                    │                  │
                    ▼                  ▼
            attempt < max        attempt >= max
                    │                  │
                    ▼                  ▼
           Retry Promoter      Dead-Letter Promoter
                    │                  │
                    ▼                  ▼
               PENDING            DEAD_LETTERED
              (retry cycle)       (terminal — needs
                                   human investigation)


        ┌──────────────────────────────────────┐
        │ Meanwhile, in parallel:               │
        │                                      │
        │ Job Reconciler:                      │
        │   Stuck PUBLISHED? → reset to PENDING│
        │   Stuck RUNNING? → mark FAILED       │
        │                                      │
        │ DLQ Drainer:                         │
        │   Service Bus DLQ? → mark FAILED     │
        │                                      │
        │ Worker Health:                       │
        │   No heartbeat? → SUSPEND worker     │
        └──────────────────────────────────────┘
```

**Ordering doesn't matter** — each task is idempotent and looks at current DB state. Running them in any order produces correct results.

---

## Idempotency

Every task is safe to run multiple times:

- If a job is already DEAD_LETTERED, the dead-letter promoter skips it
- If a DLQ message's job is already FAILED, the DLQ drainer just removes the message
- If a worker is already SUSPENDED, the health monitor skips it
- If heartbeats are already deleted, the cleanup exits cleanly

This matters because CronJobs can occasionally run twice (clock skew, scheduler hiccup).

---

## Configuration

All tasks share config via environment variables:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DATABASE_URL` | yes | — | PostgreSQL connection (Azure AD auth) |
| `SERVICE_BUS_NAMESPACE` | yes | — | For DLQ draining |
| `PUBLISHED_STALE_THRESHOLD` | no | `1h` | How long PUBLISHED before considered stale |
| `RUNNING_STALE_THRESHOLD` | no | `30m` | How long RUNNING before considered stale |
| `VALIDATING_STALE_THRESHOLD` | no | `15m` | How long VALIDATING before considered stale |
| `HEARTBEAT_TIMEOUT` | no | `5m` | No heartbeat for this long → worker SUSPENDED |
| `HEARTBEAT_RETENTION` | no | `30d` | Delete heartbeats older than this |
| `AUDIT_RETENTION` | no | `90d` | Delete audit logs older than this |
| `DLQ_BATCH_SIZE` | no | `50` | Max DLQ messages to process per tenant per run |
| `LOG_LEVEL` | no | `info` | Structured log level |

---

## Go Project Structure

```
cmd/
  system-tasks/
    main.go                     ← Entry point: subcommand dispatch

internal/
  systemtasks/
    reconciler.go               ← Job Reconciler logic
    retry.go                    ← Retry Promoter logic
    deadletter.go               ← Dead-Letter Promoter logic
    dlq.go                      ← DLQ Drainer (Service Bus reads)
    health.go                   ← Worker Health Monitor
    heartbeat_cleanup.go        ← Heartbeat retention cleanup
    audit_cleanup.go            ← Audit log retention cleanup

  store/
    (shared with API — same query layer, different DB role)

  config/
    config.go                   ← Environment variable loading
```

The `internal/store` package is shared across all services. System tasks use the same query functions but connect with the `az_rotator_system` role (bypasses RLS).

---

## Security Posture

| Concern | Mitigation |
|---------|-----------|
| Cross-tenant access | Required — system tasks need to reconcile all tenants |
| DB role | `az_rotator_system` — bypasses RLS but uses same Workload Identity auth |
| Service Bus access | `Data Receiver` at namespace level (reads DLQ only) |
| No external API calls | System tasks never call APIM — only DB and Service Bus directly |
| Pod security | Same as all pods: non-root, read-only fs, dropped caps, no priv esc |
| Secrets | None stored — Workload Identity for DB and Service Bus auth |

---

## Observability

### Key Metrics

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| Jobs recycled (PUBLISHED → PENDING) | Reconciler logs | > 5 per run → worker may be dead |
| Jobs marked FAILED by reconciler | Reconciler logs | Any → investigate worker health |
| Jobs promoted for retry | Retry promoter logs | Trending up → systemic rotation failures |
| Jobs dead-lettered | Dead-letter promoter logs | Any → tenant needs notification |
| DLQ messages processed | DLQ drainer logs | > 0 → something is failing |
| Workers suspended by health monitor | Health monitor logs | Any → check Container App status |
| Heartbeat records cleaned | Cleanup logs | Expected count vs actual (sanity check) |
| Task run duration | Pod metrics | > 60s → investigate (should be fast) |
| Task failure (non-zero exit) | Kubernetes events | Any → investigate |

### Structured Logging

Each task logs structured JSON:

```
{
  "task": "job-reconciler",
  "run_id": "uuid",
  "action": "recycle_published",
  "job_id": "uuid",
  "tenant_id": "uuid",
  "reason": "published_stale",
  "age_minutes": 67,
  "timestamp": "2026-05-20T10:05:00Z"
}
```

---

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| CronJobs (not long-running) | Periodic execution | Simple, resource-efficient, no idle overhead |
| Separate CronJobs per task | Independent schedules and failure isolation | One broken task doesn't block everything |
| Single binary, subcommands | One container image, many CronJobs | Simple build/deploy, consistent versioning |
| `concurrencyPolicy: Forbid` | No overlap | Prevents duplicate state transitions |
| Bypasses RLS | Cross-tenant reconciliation | Cannot be scoped to one tenant — must see all |
| Idempotent | Safe to run twice | CronJob scheduling isn't perfectly precise |
| DB state is authoritative | Jobs tracked in PostgreSQL | Service Bus is transport, DB is truth |
| DLQ drain syncs to DB | Keeps DB consistent with Service Bus | Without this, DB and SB can drift apart |
