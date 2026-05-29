# Worker Agent — Detailed Design

## Overview

The worker is a **long-running Go process** deployed as an Azure Container App in the tenant's environment. It has private network access to the resources it rotates secrets for and to the Key Vault(s) where rotated values are stored.

Each tenant has exactly one worker. The worker:
1. Registers itself with the platform on first boot
2. Waits for admin approval
3. Subscribes to its tenant's Service Bus topic
4. Processes rotation jobs as they arrive
5. Reports status back to the control plane API
6. Sends periodic heartbeats

---

## Deployment Model

```
┌─────────────────────────────────────────────────────────────┐
│  Tenant's Azure Environment                                  │
│  (VNET / Subscription / Resource Group)                      │
│                                                              │
│  ┌────────────────────────────────────────┐                  │
│  │ Container App: az-rotator-worker        │                  │
│  │                                         │                  │
│  │  • System-assigned Managed Identity     │                  │
│  │  • VNET-integrated (access to private   │                  │
│  │    resources: DB servers, Key Vaults)   │                  │
│  │  • Single replica (1 worker per tenant) │                  │
│  │  • Min replicas: 1, Max: 1             │                  │
│  └────────────────────────────────────────┘                  │
│                                                              │
│  Private resources accessible:                               │
│  • PostgreSQL servers (private endpoint)                     │
│  • Key Vaults (private endpoint or public with RBAC)         │
│  • App Registrations (via MS Graph API)                      │
│  • Any resource the MI has RBAC on                           │
└─────────────────────────────────────────────────────────────┘
         │
         │  Outbound (HTTPS, public internet)
         │
         ├──▶ APIM Gateway (register, heartbeat, status reports)
         └──▶ Service Bus (receive rotation job messages)
```

**Key properties**:
- Runs in the **tenant's environment**, not the control plane
- Has **private network access** to resources the control plane cannot reach
- Authenticates everywhere with its **Managed Identity** — no secrets stored
- Communicates outbound to APIM and Service Bus over HTTPS

---

## Worker Lifecycle

### State Machine

```
NOT_DEPLOYED → STARTING → REGISTERING → WAITING_APPROVAL → RUNNING → SHUTTING_DOWN
                                              │
                                              ▼
                                         SUSPENDED (if admin suspends or heartbeat timeout)
```

