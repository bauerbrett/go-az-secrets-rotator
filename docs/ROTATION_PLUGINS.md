# Rotation Plugins — Detailed Design

## Overview

Rotation plugins are the **execution engines** of the platform. Each plugin knows how to rotate one type of secret — generate a new credential, apply it to the target resource, and return the value for Key Vault storage.

Plugins are:
- **Compiled into the worker binary** — no dynamic loading, no runtime discovery
- **Stateless** — each invocation receives all context it needs via `RotationConfig`
- **Focused** — a plugin does exactly two things: rotate and validate
- **Unaware of Key Vault** — plugins return the new credential; the worker handles vault writes separately

---

## Plugin Interface

```go
package plugin

import "context"

// Plugin is the interface every rotation plugin must implement.
type Plugin interface {
    // Type returns the secret_type this plugin handles.
    // Must match the secret_registrations.secret_type value (e.g. "postgresql", "app_registration").
    Type() string

    // Rotate generates a new credential and applies it to the target resource.
    // The returned RotationResult contains the new secret value.
    // Must NOT write to Key Vault — the worker handles that.
    Rotate(ctx context.Context, req RotateRequest) (*RotateResult, error)

    // Validate tests that the new credential works against the target resource.
    // Called only if the policy has validation_required = true.
    // Should be a read-only, non-destructive check (e.g., connect and SELECT 1).
    Validate(ctx context.Context, req ValidateRequest) error
}
```

### Request & Result Types

```go
// RotateRequest is the input to a plugin's Rotate method.
type RotateRequest struct {
    // ResourceID is the Azure resource ID or external identifier.
    // e.g. "/subscriptions/.../providers/Microsoft.DBforPostgreSQL/flexibleServers/prod-db"
    ResourceID string

    // Config is plugin-specific configuration from secret_registrations.config JSONB.
    // The plugin knows its own schema and type-asserts fields from this map.
    Config map[string]any

    // SecretLifetime is how long the new credential should be valid.
    // Comes from the rotation policy. Plugins that support expiry should use this.
    SecretLifetime time.Duration

    // CurrentSecretName is the secret name in Key Vault (for plugins that need to
    // read the current value to perform rotation — e.g., key regeneration).
    // Empty if not applicable.
    CurrentSecretName string
}

// RotateResult is what the plugin returns after a successful rotation.
type RotateResult struct {
    // NewCredential is the new secret value to be written to Key Vault.
    NewCredential string

    // ExpiresAt is when the new credential expires (if applicable).
    // Used for audit logging and compliance reporting.
    ExpiresAt *time.Time

    // Metadata is optional key-value data about the rotation.
    // e.g., {"fingerprint": "abc123", "key_id": "...", "old_key_removed": "true"}
    Metadata map[string]string
}

// ValidateRequest is the input to a plugin's Validate method.
type ValidateRequest struct {
    ResourceID    string
    Config        map[string]any
    NewCredential string // The credential to validate
}
```

---

## Plugin Registry

Plugins register at worker startup. The registry maps `secret_type` → plugin implementation.

```go
package plugin

import "fmt"

// Registry holds all available rotation plugins.
type Registry struct {
    plugins map[string]Plugin
}

func NewRegistry() *Registry {
    return &Registry{plugins: make(map[string]Plugin)}
}

// Register adds a plugin to the registry. Panics on duplicate type.
func (r *Registry) Register(p Plugin) {
    if _, exists := r.plugins[p.Type()]; exists {
        panic(fmt.Sprintf("duplicate plugin registered for type: %s", p.Type()))
    }
    r.plugins[p.Type()] = p
}

// Get retrieves the plugin for a given secret type.
func (r *Registry) Get(secretType string) (Plugin, bool) {
    p, ok := r.plugins[secretType]
    return p, ok
}

// Types returns all registered secret types (for capability reporting).
func (r *Registry) Types() []string {
    types := make([]string, 0, len(r.plugins))
    for t := range r.plugins {
        types = append(types, t)
    }
    return types
}
```

### Worker Startup Registration

```go
// cmd/worker/main.go
func buildRegistry(cred azcore.TokenCredential, httpClient *http.Client) *plugin.Registry {
    reg := plugin.NewRegistry()

    reg.Register(postgresql.New(cred))
    reg.Register(appreg.New(cred))
    reg.Register(keyvault.New(cred))
    reg.Register(githubpat.New(httpClient))
    reg.Register(mysql.New(cred))
    reg.Register(storagekey.New(cred))

    return reg
}
```

