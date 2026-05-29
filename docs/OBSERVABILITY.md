# Observability — Detailed Design

## Overview

The observability stack answers three questions at all times:
1. **Is the platform healthy?** — Are services running, are jobs flowing, are workers alive?
2. **What went wrong?** — When a rotation fails, what was the chain of events?
3. **Are we meeting SLAs?** — How many secrets are overdue, what's the rotation success rate?

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│  Sources                                                                    │
│                                                                             │
│  ┌─────────┐  ┌───────────┐  ┌──────────────┐  ┌────────┐  ┌───────────┐ │
│  │ API     │  │ Scheduler │  │ System Tasks │  │ Worker │  │ APIM      │ │
│  │ Service │  │           │  │ (CronJobs)   │  │        │  │ Gateway   │ │
│  └────┬────┘  └─────┬─────┘  └──────┬───────┘  └───┬────┘  └─────┬─────┘ │
│       │              │               │              │              │        │
│       ▼              ▼               ▼              ▼              ▼        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    OpenTelemetry Collector                           │   │
│  │                    (DaemonSet in AKS)                               │   │
│  └───────────┬──────────────────┬──────────────────┬──────────────────┘   │
│              │                  │                  │                        │
└──────────────┼──────────────────┼──────────────────┼────────────────────────┘
               │                  │                  │
               ▼                  ▼                  ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐
│ Azure Monitor /  │  │ Log Analytics    │  │ Application Insights │
│ Prometheus       │  │ Workspace        │  │ (APM)                │
│ (metrics)        │  │ (logs)           │  │ (traces + deps)      │
└────────┬─────────┘  └────────┬─────────┘  └──────────┬───────────┘
         │                     │                        │
         ▼                     ▼                        ▼
┌────────────────────────────────────────────────────────────────────┐
│                    Azure Managed Grafana                            │
│                    (dashboards + alerts)                            │
└────────────────────────────────────────────────────────────────────┘
```

---

## Three Pillars

### 1. Metrics — "What's happening now"

Numeric measurements collected at regular intervals. Used for dashboards, alerts, and capacity planning.

**Storage**: Azure Monitor Metrics (for Azure-native resources) + Prometheus (for app-level metrics scraped from pods).

### 2. Logs — "What happened and why"

Structured JSON log events emitted by every service. Used for debugging, audit trails, and root cause analysis.

**Storage**: Azure Log Analytics Workspace (centralized, queryable via KQL).

### 3. Traces — "How a request flowed through the system"

Distributed traces that follow a rotation job from scheduler → Service Bus → worker → Key Vault.

**Storage**: Application Insights (backed by Log Analytics).

---

## OpenTelemetry Instrumentation

All Go services emit telemetry via the **OpenTelemetry SDK** (otel-go). This gives us vendor-neutral instrumentation with Azure-native export.

### SDK Setup (Common Across Services)

```go
// internal/observability/otel.go

func Init(ctx context.Context, serviceName string) (shutdown func(), err error) {
    // Resource: identifies this service instance
    res := resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceName(serviceName),
        semconv.ServiceVersion(version.Get()),
        attribute.String("environment", os.Getenv("ENVIRONMENT")),
    )

    // Trace exporter → OTLP (to collector)
    traceExporter, _ := otlptracegrpc.New(ctx)
    tp := trace.NewTracerProvider(
        trace.WithBatcher(traceExporter),
        trace.WithResource(res),
    )
    otel.SetTracerProvider(tp)

    // Metric exporter → OTLP (to collector)
    metricExporter, _ := otlpmetricgrpc.New(ctx)
    mp := metric.NewMeterProvider(
        metric.WithReader(metric.NewPeriodicReader(metricExporter, metric.WithInterval(30*time.Second))),
        metric.WithResource(res),
    )
    otel.SetMeterProvider(mp)

    // Log exporter → OTLP (to collector)
    // Structured logs also written to stdout for Log Analytics container log capture

    return func() {
        tp.Shutdown(ctx)
        mp.Shutdown(ctx)
    }, nil
}
```

### OTel Collector Configuration

The collector runs as a DaemonSet in AKS, receives OTLP from all pods, and exports to Azure:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 5s
    limit_mib: 512

exporters:
  azuremonitor:
    connection_string: ${APPLICATIONINSIGHTS_CONNECTION_STRING}
  prometheusremotewrite:
    endpoint: ${AZURE_MONITOR_METRICS_ENDPOINT}

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, memory_limiter]
      exporters: [azuremonitor]
    metrics:
      receivers: [otlp]
      processors: [batch, memory_limiter]
      exporters: [prometheusremotewrite]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [azuremonitor]
```