These are runtime states internal to the worker process — not the same as the `workers.status` column in the database (which tracks `PENDING_APPROVAL`, `APPROVED`, `SUSPENDED`, etc. from the platform's perspective).

---

## Startup Flow

```
Worker Container starts
    │
    ▼
┌─────────────────────────────────────────┐
│ 1. Load configuration (env vars)         │
│    - TENANT_ID (platform tenant UUID)    │
│    - APIM_ENDPOINT (gateway URL)         │
│    - SERVICE_BUS_NAMESPACE               │
│    - WORKER_NAME                         │
│    - CAPABILITIES (plugin list)          │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ 2. Acquire MI token for platform API     │
│    azidentity.NewDefaultAzureCredential  │
│    Audience: platform App Registration   │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ 3. Register with platform                │
│    POST /api/workers/register            │
│    Headers:                              │
│      Authorization: Bearer {MI-JWT}      │
│      X-Tenant-ID: {tenant-uuid}          │
│    Body: { name, capabilities, region }  │
│                                          │
│    Response: 201 (first time)            │
│             409 (already registered)     │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ 4. Check registration status             │
│    If PENDING_APPROVAL:                  │
│      → Poll every 30s until APPROVED     │
│      → Log: "Waiting for admin approval" │
│    If APPROVED:                          │
│      → Proceed to subscribe              │
│    If REJECTED/SUSPENDED/DEACTIVATED:    │
│      → Log error, exit (non-zero)        │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ 5. Connect to Service Bus                │
│    Topic: tenant-{slug}-rotations        │
│    Subscription: worker                  │
│    Auth: MI (Data Receiver role)         │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ 6. Start background goroutines:          │
│    • Heartbeat reporter (every 60s)      │
│    • Message receiver loop               │
│    • Signal handler (graceful shutdown)  │
└─────────────────────────────────────────┘
```

### First Boot vs Subsequent Boots

| Scenario | Registration Call | Result |
|----------|-------------------|--------|
| First time ever | `POST /api/workers/register` → `201 Created` | Status: PENDING_APPROVAL. Worker waits. |
| Already registered, approved | `POST /api/workers/register` → `409 Conflict` | Worker checks status → APPROVED → proceed |
| Previously suspended | `POST /api/workers/register` → `409 Conflict` | Worker checks status → SUSPENDED → exit |

The worker is **idempotent on startup** — it always tries to register. If already registered, it just verifies its status and proceeds accordingly.

---

## Runtime: Message Receive Loop

Once approved and subscribed, the worker enters its main loop:

```
┌──────────────────────────────────────────────────────┐
│ Receive Loop (runs until shutdown signal)             │
│                                                      │
│  1. receiver.ReceiveMessages(ctx, 1, timeout: 30s)   │
│     → blocks until a message arrives or timeout      │
│                                                      │
│  2. If no message: loop back to 1 (keep listening)   │
│                                                      │
│  3. Message received:                                │
│     a. Parse message body → RotationJob              │
│     b. Validate: is this a known secret_type?        │
│     c. Report RUNNING to API                         │
│     d. Select rotation plugin                        │
│     e. Execute rotation                              │
│     f. Write new secret to Key Vault                 │
│     g. Validate new credential (if required)         │
│     h. Report COMPLETED (or FAILED) to API           │
│     i. Complete (or abandon/dead-letter) message     │
│                                                      │
│  4. Loop back to 1                                   │
└──────────────────────────────────────────────────────┘
```

### Processing a Single Job (Step 3 Expanded)

```
Message received (PeekLock — 5 min timer starts)
    │
    ▼
┌─────────────────────────────────────────┐
│ a. Parse message body                    │
│    Extract: job_id, secret, vault, policy│
│    If parse fails → dead-letter message  │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ b. Select plugin based on secret_type    │
│    "postgresql" → PostgreSQLPlugin       │
│    "app_registration" → AppRegPlugin     │
│    "github_pat" → GitHubPATPlugin        │
│    Unknown → dead-letter message         │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ c. Report RUNNING                        │
│    POST /api/rotations/{jobId}/status    │
│    Body: { "status": "RUNNING" }         │
│    (If API unreachable, continue anyway  │
│     — rotation is more important)        │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ d. Execute plugin.Rotate(ctx, secret)    │
│    Plugin does:                          │
│    1. Generate new credential            │
│    2. Apply to target resource           │
│       (e.g. ALTER ROLE ... PASSWORD)     │
│    3. Return new credential value        │
│                                          │
│    If error → report FAILED, abandon msg │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ e. Write to Key Vault                    │
│    PUT secret to vault_uri/secret_name   │
│    Using MI auth (Key Vault Secrets      │
│    Officer role on the vault)            │
│                                          │
│    If error → report FAILED, abandon msg │
│    (Secret was rotated but not stored —  │
│     this is a critical failure)          │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ f. Validate (if policy requires it)      │
│    Report VALIDATING to API              │
│    Plugin.Validate(ctx, newCredential)   │
│    e.g. try connecting to DB with new pw │
│                                          │
│    If validation fails:                  │
│      → Report FAILED to API             │
│      → Abandon message (retry later)    │
│    If succeeds → continue               │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ g. Report COMPLETED                      │
│    POST /api/rotations/{jobId}/status    │
│    Body: {                               │
│      "status": "COMPLETED",              │
│      "result": {                         │
│        "rotated_at": "...",              │
│        "vault_version": "...",           │
│        "validated": true                 │
│      }                                   │
│    }                                     │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ h. Complete message                      │
│    receiver.CompleteMessage(ctx, msg)     │
│    → Message permanently removed from    │
│      subscription                        │
└─────────────────────────────────────────┘
```

---

## Rotation Plugin Interface

Plugins are the actual rotation logic for each secret type. The worker selects the right plugin based on `secret_type` from the job message.

### Interface Definition

```go
// Plugin is the interface every rotation plugin must implement
type Plugin interface {
    // Type returns the secret_type this plugin handles (e.g. "postgresql")
    Type() string

    // Rotate generates a new credential and applies it to the target resource.
    // Returns the new credential value (to be stored in Key Vault).
    // Must NOT store the credential itself — the worker handles Key Vault writes.
    Rotate(ctx context.Context, config RotationConfig) (*RotationResult, error)

    // Validate tests that the new credential works against the target resource.
    // Called only if the policy requires validation.
    Validate(ctx context.Context, config RotationConfig, newCredential string) error
}

// RotationConfig is the input to a plugin — everything it needs to perform rotation
type RotationConfig struct {
    ResourceID string         // Azure resource ID or external identifier
    Config     map[string]any // Plugin-specific config from secret_registrations.config
}

// RotationResult is what the plugin returns after a successful rotation
type RotationResult struct {
    NewCredential string            // The new secret value (written to Key Vault by worker)
    Metadata      map[string]string // Optional info: expiry, fingerprint, etc.
}
```

### Plugin Registry

Plugins register themselves at worker startup:

```go
// registry.go
type Registry struct {
    plugins map[string]Plugin // key = secret_type
}

func (r *Registry) Register(p Plugin) {
    r.plugins[p.Type()] = p
}

func (r *Registry) Get(secretType string) (Plugin, bool) {
    p, ok := r.plugins[secretType]
    return p, ok
}
```

The worker's `main.go` registers all built-in plugins:

```go
registry := plugin.NewRegistry()
registry.Register(postgresql.New(credential))
registry.Register(appreg.New(credential))
registry.Register(githubpat.New(httpClient))
registry.Register(keyvault.New(credential))
// ... etc
```

### Built-in Plugins (MVP)

| Plugin | secret_type | What It Rotates | How |
|--------|-------------|-----------------|-----|
| PostgreSQL | `postgresql` | Database user password | `ALTER ROLE ... PASSWORD` via pgx |
| App Registration | `app_registration` | Client secret on an Azure AD app | MS Graph API — add new credential, remove old |
| Key Vault Secret | `keyvault_regenerate` | Secrets that need regeneration (storage keys, etc.) | Calls Azure mgmt API to regenerate, stores result |
| GitHub PAT | `github_pat` | Fine-grained personal access token | GitHub API — create new token, revoke old |

### Plugin Auth

Each plugin uses the **worker's Managed Identity** to authenticate to the target resource. The tenant admin must grant the worker MI the necessary RBAC:

| Plugin | Required RBAC on Target |
|--------|------------------------|
| PostgreSQL | Azure AD auth enabled on DB server, MI has login permission |
| App Registration | `Application.ReadWrite.All` MS Graph permission (or app owner) |
| Key Vault Regenerate | Contributor on the resource (for regenerate calls) |
| GitHub PAT | PAT with admin:org scope, or GitHub App installation |

---

## Key Vault Write Step

After the plugin rotates a credential, the worker writes the new value to the registered Key Vault. This is separated from the plugin so that:
1. All plugins follow the same storage pattern
2. Key Vault RBAC is the same regardless of secret type
3. The vault write is explicitly tracked and can fail independently

```go
// vault_writer.go
func (w *VaultWriter) WriteSecret(ctx context.Context, vaultURI, secretName, value string) (string, error) {
    client, err := azsecrets.NewClient(vaultURI, w.credential, nil)
    if err != nil {
        return "", fmt.Errorf("vault client: %w", err)
    }

    resp, err := client.SetSecret(ctx, secretName, azsecrets.SetSecretParameters{
        Value: &value,
    }, nil)
    if err != nil {
        return "", fmt.Errorf("set secret: %w", err)
    }

    // Return the version ID for audit
    return resp.ID.Version(), nil
}
```

**Required RBAC on Key Vault**: Worker MI must have `Key Vault Secrets Officer` role.

---

## Heartbeat

A background goroutine sends heartbeats every 60 seconds:

```
┌─────────────────────────────────────────┐
│ Heartbeat goroutine (every 60s)          │
│                                          │
│  POST /api/workers/heartbeat             │
│  Headers:                                │
│    Authorization: Bearer {MI-JWT}        │
│    X-Tenant-ID: {tenant-uuid}            │
│  Body: {                                 │
│    "worker_version": "1.2.0",            │
│    "active_jobs": 1,                     │
│    "cpu_usage": 0.23,                    │
│    "memory_usage": 0.45                  │
│  }                                       │
│                                          │
│  If API unreachable:                     │
│    → Log warning, retry next cycle       │
│    → Don't crash — rotation is primary   │
│                                          │
│  If response = 403:                      │
│    → Worker has been suspended/deactivated│
│    → Initiate graceful shutdown          │
└─────────────────────────────────────────┘
```

**Why heartbeat matters**: The control plane uses heartbeats to detect dead workers. If no heartbeat for X minutes, the scheduler knows not to expect job processing and the system task can mark the worker SUSPENDED.

---

## Graceful Shutdown

When the worker receives SIGTERM (Container App scaling down, restart, or deployment):

```
SIGTERM received
    │
    ▼
┌─────────────────────────────────────────┐
│ 1. Stop receiving new messages           │
│    (cancel receive context)              │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ 2. Wait for current job to finish        │
│    (with timeout — e.g. 30 seconds)      │
│                                          │
│    If job finishes in time:              │
│      → Report COMPLETED/FAILED           │
│      → Complete/abandon message          │
│    If timeout hit:                       │
│      → Abandon message (lock expires,    │
│        Service Bus re-delivers later)    │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ 3. Stop heartbeat goroutine              │
│    (cancel heartbeat context)            │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ 4. Close Service Bus connection          │
│    Close HTTP clients                    │
│    Exit 0                                │
└─────────────────────────────────────────┘
```

**Key point**: If the worker is killed while processing, the message lock expires and Service Bus re-delivers. Nothing is lost — the worst case is the job runs twice (idempotent rotation handles this).

---

## Error Handling Strategy

| Error Type | Worker Action | Message Settlement | API Report |
|-----------|---------------|-------------------|------------|
| Unknown secret_type | Dead-letter immediately | `DeadLetterMessage` | Report FAILED with "unsupported plugin" |
| Plugin Rotate() fails | Abandon (retry later) | `AbandonMessage` | Report FAILED with error message |
| Key Vault write fails | Abandon (critical) | `AbandonMessage` | Report FAILED with "vault write failed" |
| Validation fails | Abandon (retry) | `AbandonMessage` | Report FAILED with "validation failed" |
| API unreachable (status report) | Log warning, continue | Complete if rotation succeeded | Skip (system task reconciles) |
| API returns 403 (suspended) | Initiate shutdown | Abandon current message | None (worker is suspended) |
| Message parse error | Dead-letter (bad data) | `DeadLetterMessage` | None (can't extract job_id) |
| Lock about to expire | Renew lock (if possible) | `RenewMessageLock` | None |

### Idempotent Rotation

Because messages can be re-delivered (lock expired, worker restart, abandon), rotation plugins should be **idempotent** — running the same rotation twice should not break things.

Strategies per plugin:
- **PostgreSQL**: Generate new password, `ALTER ROLE` — idempotent (last write wins)
- **App Registration**: Check if a recent credential already exists before creating a new one
- **Key Vault write**: `SetSecret` is idempotent — creates a new version each time (old versions preserved)

---

## Configuration

All configuration via environment variables:

| Variable | Required | Description |
|----------|----------|-------------|
| `TENANT_ID` | yes | Platform tenant UUID this worker belongs to |
| `TENANT_SLUG` | yes | Tenant slug (used to derive topic name) |
| `APIM_ENDPOINT` | yes | APIM gateway URL (e.g. `https://az-rotator.azure-api.net`) |
| `SERVICE_BUS_NAMESPACE` | yes | Service Bus namespace (e.g. `az-rotator-servicebus.servicebus.windows.net`) |
| `WORKER_NAME` | yes | Display name for registration |
| `CAPABILITIES` | no (default all) | Comma-separated plugin types (e.g. `postgresql,app_registration`) |
| `AZURE_REGION` | no | Region for registration metadata |
| `HEARTBEAT_INTERVAL` | no (default `60s`) | How often to send heartbeats |
| `RECEIVE_TIMEOUT` | no (default `30s`) | How long to wait for a message before looping |
| `SHUTDOWN_TIMEOUT` | no (default `30s`) | Max time to wait for current job on shutdown |
| `LOG_LEVEL` | no (default `info`) | Structured log level |

---

## Go Project Structure

```
cmd/
  worker/
    main.go                     ← Entry point: config, DI, plugin registry, start

internal/
  worker/
    worker.go                   ← Orchestrator: startup, run loop, shutdown
    receiver.go                 ← Service Bus message receiver
    processor.go                ← Job processing logic (plugin dispatch, vault write, status)
    heartbeat.go                ← Background heartbeat goroutine
    registration.go             ← Registration + approval polling logic

  plugin/
    plugin.go                   ← Plugin interface definition
    registry.go                 ← Plugin registry (type → implementation)
    postgresql/
      postgresql.go             ← PostgreSQL password rotation
    appreg/
      appreg.go                 ← Azure AD App Registration client secret rotation
    keyvault/
      keyvault.go               ← Key Vault secret regeneration
    githubpat/
      githubpat.go              ← GitHub PAT rotation

  vault/
    writer.go                   ← Key Vault secret writer (SetSecret)

  client/
    api.go                      ← HTTP client for APIM (register, heartbeat, status)
    auth.go                     ← MI token acquisition for API calls

  config/
    config.go                   ← Environment variable loading + validation
```

---

## Security Posture

| Concern | Mitigation |
|---------|-----------|
| Worker has no stored secrets | All auth via Managed Identity — no passwords, keys, or certs |
| Worker can't access other tenants | Service Bus RBAC scoped to its topic only |
| Worker can't access vaults it shouldn't | Key Vault RBAC must be explicitly assigned by tenant admin |
| Worker token is short-lived | MI tokens expire (default 1h), auto-refreshed by SDK |
| Compromised worker | Can only rotate secrets in its own tenant; admin can suspend via API |
| Worker impersonation | Registration requires MI JWT with specific OID — can't fake |
| Secrets in memory | New credential held in memory briefly, written to vault, then discarded — never logged |
| Container hardened | Non-root, read-only filesystem, no privilege escalation |

---

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| One worker per tenant | Simplicity | Matches 1:1 with environment — no coordination needed |
| Container App (not AKS pod) | Isolation | Worker runs in tenant's env, not control plane cluster |
| Plugins compiled in | Static binary | No dynamic loading — simpler, auditable, type-safe |
| Vault write separated from plugins | Consistency | All plugins use same Key Vault write path; plugin only generates the credential |
| Heartbeat independent of receive loop | Availability | Heartbeat still fires even if receive is blocked waiting |
| Graceful shutdown with timeout | Reliability | Finish current job if possible; abandon if not — Service Bus handles retry |
| MI for everything | Zero secrets | No connection strings, passwords, or API keys in worker config |
| Worker polls for approval | Simplicity | No push notification needed — just check every 30s on startup |