The registry's `.Types()` method is used during worker registration — the worker reports its capabilities so the control plane knows what secret types it can handle.

---

## Plugin Dispatch Flow

When the worker receives a rotation job message:

```
Message received (secret_type = "postgresql")
    │
    ▼
┌─────────────────────────────────────────────────┐
│ 1. Look up plugin: registry.Get("postgresql")    │
│    → If not found: DeadLetterMessage             │
│      (worker can't handle this type)             │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│ 2. Build RotateRequest from message body:        │
│    - ResourceID = secret.resource_id             │
│    - Config = secret.config                      │
│    - SecretLifetime = policy.secret_lifetime     │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│ 3. Call plugin.Rotate(ctx, req)                  │
│    → Plugin generates new credential             │
│    → Plugin applies to target resource           │
│    → Returns RotateResult                        │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│ 4. Worker writes RotateResult.NewCredential      │
│    to Key Vault (vault_uri + secret_name)        │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│ 5. If policy.validation_required:                │
│    Call plugin.Validate(ctx, validateReq)         │
│    → Plugin tests new credential works           │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│ 6. Report COMPLETED to API                       │
│    CompleteMessage on Service Bus                 │
└─────────────────────────────────────────────────┘
```

---

## Built-in Plugins

### 1. PostgreSQL (`postgresql`)

**What it rotates**: Password for a PostgreSQL user/role on Azure Database for PostgreSQL Flexible Server.

**Rotation strategy**: Atomic swap — new password immediately invalidates the old one.

**Config schema** (`secret_registrations.config`):
```json
{
  "host": "prod-db.postgres.database.azure.com",
  "port": 5432,
  "username": "app_user",
  "database": "appdb",
  "password_length": 32
}
```

**Rotate logic**:
```go
func (p *PostgreSQLPlugin) Rotate(ctx context.Context, req plugin.RotateRequest) (*plugin.RotateResult, error) {
    cfg := parseConfig(req.Config) // host, port, username, database, password_length

    // 1. Generate new password (crypto/rand, no ambiguous chars)
    newPassword := generatePassword(cfg.PasswordLength)

    // 2. Connect to PostgreSQL with worker's MI (Azure AD auth)
    conn, err := pgx.Connect(ctx, buildDSN(cfg.Host, cfg.Port, cfg.Database, p.credential))
    if err != nil {
        return nil, fmt.Errorf("connect to postgres: %w", err)
    }
    defer conn.Close(ctx)

    // 3. Alter role password
    //    Uses SET ROLE or direct ALTER depending on permissions
    _, err = conn.Exec(ctx, fmt.Sprintf(
        "ALTER ROLE %s WITH PASSWORD %s VALID UNTIL %s",
        pgx.Identifier{cfg.Username}.Sanitize(),
        quoteLiteral(newPassword),
        quoteLiteral(expiryTime(req.SecretLifetime)),
    ))
    if err != nil {
        return nil, fmt.Errorf("alter role: %w", err)
    }

    expiry := time.Now().Add(req.SecretLifetime)
    return &plugin.RotateResult{
        NewCredential: newPassword,
        ExpiresAt:     &expiry,
        Metadata:      map[string]string{"username": cfg.Username, "host": cfg.Host},
    }, nil
}
```

**Validate logic**:
```go
func (p *PostgreSQLPlugin) Validate(ctx context.Context, req plugin.ValidateRequest) error {
    cfg := parseConfig(req.Config)

    // Connect with the NEW password (not MI — testing the credential itself)
    connStr := fmt.Sprintf("host=%s port=%d user=%s password=%s dbname=%s sslmode=require",
        cfg.Host, cfg.Port, cfg.Username, req.NewCredential, cfg.Database)

    conn, err := pgx.Connect(ctx, connStr)
    if err != nil {
        return fmt.Errorf("validation connect failed: %w", err)
    }
    defer conn.Close(ctx)

    // Simple health check
    _, err = conn.Exec(ctx, "SELECT 1")
    return err
}
```

**Required RBAC / Permissions**:
- Worker MI must be able to connect to PostgreSQL (Azure AD auth enabled, MI granted login)
- Worker MI must have permission to `ALTER ROLE` for the target username (typically requires `CREATEROLE` or superuser)