---

## Metrics Catalog

### API Service

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `api.request.count` | Counter | `method`, `path`, `status_code` | Total HTTP requests |
| `api.request.duration_ms` | Histogram | `method`, `path`, `status_code` | Request latency |
| `api.request.active` | UpDownCounter | — | Currently in-flight requests |
| `api.auth.failures` | Counter | `reason` (`expired`, `invalid_aud`, `missing_role`) | Auth rejection count |
| `api.worker.registrations` | Counter | `tenant_id` | Worker registrations received |
| `api.worker.approvals` | Counter | `tenant_id` | Workers approved |
| `api.jobs.created` | Counter | `tenant_id`, `source` (`manual`, `scheduler`) | Jobs created |
| `api.db.query_duration_ms` | Histogram | `query_name` | Database query latency |
| `api.db.pool.active` | Gauge | — | Active DB connections |
| `api.db.pool.idle` | Gauge | — | Idle DB connections |

### Scheduler

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `scheduler.create.secrets_scanned` | Counter | — | Secrets evaluated per cycle |
| `scheduler.create.jobs_created` | Counter | `tenant_id` | Jobs created |
| `scheduler.create.skipped_no_worker` | Counter | `tenant_id` | Skipped — worker unhealthy |
| `scheduler.publish.jobs_published` | Counter | `tenant_id` | Jobs published to Service Bus |
| `scheduler.publish.batch_size` | Histogram | `tenant_id` | Messages per batch |
| `scheduler.publish.latency_ms` | Histogram | `tenant_id` | Publish latency |
| `scheduler.publish.errors` | Counter | `tenant_id`, `error_type` | Publish failures |
| `scheduler.loop.duration_ms` | Histogram | `loop` | Loop iteration time |
| `scheduler.loop.last_success` | Gauge | `loop` | Unix timestamp of last success |

### Worker (Container App)

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `worker.rotation.count` | Counter | `tenant_id`, `secret_type`, `status` | Rotations attempted |
| `worker.rotation.duration_ms` | Histogram | `secret_type` | End-to-end rotation time |
| `worker.rotation.plugin_duration_ms` | Histogram | `secret_type` | Plugin execution time only |
| `worker.rotation.vault_write_ms` | Histogram | — | Key Vault write latency |
| `worker.rotation.validation_ms` | Histogram | `secret_type` | Validation step latency |
| `worker.heartbeat.sent` | Counter | — | Heartbeats sent |
| `worker.heartbeat.failures` | Counter | — | Failed heartbeat reports |
| `worker.servicebus.messages_received` | Counter | — | Messages received from SB |
| `worker.servicebus.messages_completed` | Counter | — | Messages completed (success) |
| `worker.servicebus.messages_abandoned` | Counter | — | Messages abandoned (retry) |
| `worker.servicebus.messages_deadlettered` | Counter | — | Messages dead-lettered |

### System Tasks (CronJobs)

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `systemtask.run.duration_ms` | Histogram | `task_name` | Task execution time |
| `systemtask.run.items_processed` | Counter | `task_name` | Items handled per run |
| `systemtask.run.errors` | Counter | `task_name` | Errors per run |
| `systemtask.jobs.recycled` | Counter | — | Stale jobs reset to PENDING |
| `systemtask.jobs.retried` | Counter | — | Failed jobs promoted for retry |
| `systemtask.jobs.dead_lettered` | Counter | — | Jobs permanently dead-lettered |
| `systemtask.workers.suspended` | Counter | — | Workers marked SUSPENDED |
| `systemtask.dlq.messages_drained` | Counter | `tenant_id` | DLQ messages processed |

### Azure-Native Metrics (Service Bus)

Collected automatically by Azure Monitor — no instrumentation needed:

| Metric | Source | Description |
|--------|--------|-------------|
| `ActiveMessages` | Service Bus | Messages waiting in subscription |
| `DeadLetteredMessages` | Service Bus | Messages in DLQ |
| `IncomingMessages` | Service Bus | Messages published |
| `OutgoingMessages` | Service Bus | Messages delivered to consumers |
| `ServerErrors` | Service Bus | 5xx errors from SB |
| `ThrottledRequests` | Service Bus | Rate limit hits |

### Azure-Native Metrics (PostgreSQL)

| Metric | Source | Description |
|--------|--------|-------------|
| `cpu_percent` | PostgreSQL | Server CPU usage |
| `memory_percent` | PostgreSQL | Server memory usage |
| `active_connections` | PostgreSQL | Current connection count |
| `storage_percent` | PostgreSQL | Disk usage |
| `network_bytes_ingress/egress` | PostgreSQL | Network throughput |

---

## Structured Logging

All services use Go's `slog` package with JSON output. Every log line includes:

### Common Fields (Every Log Line)

| Field | Source | Example |
|-------|--------|---------|
| `level` | slog | `info`, `warn`, `error` |
| `msg` | code | `"job published"` |
| `service` | config | `api`, `scheduler`, `worker`, `system-tasks` |
| `timestamp` | slog | `2026-05-20T14:30:00.123Z` |
| `trace_id` | OTel context | `abc123def456...` (links log to trace) |
| `span_id` | OTel context | `789xyz...` |

### Service-Specific Fields

**API**:
- `method`, `path`, `status_code`, `duration_ms`, `caller_oid`, `tenant_id`, `request_id`

**Scheduler**:
- `loop` (`create`/`publish`), `tenant_id`, `job_id`, `duration_ms`, `batch_size`

**Worker**:
- `tenant_id`, `job_id`, `secret_type`, `phase` (`rotate`/`validate`/`vault_write`), `duration_ms`

**System Tasks**:
- `task_name`, `items_found`, `items_processed`, `duration_ms`

### Log Levels

| Level | When |
|-------|------|
| `debug` | Verbose: individual query results, message details (disabled in prod) |
| `info` | Normal operations: job created, rotation complete, cycle finished |
| `warn` | Recoverable issues: heartbeat failed, publish retry, worker approaching timeout |
| `error` | Failures: rotation failed, DB unreachable, Service Bus auth error |

### Example Log Lines

```json
{"level":"info","service":"scheduler","loop":"publish","msg":"batch published","tenant_id":"abc-123","topic":"tenant-team-platform-prod-rotations","count":3,"duration_ms":85,"trace_id":"4bf92f3577b34da6a3ce929d0e0e4736"}
```

```json
{"level":"error","service":"worker","msg":"rotation failed","tenant_id":"abc-123","job_id":"def-456","secret_type":"postgresql","phase":"rotate","error":"connect to postgres: dial tcp timeout","duration_ms":30012,"trace_id":"7a8b9c0d1e2f3a4b"}
```

```json
{"level":"warn","service":"system-tasks","task_name":"worker_health_monitor","msg":"worker suspended","tenant_id":"abc-123","worker_id":"ghi-789","last_heartbeat":"2026-05-20T14:20:00Z","reason":"heartbeat_timeout"}
```

---

## Log Collection Pipeline

```
Pod stdout (JSON)
    │
    ▼
AKS Container Insights agent (DaemonSet)
    │
    ▼
Azure Log Analytics Workspace
    │
    ├── ContainerLogV2 table (all pod logs)
    ├── KQL queries for filtering
    └── Workbook dashboards
```

**Container Apps (Worker)**: Worker logs flow to the Container App's Log Analytics workspace (can be the same workspace or federated). Container Apps have built-in log streaming to Log Analytics.

### Azure Resource Diagnostic Logs

| Resource | Diagnostic Categories | Destination |
|----------|----------------------|-------------|
| APIM | `GatewayLogs`, `WebSocketConnectionLogs` | Log Analytics |
| Service Bus | `OperationalLogs`, `RuntimeAuditLogs`, `VNetAndIPFilteringLogs` | Log Analytics |
| PostgreSQL | `PostgreSQLLogs`, `PostgreSQLFlexQueryStoreRuntime` | Log Analytics |
| Key Vault | `AuditEvent` | Log Analytics |
| AKS | `kube-apiserver`, `kube-controller-manager`, `kube-scheduler` | Log Analytics |