**Idempotency**: Last write wins — running `ALTER ROLE ... PASSWORD` twice with different values just means the second password is active. The Key Vault always stores the latest.

---

### 2. App Registration (`app_registration`)

**What it rotates**: Client secret on an Azure AD (Entra ID) App Registration.

**Rotation strategy**: Overlap — adds a new credential, old one expires naturally at its `secret_lifetime`.

**Config schema**:
```json
{
  "app_object_id": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
  "credential_description": "rotated-by-az-rotator"
}
```

**Rotate logic**:
```go
func (p *AppRegPlugin) Rotate(ctx context.Context, req plugin.RotateRequest) (*plugin.RotateResult, error) {
    cfg := parseConfig(req.Config)

    // 1. Create Microsoft Graph client with worker MI
    graphClient := p.newGraphClient()

    // 2. Add new client secret to app registration
    endDateTime := time.Now().Add(req.SecretLifetime)
    addReq := &graphmodels.PasswordCredential{
        DisplayName: &cfg.CredentialDescription,
        EndDateTime: &endDateTime,
    }

    result, err := graphClient.Applications().
        ByApplicationId(cfg.AppObjectID).
        AddPassword().
        Post(ctx, addReq, nil)
    if err != nil {
        return nil, fmt.Errorf("add password: %w", err)
    }

    // 3. Optionally remove old credentials that are near expiry
    //    (keep only the new one + any still within buffer window)
    p.cleanupExpiredCredentials(ctx, graphClient, cfg.AppObjectID, endDateTime)

    return &plugin.RotateResult{
        NewCredential: *result.GetSecretText(),
        ExpiresAt:     &endDateTime,
        Metadata: map[string]string{
            "key_id":    result.GetKeyId().String(),
            "app_id":    cfg.AppObjectID,
            "ends_at":   endDateTime.Format(time.RFC3339),
        },
    }, nil
}
```

**Validate logic**:
```go
func (p *AppRegPlugin) Validate(ctx context.Context, req plugin.ValidateRequest) error {
    cfg := parseConfig(req.Config)

    // Acquire token using client_id + new secret (client credentials flow)
    // The app's Application (client) ID is needed — derive from app_object_id
    // or require it in config
    cred, err := azidentity.NewClientSecretCredential(
        p.tenantID, cfg.ClientID, req.NewCredential, nil)
    if err != nil {
        return fmt.Errorf("create credential: %w", err)
    }

    // Try to get a token (proves the secret works)
    _, err = cred.GetToken(ctx, policy.TokenRequestOptions{
        Scopes: []string{"https://graph.microsoft.com/.default"},
    })
    return err
}
```