All diagnostic settings configured via Terraform — every Azure resource sends its logs to the same Log Analytics workspace.

---

## Distributed Tracing

### Trace Propagation

A rotation job creates a trace that spans multiple services:

```
Trace: rotation of "prod-db-password" (job abc-123)

├── Span: scheduler.publish (scheduler)
│   ├── db.query (find pending jobs)
│   ├── servicebus.send (publish message)
│   └── db.update (mark PUBLISHED)
│
├── Span: worker.process (worker) ← trace_id propagated via SB message properties
│   ├── api.report_status (RUNNING)
│   ├── plugin.rotate (postgresql)
│   │   ├── db.connect (to target PostgreSQL)
│   │   └── db.exec (ALTER ROLE)
│   ├── vault.write (Key Vault SetSecret)
│   ├── plugin.validate
│   │   ├── db.connect (with new password)
│   │   └── db.exec (SELECT 1)
│   └── api.report_status (COMPLETED)
│
└── Span: api.status_update (API)
    ├── db.update (job → COMPLETED)
    └── db.update (secret → next_rotation_at)
```

### Trace Context Propagation

The `trace_id` is passed through Service Bus message application properties:

```go
// Scheduler: inject trace context into message
msg.ApplicationProperties["traceparent"] = propagation.TraceContext{}.Inject(ctx)

// Worker: extract trace context from message
ctx = propagation.TraceContext{}.Extract(ctx, msg.ApplicationProperties)
```

This links the scheduler's publish span to the worker's processing span — a single trace covers the entire job lifecycle.

---

## Application Insights

Application Insights provides:
- **Live Metrics** — real-time view of requests, failures, dependencies
- **Application Map** — visual dependency graph (API → DB, Scheduler → SB, Worker → KV)
- **Failure Analysis** — group errors by type, see stack traces
- **Performance** — slow requests, dependency bottlenecks
- **Availability** — external URL probes (APIM health endpoint)

### Connection

All telemetry routes through the OTel Collector which exports to Application Insights via its connection string. No direct App Insights SDK needed — OTel handles export.

```
APPLICATIONINSIGHTS_CONNECTION_STRING=InstrumentationKey=xxx;IngestionEndpoint=https://eastus-1.in.applicationinsights.azure.com/
```

---

## Dashboards (Azure Managed Grafana)

### Dashboard: Platform Health

| Panel | Data Source | Visualization |
|-------|-------------|---------------|
| API request rate | Prometheus | Time series |
| API error rate (4xx, 5xx) | Prometheus | Time series + threshold |
| API latency (p50, p95, p99) | Prometheus | Time series |
| Scheduler loop status | Prometheus (`loop.last_success`) | Stat (time since last) |
| DB connection pool usage | Prometheus | Gauge |
| Service Bus active messages (all tenants) | Azure Monitor | Time series |
| Pod restart count | Prometheus | Stat |

### Dashboard: Rotation Status

| Panel | Data Source | Visualization |
|-------|-------------|---------------|
| Jobs by status (PENDING/PUBLISHED/RUNNING/COMPLETED/FAILED) | PostgreSQL (via Grafana Postgres plugin) | Pie chart |
| Rotation success rate (24h) | Prometheus | Stat (percentage) |
| Rotation latency distribution | Prometheus | Histogram heat map |
| Failed rotations by secret_type | Prometheus | Bar chart |
| Dead-lettered jobs (last 7 days) | PostgreSQL | Table |
| Overdue secrets count | PostgreSQL | Stat (alert if > 0) |

### Dashboard: Tenant Overview

| Panel | Data Source | Visualization |
|-------|-------------|---------------|
| Tenant list with health status | PostgreSQL | Table |
| Worker status per tenant | PostgreSQL | Status map |
| Secrets per tenant | PostgreSQL | Bar chart |
| Rotation history per tenant (selectable) | PostgreSQL | Time series |
| DLQ depth per tenant | Azure Monitor (Service Bus) | Bar chart |

### Dashboard: Worker Health