**Required RBAC / Permissions**:
- Worker MI must have `Application.ReadWrite.OwnedBy` or be an owner of the target application
- Or: MS Graph `Application.ReadWrite.All` (broader — allows rotating any app's secrets)

**Idempotency**: Adding a new credential is always safe — it creates a new key_id. If Rotate runs twice, two credentials are added; cleanup logic removes extras near expiry.

---

### 3. Storage Account Key (`storage_account_key`)

**What it rotates**: Access key (key1 or key2) on an Azure Storage Account.

**Rotation strategy**: Alternate keys — rotate key1, then next time rotate key2. Always one valid key.

**Config schema**:
```json
{
  "subscription_id": "...",
  "resource_group": "rg-prod",
  "account_name": "prodstore01",
  "key_name": "key1"
}
```

**Rotate logic**:
```go
func (p *StorageKeyPlugin) Rotate(ctx context.Context, req plugin.RotateRequest) (*plugin.RotateResult, error) {
    cfg := parseConfig(req.Config)

    // 1. Create ARM client with worker MI
    client, _ := armstorage.NewAccountsClient(cfg.SubscriptionID, p.credential, nil)

    // 2. Regenerate the specified key
    result, err := client.RegenerateKey(ctx, cfg.ResourceGroup, cfg.AccountName,
        armstorage.AccountRegenerateKeyParameters{
            KeyName: &cfg.KeyName,
        }, nil)
    if err != nil {
        return nil, fmt.Errorf("regenerate key: %w", err)
    }

    // 3. Find the regenerated key in the response
    for _, key := range result.Keys {
        if *key.KeyName == cfg.KeyName {
            return &plugin.RotateResult{
                NewCredential: *key.Value,
                Metadata: map[string]string{
                    "key_name":     cfg.KeyName,
                    "account_name": cfg.AccountName,
                },
            }, nil
        }
    }

    return nil, fmt.Errorf("key %s not found in response", cfg.KeyName)
}
```

**Validate logic**:
```go
func (p *StorageKeyPlugin) Validate(ctx context.Context, req plugin.ValidateRequest) error {
    cfg := parseConfig(req.Config)

    // Use the new key to list containers (proves it works)
    cred, _ := azblob.NewSharedKeyCredential(cfg.AccountName, req.NewCredential)
    client, _ := azblob.NewClientWithSharedKeyCredential(
        fmt.Sprintf("https://%s.blob.core.windows.net", cfg.AccountName), cred, nil)

    // Just try to list (max 1 result)
    pager := client.NewListContainersPager(&azblob.ListContainersOptions{MaxResults: to.Ptr(int32(1))})
    _, err := pager.NextPage(ctx)
    return err
}
```

**Required RBAC**: Worker MI needs `Storage Account Key Operator Service Role` on the storage account.

**Idempotency**: Regenerating the same key twice just gives a new value — safe to repeat.

**Note**: `SecretLifetime` is not directly applicable here (storage keys don't have expiry). The plugin ignores it. The rotation interval alone governs frequency.

---

### 4. GitHub PAT (`github_pat`)

**What it rotates**: Fine-grained Personal Access Token (or classic PAT via GitHub Apps).

**Rotation strategy**: Overlap — create new token, old one expires naturally at its `secret_lifetime`. No revocation.

**Config schema**:
```json
{
  "owner": "my-org",
  "github_username": "svc-deploy-bot",
  "token_name": "az-rotator-deploy",
  "permissions": {
    "contents": "read",
    "metadata": "read",
    "actions": "write"
  },
  "repositories": ["repo-1", "repo-2"],
  "github_app_installation_id": "12345678"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `owner` | yes | GitHub org or user that owns the repositories |
| `github_username` | yes | The GitHub user the token is generated for (the identity the PAT belongs to) |
| `token_name` | yes | Display name for the token in GitHub |
| `permissions` | yes | Fine-grained permission set for the new token |
| `repositories` | no | Scope to specific repos (empty = all repos the user can access) |
| `github_app_installation_id` | yes | GitHub App installation that has permission to manage tokens on behalf of the user |

**Rotate logic**:
```go
func (p *GitHubPATPlugin) Rotate(ctx context.Context, req plugin.RotateRequest) (*plugin.RotateResult, error) {
    cfg := parseConfig(req.Config)

    // 1. Authenticate as GitHub App installation
    appClient := p.getInstallationClient(cfg.GitHubAppInstallationID)

    // 2. Create new fine-grained token via GitHub API
    //    Old token continues to work until its expiry (overlap period)
    expiry := time.Now().Add(req.SecretLifetime)
    token, err := appClient.CreateToken(ctx, &github.CreateTokenRequest{
        Owner:        cfg.Owner,
        User:         cfg.GitHubUsername,
        TokenName:    cfg.TokenName,
        Permissions:  cfg.Permissions,
        Repositories: cfg.Repositories,
        ExpiresAt:    expiry,
    })
    if err != nil {
        return nil, fmt.Errorf("create token: %w", err)
    }

    return &plugin.RotateResult{
        NewCredential: token.Token,
        ExpiresAt:     &expiry,
        Metadata: map[string]string{
            "token_id": token.ID,
            "owner":    cfg.Owner,
            "user":     cfg.GitHubUsername,
        },
    }, nil
}
```

**Validate logic**:
```go
func (p *GitHubPATPlugin) Validate(ctx context.Context, req plugin.ValidateRequest) error {
    // Verify token by calling GET /user or GET /installation
    client := github.NewClient(nil).WithAuthToken(req.NewCredential)
    _, _, err := client.Users.Get(ctx, "")
    return err
}
```

**Required permissions**: A GitHub App installation with permission to manage fine-grained tokens for the organization. Worker MI is used to authenticate to Key Vault to fetch the GitHub App private key for auth.

**Idempotency**: Creating a new token is always safe. If called twice, you get two tokens — but only the last Key Vault write persists. Old tokens expire gracefully at their `SecretLifetime`.

---

### 5. MySQL (`mysql`)

**What it rotates**: Password for a MySQL user on Azure Database for MySQL Flexible Server.

**Rotation strategy**: Atomic swap — same as PostgreSQL.

**Config schema**:
```json
{
  "host": "prod-mysql.mysql.database.azure.com",
  "port": 3306,
  "username": "app_user",
  "database": "appdb",
  "password_length": 32
}
```

**Rotate logic**:
```go
func (p *MySQLPlugin) Rotate(ctx context.Context, req plugin.RotateRequest) (*plugin.RotateResult, error) {
    cfg := parseConfig(req.Config)

    newPassword := generatePassword(cfg.PasswordLength)

    // Connect with worker MI (Azure AD auth)
    db, err := sql.Open("mysql", buildMySQLDSN(cfg, p.credential))
    if err != nil {
        return nil, fmt.Errorf("connect: %w", err)
    }
    defer db.Close()

    // ALTER USER to set new password
    // MySQL: ALTER USER 'app_user'@'%' IDENTIFIED BY 'newpass'
    _, err = db.ExecContext(ctx, fmt.Sprintf(
        "ALTER USER %s@'%%' IDENTIFIED BY %s PASSWORD EXPIRE INTERVAL %d DAY",
        quoteIdentifier(cfg.Username),
        quoteLiteral(newPassword),
        int(req.SecretLifetime.Hours()/24),
    ))
    if err != nil {
        return nil, fmt.Errorf("alter user: %w", err)
    }

    expiry := time.Now().Add(req.SecretLifetime)
    return &plugin.RotateResult{
        NewCredential: newPassword,
        ExpiresAt:     &expiry,
        Metadata:      map[string]string{"username": cfg.Username, "host": cfg.Host},
    }, nil
}
```

**Validate**: Same pattern as PostgreSQL — connect with new password, run `SELECT 1`.

**Required RBAC**: Worker MI needs Azure AD auth setup on MySQL, plus `ALTER USER` privilege.

---

### 6. Databricks PAT (`databricks_pat`)

**What it rotates**: Databricks Personal Access Token.

**Rotation strategy**: Overlap — create new token, old one expires naturally at its `secret_lifetime`. No revocation.

**Config schema**:
```json
{
  "workspace_url": "https://adb-1234567890.azuredatabricks.net",
  "databricks_user": "svc-pipeline@contoso.com",
  "token_comment": "az-rotator-pipeline-token",
  "lifetime_seconds_override": null
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `workspace_url` | yes | Databricks workspace base URL |
| `databricks_user` | yes | The Databricks user/service principal the PAT is generated for (worker impersonates this identity) |
| `token_comment` | yes | Comment tag on the token (used to identify it in the workspace) |
| `lifetime_seconds_override` | no | Override `SecretLifetime` for this specific token (null = use policy value) |

**Rotate logic**:
```go
func (p *DatabricksPATPlugin) Rotate(ctx context.Context, req plugin.RotateRequest) (*plugin.RotateResult, error) {
    cfg := parseConfig(req.Config)

    // 1. Authenticate to Databricks workspace using worker MI
    //    (Azure AD token scoped to 2ff814a6-3304-4ab8-85cb-cd0e6f879c1d — Databricks resource ID)
    //    Impersonate the target user (databricks_user) for token creation
    client := p.getDatabricksClient(cfg.WorkspaceURL, cfg.DatabricksUser)

    // 2. Create new token — old token continues to work until its expiry
    lifetimeSec := int(req.SecretLifetime.Seconds())
    result, err := client.Tokens.Create(ctx, databricks.CreateTokenRequest{
        Comment:         cfg.TokenComment,
        LifetimeSeconds: lifetimeSec,
    })
    if err != nil {
        return nil, fmt.Errorf("create token: %w", err)
    }

    expiry := time.Now().Add(req.SecretLifetime)
    return &plugin.RotateResult{
        NewCredential: result.TokenValue,
        ExpiresAt:     &expiry,
        Metadata: map[string]string{
            "token_id":  result.TokenInfo.TokenID,
            "workspace": cfg.WorkspaceURL,
            "user":      cfg.DatabricksUser,
        },
    }, nil
}
```

**Validate**: Call `GET /api/2.0/clusters/list` or `/api/2.0/token/list` with new token.

**Required RBAC**: Worker MI needs `Contributor` on the Databricks workspace (for AAD token exchange), and the token's Databricks user identity must have workspace access.

---

## Plugin Configuration Schema

Each plugin defines its own config schema. The `secret_registrations.config` JSONB field stores plugin-specific data. The API service validates config on secret registration using a schema registry:

```go
// internal/api/validation/plugin_config.go

var pluginConfigSchemas = map[string]ConfigValidator{
    "postgresql":          postgresqlConfigValidator,
    "app_registration":    appRegConfigValidator,
    "storage_account_key": storageKeyConfigValidator,
    "github_pat":          githubPATConfigValidator,
    "mysql":               mysqlConfigValidator,
    "databricks_pat":      databricksConfigValidator,
}

type ConfigValidator func(config map[string]any) error

func postgresqlConfigValidator(config map[string]any) error {
    required := []string{"host", "port", "username", "database"}
    for _, key := range required {
        if _, ok := config[key]; !ok {
            return fmt.Errorf("missing required field: %s", key)
        }
    }
    return nil
}
```

When a user registers a secret (`POST /api/tenants/{tid}/secrets`), the API:
1. Looks up the validator for `secret_type`
2. Validates `config` against the schema
3. Returns `400 Bad Request` if invalid

This catches misconfiguration early — before the scheduler creates a job that would inevitably fail.

---

## Adding Custom Plugins

Adding a new plugin is straightforward:

### 1. Implement the Interface

```go
// internal/plugin/cosmosdb/cosmosdb.go
package cosmosdb

type Plugin struct {
    credential azcore.TokenCredential
}

func New(cred azcore.TokenCredential) *Plugin {
    return &Plugin{credential: cred}
}

func (p *Plugin) Type() string { return "cosmosdb" }

func (p *Plugin) Rotate(ctx context.Context, req plugin.RotateRequest) (*plugin.RotateResult, error) {
    // Regenerate Cosmos DB primary key via ARM API
    // Return the new key
}

func (p *Plugin) Validate(ctx context.Context, req plugin.ValidateRequest) error {
    // Connect to Cosmos DB with the new key
    // Run a simple query
}
```

### 2. Register in Worker Startup

```go
// cmd/worker/main.go
import "internal/plugin/cosmosdb"

reg.Register(cosmosdb.New(credential))
```

### 3. Add Config Validator in API

```go
// internal/api/validation/plugin_config.go
pluginConfigSchemas["cosmosdb"] = cosmosDBConfigValidator
```

### 4. Document Config Schema

The config schema should be documented so tenant admins know what to put in the `config` field when registering a secret of this type.

---

## Plugin Error Handling

Plugins should return errors that give the worker enough context to decide settlement strategy:

```go
// Recoverable errors (worker abandons message — retry later)
return nil, fmt.Errorf("connect to postgres: %w", err)  // transient

// Permanent errors (worker should dead-letter — no point retrying)
return nil, &plugin.PermanentError{Err: fmt.Errorf("app %s not found", appID)}
```

```go
// plugin/errors.go

// PermanentError indicates the rotation can never succeed without human intervention.
// The worker should dead-letter the message instead of retrying.
type PermanentError struct {
    Err error
}

func (e *PermanentError) Error() string { return e.Err.Error() }
func (e *PermanentError) Unwrap() error { return e.Err }

func IsPermanent(err error) bool {
    var pe *PermanentError
    return errors.As(err, &pe)
}
```

Worker settlement logic:

```go
result, err := plugin.Rotate(ctx, req)
if err != nil {
    if plugin.IsPermanent(err) {
        // No point retrying — dead-letter immediately
        receiver.DeadLetterMessage(ctx, msg, &azservicebus.DeadLetterOptions{
            Reason: to.Ptr("permanent_failure"),
        })
    } else {
        // Transient — abandon and let Service Bus retry
        receiver.AbandonMessage(ctx, msg, nil)
    }
    reportFailed(ctx, jobID, err)
    return
}
```

---

## Plugin Auth Patterns

All plugins authenticate to their target resource using the **worker's Managed Identity**. No plugin stores credentials.

| Plugin | Auth Method | Target |
|--------|-------------|--------|
| `postgresql` | Azure AD token (MI) | PostgreSQL Flexible Server |
| `mysql` | Azure AD token (MI) | MySQL Flexible Server |
| `app_registration` | Azure AD token (MI) → MS Graph | Microsoft Graph API |
| `storage_account_key` | Azure AD token (MI) → ARM | Azure Resource Manager |
| `github_pat` | GitHub App private key (from Key Vault) | GitHub API |
| `databricks_pat` | Azure AD token (MI) → Databricks | Databricks REST API |

For plugins that need credentials beyond MI (like GitHub App private key), the plugin reads them from Key Vault at rotation time — never stored in config.

---

## Capability Reporting

When the worker registers with the control plane, it reports which plugins it has:

```json
POST /api/workers/register
{
  "name": "worker-team-platform-prod",
  "capabilities": ["postgresql", "app_registration", "storage_account_key", "github_pat", "mysql", "databricks_pat"]
}
```

This allows the control plane to:
- Warn if a secret is registered with a `secret_type` that the tenant's worker can't handle
- Future: route jobs to capable workers (if multi-worker per tenant is ever added)

---

## Plugin Testing

Each plugin should have:

### Unit Tests
- Mock the target resource API (httptest for REST, pgxmock for PostgreSQL)
- Verify correct API calls are made
- Verify password generation meets requirements

### Integration Tests
- Run against real (test) resources in a dev subscription
- Tagged with `//go:build integration` — not run in CI by default
- Use short-lived test resources provisioned by Terraform

### Validation Tests
- Verify `Validate()` correctly detects working vs broken credentials
- Test timeout behavior (what if target is slow to respond)

---

## Go Project Structure

```
internal/
  plugin/
    plugin.go                     ← Plugin interface, RotateRequest, RotateResult
    registry.go                   ← Registry (map[string]Plugin)
    errors.go                     ← PermanentError, IsPermanent helper
    password.go                   ← Shared password generation (crypto/rand)

    postgresql/
      postgresql.go               ← Plugin implementation
      postgresql_test.go          ← Unit tests (pgxmock)
      config.go                   ← Config parsing + validation

    appreg/
      appreg.go                   ← Plugin implementation
      appreg_test.go              ← Unit tests (mock Graph client)
      config.go                   ← Config parsing
      cleanup.go                  ← Old credential cleanup logic

    storagekey/
      storagekey.go               ← Plugin implementation
      storagekey_test.go          ← Unit tests
      config.go                   ← Config parsing

    githubpat/
      githubpat.go                ← Plugin implementation
      githubpat_test.go           ← Unit tests
      config.go                   ← Config parsing

    mysql/
      mysql.go                    ← Plugin implementation
      mysql_test.go               ← Unit tests
      config.go                   ← Config parsing

    databricks/
      databricks.go               ← Plugin implementation
      databricks_test.go          ← Unit tests
      config.go                   ← Config parsing
```

---

## Summary Table

| Plugin | `secret_type` | Strategy | Uses `SecretLifetime` | Validation |
|--------|---------------|----------|----------------------|------------|
| PostgreSQL | `postgresql` | Atomic swap | Yes (`VALID UNTIL`) | Connect + `SELECT 1` |
| App Registration | `app_registration` | Overlap (add new, old expires) | Yes (`endDateTime`) | Client credentials token acquisition |
| Storage Account Key | `storage_account_key` | Key regeneration (alternate keys) | No (keys don't expire) | List containers with new key |
| GitHub PAT | `github_pat` | Overlap (create new, old expires) | Yes (token expiry) | `GET /user` |
| MySQL | `mysql` | Atomic swap | Yes (`PASSWORD EXPIRE INTERVAL`) | Connect + `SELECT 1` |
| Databricks PAT | `databricks_pat` | Overlap (create new, old expires) | Yes (token lifetime) | List tokens API |

---

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Plugins compiled in (not dynamic) | Static binary | Auditable, type-safe, no dependency management at runtime |
| Plugins don't write to Key Vault | Worker handles vault | Single Key Vault write path, consistent error handling, plugin stays focused |
| `PermanentError` for dead-letter | Error type sentinel | Worker can distinguish "retry later" from "never gonna work" |
| Config validated at registration | Fail fast | Catches bad config before scheduler creates a job that will fail |
| `SecretLifetime` passed to plugin | Plugin sets expiry | Plugin knows how to use it for the target resource's API |
| Shared password generator | `crypto/rand` + configurable | Consistent entropy across all password-based plugins |
| Capability reporting | Worker → API on register | Control plane knows if a type is unsupported before job fires |