| Panel | Data Source | Visualization |
|-------|-------------|---------------|
| Worker heartbeat age | PostgreSQL (`NOW() - last_heartbeat_at`) | Table with thresholds |
| Worker rotation throughput | Prometheus | Time series per tenant |
| Worker error rate | Prometheus | Time series |
| Message processing latency | Prometheus | Histogram |
| Container App CPU/Memory | Azure Monitor (Container Apps) | Time series |

---

## Alerting Rules

### Critical (Page)

| Alert | Condition | Meaning |
|-------|-----------|---------|
| Scheduler loop stale | `scheduler.loop.last_success` > 5 min ago | Scheduler stopped, no jobs being published |
| API 5xx rate spike | > 10 errors in 5 min | API may be down or DB unreachable |
| DLQ depth growing | `DeadLetteredMessages` > 5 for any tenant | Worker can't process jobs — needs investigation |
| Worker suspended (heartbeat timeout) | `systemtask.workers.suspended` fires | Worker offline, tenant's rotations paused |
| DB connection pool exhausted | `api.db.pool.active` = max for > 2 min | API will start rejecting requests |

### Warning (Notify)

| Alert | Condition | Meaning |
|-------|-----------|---------|
| Publish errors | `scheduler.publish.errors` > 3 in 5 min | Service Bus connectivity issue |
| Rotation failures spike | `worker.rotation.count{status=failed}` > 5 in 1h | Something changed in target resources |
| Overdue secrets | Query: secrets where `next_rotation_at < NOW() - 1 day` | Rotation behind schedule |
| Service Bus active messages > 10 | Per-tenant subscription | Worker falling behind |
| CronJob failed | Kubernetes job failure event | System task didn't complete |
| API latency p99 > 2s | `api.request.duration_ms` p99 > 2000 | Performance degradation |

### Informational (Log Only)

| Alert | Condition | Meaning |
|-------|-----------|---------|
| Worker registered | `api.worker.registrations` fires | New worker awaiting approval |
| Manual rotation triggered | `api.jobs.created{source=manual}` | Admin triggered ad-hoc rotation |
| Policy updated | Audit log `policy.updated` | May affect rotation schedules |

---

## Log Analytics Queries (KQL)

### Recent Rotation Failures

```kql
ContainerLogV2
| where ContainerName == "worker"
| where LogMessage contains "rotation failed"
| extend parsed = parse_json(LogMessage)
| project TimeGenerated, 
    TenantId = tostring(parsed.tenant_id),
    JobId = tostring(parsed.job_id),
    SecretType = tostring(parsed.secret_type),
    Error = tostring(parsed.error)
| order by TimeGenerated desc
| take 50
```

### Scheduler Health (Last Hour)

```kql
ContainerLogV2
| where ContainerName == "scheduler"
| where TimeGenerated > ago(1h)
| extend parsed = parse_json(LogMessage)
| where parsed.msg == "cycle complete" or parsed.msg == "batch published"
| summarize 
    CyclesCompleted = countif(parsed.msg == "cycle complete"),
    JobsPublished = sumif(toint(parsed.count), parsed.msg == "batch published"),
    AvgDuration = avg(toint(parsed.duration_ms))
  by bin(TimeGenerated, 5m)
```

### Worker Heartbeat Gaps

```kql
ContainerLogV2
| where ContainerName has "system-tasks"
| extend parsed = parse_json(LogMessage)
| where parsed.msg == "worker suspended"
| project TimeGenerated,
    TenantId = tostring(parsed.tenant_id),
    WorkerId = tostring(parsed.worker_id),
    LastHeartbeat = todatetime(parsed.last_heartbeat),
    Reason = tostring(parsed.reason)
| order by TimeGenerated desc
```

### Dead-Letter Trend (7 Days)

```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SERVICEBUS"
| where Category == "RuntimeAuditLogs"
| where OperationName == "DeadLetter"
| summarize DLQCount = count() by bin(TimeGenerated, 1h), Resource
| render timechart
```

---

## Audit Log (Application-Level)

Separate from infrastructure logs — the `audit_logs` table in PostgreSQL records business-level events:

| Event | Who | What |
|-------|-----|------|
| `tenant.created` | Platform.Admin | New tenant created |
| `worker.registered` | Worker MI | Worker self-registered |
| `worker.approved` | Tenant Admin | Worker approved for duty |
| `worker.suspended` | System Task | Worker heartbeat timeout |
| `job.created` | Scheduler / Admin | Rotation job created |
| `job.published` | Scheduler | Job published to Service Bus |
| `job.completed` | Worker | Rotation succeeded |
| `job.failed` | Worker | Rotation failed |
| `job.dead_lettered` | System Task | Job permanently failed |
| `secret.registered` | Tenant Admin | New secret onboarded |
| `secret.policy_changed` | Tenant Admin | Secret assigned new policy |
| `policy.created` | Tenant Admin | New rotation policy |
| `policy.updated` | Tenant Admin | Policy settings changed |

Audit logs are **append-only** — never updated or deleted by application code. Retention managed by the Audit Log Cleanup CronJob (see [SYSTEM_TASKS.md](SYSTEM_TASKS.md)).

---

## Infrastructure (What Gets Deployed)

| Component | Deployment | Purpose |
|-----------|-----------|---------|
| OTel Collector | DaemonSet in AKS | Receive OTLP, export to Azure |
| Log Analytics Workspace | Terraform | Central log store |
| Application Insights | Terraform (attached to workspace) | APM, traces, live metrics |
| Azure Managed Grafana | Terraform | Dashboards, alerts |
| Diagnostic Settings | Terraform (per resource) | Route Azure resource logs to workspace |
| Container Insights | AKS add-on | Pod log collection |

### Terraform Resources

```hcl
resource "azurerm_log_analytics_workspace" "main" {
  name                = "la-az-rotator"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  sku                 = "PerGB2018"
  retention_in_days   = 90
}

resource "azurerm_application_insights" "main" {
  name                = "ai-az-rotator"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  workspace_id        = azurerm_log_analytics_workspace.main.id
  application_type    = "other"
}

resource "azurerm_dashboard_grafana" "main" {
  name                = "grafana-az-rotator"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  sku                 = "Standard"

  azure_monitor_workspace_integrations {
    resource_id = azurerm_monitor_workspace.main.id
  }
}
```

---

## What We're NOT Doing (MVP)

| Approach | Why Not |
|----------|---------|
| Self-hosted Prometheus + Grafana in cluster | Azure Managed Grafana with Azure Monitor data source handles this without ops burden |
| ELK stack for logs | Log Analytics is native, integrated with AKS, and queryable via KQL |
| Custom metrics pipeline | OTel + Azure Monitor covers it; no need for Datadog/New Relic |
| Per-tenant isolated observability | All telemetry in one workspace; tenant_id label for filtering |
| Secret values in logs | Hard rule — never log credentials. Only metadata (secret_name, vault_uri, job_id) |

---

## Data Retention

| Data | Store | Retention | Rationale |
|------|-------|-----------|-----------|
| Metrics (Prometheus) | Azure Monitor Metrics | 93 days | Azure default; sufficient for trend analysis |
| Logs (structured) | Log Analytics | 90 days | Configurable; balance cost vs debugging history |
| Traces (App Insights) | Log Analytics (backing) | 90 days | Same workspace, same retention |
| Audit logs (DB) | PostgreSQL `audit_logs` | Configurable (default 365 days) | Compliance — longer retention for business events |
| Heartbeat history (DB) | PostgreSQL `worker_heartbeats` | 30 days | Diagnostics only, cleaned by CronJob |
| Azure resource diagnostics | Log Analytics | 90 days | Matches main workspace retention |

---

## Summary

| Question | Answer |
|----------|--------|
| What collects telemetry from pods? | OTel Collector (DaemonSet) + Container Insights |
| Where are metrics stored? | Azure Monitor Metrics (Prometheus-compatible) |
| Where are logs stored? | Log Analytics Workspace |
| Where are traces stored? | Application Insights (backed by same workspace) |
| Where do dashboards live? | Azure Managed Grafana |
| Where do alerts fire? | Azure Monitor Alerts → Action Groups (email, Teams, PagerDuty) |
| What about Azure resource logs? | Diagnostic Settings → same Log Analytics Workspace |
| What about the audit trail? | PostgreSQL `audit_logs` table (365-day retention) |
| What's the instrumentation library? | OpenTelemetry Go SDK (otel-go) |
| Are secrets ever logged? | Never. Only metadata (names, IDs, types). |
