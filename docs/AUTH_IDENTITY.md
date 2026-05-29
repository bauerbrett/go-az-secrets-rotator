# Authentication & Identity

How every principal in the platform proves who it is, what it's allowed to do, and how infrastructure connections are secured — all without a single stored password.

---

## Table of Contents

- [Design Principles](#design-principles)
- [Identity Map](#identity-map)
- [Azure AD App Registration](#azure-ad-app-registration)
- [Token Flows](#token-flows)
- [Workload Identity (AKS Pods)](#workload-identity-aks-pods)
- [Managed Identity (Workers)](#managed-identity-workers)
- [APIM Gateway Identity](#apim-gateway-identity)
- [Infrastructure Connections](#infrastructure-connections)
- [Application Authentication](#application-authentication)
- [Application Authorization](#application-authorization)
- [Token Validation Implementation](#token-validation-implementation)
- [GitHub Actions OIDC](#github-actions-oidc)
- [Key Vault Access Model](#key-vault-access-model)
- [Dynamic RBAC for Workers](#dynamic-rbac-for-workers)
- [Security Invariants](#security-invariants)

---

## Design Principles

| Principle | Enforcement |
|-----------|-------------|
| Zero stored credentials | MI + Workload Identity everywhere; no connection strings, no SAS keys, no passwords |
| Short-lived tokens only | Azure AD tokens (~1h), auto-refreshed by SDK |
| Least privilege RBAC | Each service gets only the roles it needs, scoped as narrowly as possible |
| Defence in depth | APIM validates → backend re-validates → RLS enforces at DB level |
| Infra-level isolation | Service Bus RBAC is topic-scoped; workers physically can't read other tenants' queues |
| Auditability | Every request logs `actor_oid`, `actor_type`, `gateway_oid` |

---

## Identity Map

Every principal, what type of identity it has, and where its token is accepted.

| Principal | Identity Type | Authenticates To | Token Scope |
|-----------|--------------|-----------------|-------------|
| User (admin/operator) | Azure AD org account | APIM → API | `api://{app-id}/.default` |
| Worker (Container App) | User-Assigned MI | APIM → API, Service Bus, Key Vault, target resources | `api://{app-id}/.default` (for API); resource-specific for others |
| APIM Gateway | System-Assigned MI | Backend API (proves gateway origin) | `api://{app-id}/.default` |
| API Service pod | User-Assigned MI (Workload Identity) | PostgreSQL, Service Bus, Key Vault | Resource-specific scopes |
| Scheduler pod | User-Assigned MI (Workload Identity) | PostgreSQL, Service Bus | Resource-specific scopes |
| System Tasks pod | User-Assigned MI (Workload Identity) | PostgreSQL, Service Bus | Resource-specific scopes |
| AKS kubelet | System-Assigned MI | ACR (image pull) | `https://management.azure.com/.default` |
| GitHub Actions | Federated OIDC | Azure (Terraform, ACR push) | `https://management.azure.com/.default` |

---

## Azure AD App Registration

A single App Registration represents the platform. All callers (users, workers, APIM) target this one audience.

### Registration Setup

```json
{
  "displayName": "AZ Secrets Rotator Platform",
  "signInAudience": "AzureADMyOrg",
  "identifierUris": ["api://az-rotator-platform"],
  "api": {
    "oauth2PermissionScopes": [],
    "requestedAccessTokenVersion": 2
  },
  "appRoles": [
    {
      "allowedMemberTypes": ["User"],
      "displayName": "Platform Admin",
      "value": "Platform.Admin",
      "description": "Full platform access, bypass tenant isolation"
    },
    {
      "allowedMemberTypes": ["User"],
      "displayName": "Platform User",
      "value": "Platform.User",
      "description": "Per-tenant access, role looked up from DB"
    }
  ]
}
```

### Key Configuration Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Audience format | `api://az-rotator-platform` | Human-readable, stable across environments (with suffix) |
| Token version | v2.0 | Consistent claim names, `aud` = App ID URI |
| App Roles | User-only (`Platform.Admin`, `Platform.User`) | Workers don't get App Roles — they're identified by MI claims |
| `signInAudience` | `AzureADMyOrg` | Platform is internal; external tenants authenticate via their own Azure AD but target this audience |
| Scopes | None defined | Using App Roles for authorization, not delegated scopes |

### Per-Environment App Registrations

```
az-rotator-platform-dev      → api://az-rotator-platform-dev
az-rotator-platform-staging  → api://az-rotator-platform-staging
az-rotator-platform-prod     → api://az-rotator-platform-prod
```

Each environment gets its own registration so tokens from dev can't be replayed against prod.

### Assigning App Roles to Users

```bash
# Via Azure CLI
az ad app show --id "api://az-rotator-platform-prod" --query id -o tsv
# → {enterprise-app-object-id}

az rest --method POST \
  --uri "https://graph.microsoft.com/v1.0/servicePrincipals/{sp-id}/appRoleAssignedTo" \
  --body '{
    "principalId": "{user-object-id}",
    "resourceId": "{sp-id}",
    "appRoleId": "{platform-admin-role-id}"
  }'
```

Users without either App Role get a valid JWT but are **rejected at APIM** (policy checks `roles` claim exists).

---

## Token Flows

### User → API (Authorization Code + PKCE)

```
┌──────────┐       ┌───────────┐       ┌──────┐       ┌──────────┐
│   User   │       │  Azure AD │       │ APIM │       │ API Pod  │
│ (Browser)│       │           │       │      │       │          │
└────┬─────┘       └─────┬─────┘       └──┬───┘       └────┬─────┘
     │  1. /authorize (PKCE)               │                │
     │────────────────────►                │                │
     │  2. Login + consent                 │                │
     │◄────────────────────                │                │
     │  3. auth code returned              │                │
     │────────────────────►                │                │
     │  4. Exchange code → tokens          │                │
     │◄────────────────────                │                │
     │       (access_token + refresh_token)│                │
     │                                     │                │
     │  5. GET /api/secrets                │                │
     │     Authorization: Bearer {JWT}     │                │
     │─────────────────────────────────────►                │
     │                                     │  6. Validate   │
     │                                     │     client JWT │
     │                                     │  7. Acquire MI │
     │                                     │     token      │
     │                                     │  8. Forward    │
     │                                     │     both JWTs  │
     │                                     │────────────────►
     │                                     │                │ 9. Verify gateway JWT
     │                                     │                │ 10. Verify client JWT
     │                                     │                │ 11. Authorize (App Role + DB)
     │                                     │                │ 12. Execute + RLS
     │  13. Response                       │◄────────────────
     │◄─────────────────────────────────────                │
```

**Token details:**

| Field | Value |
|-------|-------|
| `iss` | `https://login.microsoftonline.com/{tenant-id}/v2.0` |
| `aud` | `api://az-rotator-platform-prod` |
| `oid` | User's Azure AD object ID |
| `tid` | User's Azure AD tenant ID |
| `roles` | `["Platform.Admin"]` or `["Platform.User"]` |
| `preferred_username` | `brett@contoso.com` |
| Lifetime | ~1h (configurable via CAE policy) |

### Worker → API (Client Credentials via MI)

```
┌────────────┐       ┌──────────────┐       ┌──────┐       ┌──────────┐
│   Worker   │       │   Azure AD   │       │ APIM │       │ API Pod  │
│ (Cont.App) │       │   (IMDS)     │       │      │       │          │
└─────┬──────┘       └──────┬───────┘       └──┬───┘       └────┬─────┘
      │  1. GET token from IMDS                 │                │
      │     resource=api://az-rotator-platform  │                │
      │─────────────────────►                   │                │
      │  2. MI JWT returned (~1h lifetime)      │                │
      │◄─────────────────────                   │                │
      │                                         │                │
      │  3. POST /api/workers/heartbeat         │                │
      │     Authorization: Bearer {MI JWT}      │                │
      │     X-Tenant-ID: {tenant-uuid}          │                │
      │─────────────────────────────────────────►                │
      │                                         │  4. Validate   │
      │                                         │     (no roles  │
      │                                         │      → worker) │
      │                                         │  5. Forward    │
      │                                         │────────────────►
      │                                         │                │ 6. Verify both JWTs
      │                                         │                │ 7. Lookup worker by oid
      │                                         │                │ 8. Check APPROVED status
      │                                         │                │ 9. Execute
      │  10. 200 OK                             │◄────────────────
      │◄─────────────────────────────────────────                │
```

**Token details (worker MI JWT):**

| Field | Value |
|-------|-------|
| `iss` | `https://login.microsoftonline.com/{tenant-id}/v2.0` |
| `aud` | `api://az-rotator-platform-prod` |
| `oid` | MI's object ID (matches `workers.azure_oid`) |
| `appid` | MI's client ID |
| `tid` | Tenant's Azure AD tenant ID |
| `roles` | **absent** (this is how we detect worker vs user) |
| Lifetime | ~1h |

**Caller-type detection logic:**

```go
func callerType(claims jwt.MapClaims) string {
    if roles, ok := claims["roles"].([]interface{}); ok && len(roles) > 0 {
        return "user"
    }
    return "worker"
}
```

---

## Workload Identity (AKS Pods)

How control plane pods (API, Scheduler, System Tasks) authenticate to Azure resources without any stored credentials.

### How It Works

```
┌─────────────┐      ┌─────────────────┐      ┌───────────┐      ┌──────────────┐
│  Pod        │      │ AKS OIDC Issuer │      │ Azure AD  │      │ Azure        │
│  (API svc)  │      │                 │      │           │      │ Resource     │
└──────┬──────┘      └────────┬────────┘      └─────┬─────┘      └──────┬───────┘
       │  1. Read projected SA token                 │                    │
       │     (/var/run/secrets/...token)             │                    │
       │                                             │                    │
       │  2. POST /oauth2/v2.0/token                 │                    │
       │     grant_type=client_credentials           │                    │
       │     client_assertion_type=urn:ietf:...jwt   │                    │
       │     client_assertion={SA token}             │                    │
       │     client_id={MI client ID}                │                    │
       │     scope={resource}/.default               │                    │
       │─────────────────────────────────────────────►                    │
       │                                             │                    │
       │  3. Azure AD validates:                     │                    │
       │     - SA token issuer = AKS OIDC URL        │                    │
       │     - SA token subject matches federated    │                    │
       │       credential config                     │                    │
       │     - Client ID matches MI                  │                    │
       │                                             │                    │
       │  4. Access token returned                   │                    │
       │◄─────────────────────────────────────────────                    │
       │                                                                  │
       │  5. Use access token to call resource                            │
       │─────────────────────────────────────────────────────────────────►│
```

### Three Pieces That Connect

```
┌─────────────────────────┐
│ 1. User-Assigned MI     │  ← Azure resource (has client ID + object ID)
│    Name: mi-az-rotator-api
│    Client ID: abc123...
└───────────┬─────────────┘
            │ linked via federated credential
            ▼
┌─────────────────────────┐
│ 2. Federated Credential │  ← Tells Azure AD: "trust tokens from this K8s SA"
│    Issuer: https://oidc.prod-aks...
│    Subject: system:serviceaccount:az-rotator:az-rotator-api-sa
│    Audience: api://AzureADTokenExchange
└───────────┬─────────────┘
            │ matches
            ▼
┌─────────────────────────┐
│ 3. Kubernetes SA        │  ← In-cluster identity
│    Name: az-rotator-api-sa
│    Annotation: azure.workload.identity/client-id: abc123...
│    Namespace: az-rotator
└─────────────────────────┘
```

### Terraform Configuration

```hcl
# 1. User-Assigned Managed Identity
resource "azurerm_user_assigned_identity" "api" {
  name                = "mi-az-rotator-api-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
}

# 2. Federated Credential (links K8s SA → MI)
resource "azurerm_federated_identity_credential" "api" {
  name                = "fed-cred-api"
  resource_group_name = azurerm_resource_group.main.name
  parent_id           = azurerm_user_assigned_identity.api.id

  issuer    = azurerm_kubernetes_cluster.main.oidc_issuer_url
  subject   = "system:serviceaccount:az-rotator:az-rotator-api-sa"
  audience  = ["api://AzureADTokenExchange"]
}

# 3. Role assignments (what this MI can do)
resource "azurerm_role_assignment" "api_servicebus_owner" {
  scope                = azurerm_servicebus_namespace.main.id
  role_definition_name = "Azure Service Bus Data Owner"
  principal_id         = azurerm_user_assigned_identity.api.principal_id
}

resource "azurerm_role_assignment" "api_keyvault_secrets_user" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_user_assigned_identity.api.principal_id
}
```

### All Federated Credentials

| MI | Kubernetes SA | Subject Claim |
|----|---------------|---------------|
| `mi-az-rotator-api` | `az-rotator-api-sa` | `system:serviceaccount:az-rotator:az-rotator-api-sa` |
| `mi-az-rotator-scheduler` | `az-rotator-scheduler-sa` | `system:serviceaccount:az-rotator:az-rotator-scheduler-sa` |
| `mi-az-rotator-system` | `az-rotator-system-sa` | `system:serviceaccount:az-rotator:az-rotator-system-sa` |

### In Go Code

The SDK handles the entire exchange transparently:

```go
import "github.com/Azure/azure-sdk-for-go/sdk/azidentity"

// This automatically detects Workload Identity in AKS
// (reads AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_FEDERATED_TOKEN_FILE from projected env)
cred, err := azidentity.NewDefaultAzureCredential(nil)

// Get a token for PostgreSQL
token, err := cred.GetToken(ctx, policy.TokenRequestOptions{
    Scopes: []string{"https://ossrdbms-aad.database.windows.net/.default"},
})

// Get a token for Service Bus
token, err := cred.GetToken(ctx, policy.TokenRequestOptions{
    Scopes: []string{"https://servicebus.azure.net/.default"},
})
```

The Azure SDK injects these env vars into pods via the mutating webhook:

| Env Var | Value | Source |
|---------|-------|--------|
| `AZURE_CLIENT_ID` | MI client ID | From SA annotation |
| `AZURE_TENANT_ID` | Azure AD tenant ID | From AKS config |
| `AZURE_FEDERATED_TOKEN_FILE` | `/var/run/secrets/azure/tokens/azure-identity-token` | Projected volume |
| `AZURE_AUTHORITY_HOST` | `https://login.microsoftonline.com/` | Default |

---

## Managed Identity (Workers)

Workers run outside AKS (Container Apps, VMs). They use **User-Assigned Managed Identity** directly — no OIDC federation needed because the compute platform provides IMDS.

### Why User-Assigned (not System-Assigned)

| | System-Assigned | User-Assigned |
|---|---|---|
| Lifecycle | Tied to resource (delete Container App → MI gone) | Independent (survives redeployment) |
| Reuse | One per resource | Can attach to multiple resources |
| Role tracking | Must reassign roles on recreate | Roles persist |
| Control | Less explicit | Explicit: you know exactly which MI object ID to trust |

We use **User-Assigned** because:
- Worker restarts/redeployments shouldn't require re-registering with the platform
- The MI's object ID is stored in `workers.azure_oid` — it must be stable
- Tenant teams create the MI in their own subscription, attach to their Container App

### Token Acquisition

```go
// Worker startup
cred, err := azidentity.NewDefaultAzureCredential(nil)
// In Container Apps, this uses IMDS (no projected token file needed)

// Token for calling the control plane API
token, err := cred.GetToken(ctx, policy.TokenRequestOptions{
    Scopes: []string{"api://az-rotator-platform-prod/.default"},
})

// Token for Service Bus (receiving rotation jobs)
token, err := cred.GetToken(ctx, policy.TokenRequestOptions{
    Scopes: []string{"https://servicebus.azure.net/.default"},
})

// Token for target Key Vault (writing rotated secrets)
token, err := cred.GetToken(ctx, policy.TokenRequestOptions{
    Scopes: []string{"https://vault.azure.net/.default"},
})
```

### Required Environment Variable

```bash
AZURE_CLIENT_ID="{user-assigned-mi-client-id}"
```

When a User-Assigned MI is attached, the worker must specify which MI to use (Container Apps can have multiple). `DefaultAzureCredential` reads `AZURE_CLIENT_ID` to select the correct one.

---

## APIM Gateway Identity

APIM has a **System-Assigned MI** that it uses to prove to the backend that a request legitimately came through the gateway (not directly to the pod).

### Token Acquisition (APIM Policy)

```xml
<!-- In APIM inbound policy -->
<authentication-managed-identity
    resource="api://az-rotator-platform-prod"
    output-token-variable-name="gateway-token" />

<set-header name="X-Gateway-Auth" exists-action="override">
    <value>@("Bearer " + (string)context.Variables["gateway-token"])</value>
</set-header>
```

### What the Backend Validates

```go
// Backend middleware step 2: Gateway Auth Verification
func verifyGateway(gatewayToken string, knownAPIMOid string) error {
    claims, err := validateJWT(gatewayToken, expectedAudience)
    if err != nil {
        return fmt.Errorf("gateway token invalid: %w", err)
    }

    // Must be the known APIM MI — not some random MI
    if claims["oid"] != knownAPIMOid {
        return fmt.Errorf("gateway OID mismatch: got %s, want %s", claims["oid"], knownAPIMOid)
    }

    return nil
}
```

**Config:**

```yaml
env:
  - name: APIM_MI_OID
    value: "a1b2c3d4-..."  # APIM's system-assigned MI object ID
```

This prevents bypass attacks: even if an attacker reaches the pod directly (e.g., via cluster compromise), they can't forge a valid `X-Gateway-Auth` header because they'd need to impersonate APIM's MI.

---

## Infrastructure Connections

Every connection from a service to an Azure resource, what token it uses, and what scope/role is required.

### API Service

| Target Resource | Scope | Role | Purpose |
|----------------|-------|------|---------|
| PostgreSQL | `https://ossrdbms-aad.database.windows.net/.default` | Azure AD Auth (DB role: `az_rotator_api`) | Read/write tenant data (RLS enforced) |
| Service Bus | `https://servicebus.azure.net/.default` | `Azure Service Bus Data Owner` | Create/delete topics, publish messages |
| Key Vault | `https://vault.azure.net/.default` | `Key Vault Secrets User` | Read signing keys (via ESO) |

### Scheduler

| Target Resource | Scope | Role | Purpose |
|----------------|-------|------|---------|
| PostgreSQL | `https://ossrdbms-aad.database.windows.net/.default` | Azure AD Auth (DB role: `az_rotator_system`) | Poll due jobs (RLS bypassed) |
| Service Bus | `https://servicebus.azure.net/.default` | `Azure Service Bus Data Sender` | Publish job messages to topics |

### System Tasks

| Target Resource | Scope | Role | Purpose |
|----------------|-------|------|---------|
| PostgreSQL | `https://ossrdbms-aad.database.windows.net/.default` | Azure AD Auth (DB role: `az_rotator_system`) | Cleanup, reconciliation (RLS bypassed) |
| Service Bus | `https://servicebus.azure.net/.default` | `Azure Service Bus Data Receiver` | Read DLQ for retry handling |

### Worker

| Target Resource | Scope | Role | Purpose |
|----------------|-------|------|---------|
| Platform API | `api://az-rotator-platform-prod/.default` | None (authed by `workers` table + APPROVED status) | Register, heartbeat, report results |
| Service Bus | `https://servicebus.azure.net/.default` | `Azure Service Bus Data Receiver` (topic-scoped) | Receive rotation jobs |
| Key Vault (target) | `https://vault.azure.net/.default` | `Key Vault Secrets Officer` | Write rotated secrets |
| MS Graph | `https://graph.microsoft.com/.default` | `Application.ReadWrite.OwnedBy` (app-level) | Rotate App Registration client secrets |
| PostgreSQL (target) | `https://ossrdbms-aad.database.windows.net/.default` | Login permission on target DB | Rotate PostgreSQL passwords |

### PostgreSQL Azure AD Auth Setup

```sql
-- Run as AAD admin on the Flexible Server

-- Create roles for each service (mapped to MI client IDs)
CREATE ROLE "az_rotator_api" LOGIN;
CREATE ROLE "az_rotator_system" LOGIN;
CREATE ROLE "az_rotator_migrations" LOGIN;

-- Grant permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO az_rotator_api;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO az_rotator_system;
GRANT ALL ON SCHEMA public TO az_rotator_migrations;

-- Enable these as Azure AD roles
-- (Azure handles mapping MI client ID → PostgreSQL role name automatically
--  when the DB has AAD auth enabled and the MI is set as AAD admin or granted login)
```

**Connection from Go:**

```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
    "github.com/jackc/pgx/v5/pgxpool"
)

func newPool(ctx context.Context) (*pgxpool.Pool, error) {
    cred, _ := azidentity.NewDefaultAzureCredential(nil)

    config, _ := pgxpool.ParseConfig(
        "host=psql-azrotator-prod.postgres.database.azure.com " +
        "port=5432 dbname=azrotator sslmode=require " +
        "user=mi-az-rotator-api-prod",  // MI client ID or name
    )

    // Override password with Azure AD token on each connection
    config.BeforeConnect = func(ctx context.Context, cfg *pgx.ConnConfig) error {
        token, err := cred.GetToken(ctx, policy.TokenRequestOptions{
            Scopes: []string{"https://ossrdbms-aad.database.windows.net/.default"},
        })
        if err != nil {
            return err
        }
        cfg.Password = token.Token
        return nil
    }

    return pgxpool.NewWithConfig(ctx, config)
}
```

The `BeforeConnect` hook means every new connection gets a fresh token. No stale credentials.

---

## Application Authentication

How the API service authenticates incoming requests (both user and worker).

### JWT Validation

```go
type JWTValidator struct {
    jwksURL   string          // https://login.microsoftonline.com/{tenant}/discovery/v2.0/keys
    audience  string          // api://az-rotator-platform-prod
    issuer    string          // https://login.microsoftonline.com/{tenant}/v2.0
    keySet    jwk.Set         // cached JWKS keys
    lastFetch time.Time
    mu        sync.RWMutex
}

func (v *JWTValidator) Validate(tokenString string) (jwt.MapClaims, error) {
    // 1. Parse without verification to read header (get kid)
    // 2. Lookup kid in cached JWKS (refresh if not found)
    // 3. Verify signature with matching public key
    // 4. Validate claims:
    //    - iss == expected issuer
    //    - aud == expected audience
    //    - exp > now (with 30s clock skew tolerance)
    //    - nbf < now
    //    - iat is reasonable
    // 5. Return validated claims
}
```

### JWKS Caching

| Behaviour | Value |
|-----------|-------|
| Initial fetch | On startup (readiness probe blocks until done) |
| Refresh interval | Every 24h (background goroutine) |
| On `kid` miss | Immediate re-fetch (handles key rotation) |
| Max staleness | 48h — if refresh fails for 48h, start rejecting tokens |
| Cache type | In-memory (`jwk.Set`) |

Azure AD rotates signing keys roughly every 6 weeks. The `kid`-miss strategy handles this seamlessly.

### Clock Skew Tolerance

```go
const clockSkew = 30 * time.Second

func validateExpiry(claims jwt.MapClaims) error {
    exp := time.Unix(int64(claims["exp"].(float64)), 0)
    if time.Now().After(exp.Add(clockSkew)) {
        return ErrTokenExpired
    }
    return nil
}
```

### Multi-Tenant Issuers

Workers from different Azure AD tenants have different `iss` values. The validator accepts any issuer matching the pattern:

```
https://login.microsoftonline.com/{any-tenant-id}/v2.0
```

But validates the signature against Microsoft's common JWKS endpoint, which covers all tenants:

```
https://login.microsoftonline.com/common/discovery/v2.0/keys
```

The `tid` claim in the token is then matched against the tenant record to ensure cross-org protection.

---

## Application Authorization

After authentication succeeds, authorization determines what the caller can do.

### Authorization Decision Tree

```
Request arrives (authenticated)
│
├─ Caller type = "user" (has `roles` claim)
│   │
│   ├─ Role = "Platform.Admin"
│   │   └─ ✅ ALLOW (full access, no tenant scoping)
│   │
│   └─ Role = "Platform.User"
│       │
│       ├─ Lookup tenant_members WHERE user_oid = {oid} AND tenant_id = {target-tenant}
│       │   │
│       │   ├─ Row found → check role ≥ required permission
│       │   │   ├─ owner  → all operations
│       │   │   ├─ admin  → manage secrets, workers, policies
│       │   │   ├─ operator → trigger rotations, view status
│       │   │   └─ viewer → read-only
│       │   │
│       │   └─ No row → ❌ 403 Forbidden
│       │
│       └─ Cross-org check: JWT `tid` must match tenant's `azure_ad_tenant_id`
│           ├─ Match → continue
│           └─ Mismatch → ❌ 403 Forbidden
│
└─ Caller type = "worker" (no `roles` claim)
    │
    ├─ Lookup workers WHERE azure_oid = {oid} AND tenant_id = {X-Tenant-ID}
    │   │
    │   ├─ Found + status = "APPROVED" → ✅ ALLOW (scoped to worker endpoints)
    │   ├─ Found + status ≠ "APPROVED" → ❌ 403 Forbidden (suspended/pending)
    │   └─ Not found → ❌ 401 Unauthorized (not registered)
    │
    └─ Cross-org check: JWT `tid` must match tenant's `azure_ad_tenant_id`
```

### Tenant Role Hierarchy

```
owner > admin > operator > viewer
```

| Role | Secrets CRUD | Trigger Rotation | Manage Workers | Manage Members | Manage Policies |
|------|:---:|:---:|:---:|:---:|:---:|
| owner | ✅ | ✅ | ✅ | ✅ | ✅ |
| admin | ✅ | ✅ | ✅ | ❌ | ✅ |
| operator | ❌ | ✅ | ❌ | ❌ | ❌ |
| viewer | ❌ | ❌ | ❌ | ❌ | ❌ |

### Endpoint Permission Matrix

| Endpoint | Required Role | Notes |
|----------|--------------|-------|
| `GET /api/tenants/{id}/secrets` | viewer+ | RLS + role check |
| `POST /api/tenants/{id}/secrets` | admin+ | Create secret registration |
| `POST /api/tenants/{id}/secrets/{id}/rotate` | operator+ | Trigger manual rotation |
| `GET /api/tenants/{id}/workers` | viewer+ | List workers |
| `POST /api/workers/register` | worker only | MI JWT auth |
| `PUT /api/workers/{id}/approve` | Platform.Admin | Admin approval gate |
| `GET /api/tenants/{id}/audit-logs` | admin+ | Audit trail |
| `POST /api/tenants` | Platform.Admin | Create new tenant |
| `POST /api/tenants/{id}/members` | owner | Invite member |

### Authorization Middleware (Go)

```go
type Permission struct {
    MinRole    string // "viewer", "operator", "admin", "owner"
    CallerType string // "user", "worker", "any"
    AdminOnly  bool   // requires Platform.Admin
}

func Authorize(perm Permission) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            caller := CallerFromContext(r.Context())

            // Platform.Admin bypasses everything
            if caller.Type == "user" && caller.HasRole("Platform.Admin") {
                next.ServeHTTP(w, r)
                return
            }

            // Admin-only endpoints
            if perm.AdminOnly {
                http.Error(w, "forbidden", http.StatusForbidden)
                return
            }

            // Caller type check
            if perm.CallerType != "any" && caller.Type != perm.CallerType {
                http.Error(w, "forbidden", http.StatusForbidden)
                return
            }

            // Worker: check APPROVED status (already done in auth middleware)
            if caller.Type == "worker" {
                next.ServeHTTP(w, r)
                return
            }

            // User: check tenant membership role
            tenantID := chi.URLParam(r, "tenantID")
            membership, err := lookupMembership(r.Context(), caller.OID, tenantID)
            if err != nil || !meetsMinRole(membership.Role, perm.MinRole) {
                http.Error(w, "forbidden", http.StatusForbidden)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

### Cross-Organization Guard

Prevents a user in `contoso.com` from accessing a tenant registered by `fabrikam.com`:

```go
func crossOrgCheck(callerTID string, tenant *Tenant) error {
    if callerTID != tenant.AzureADTenantID {
        return fmt.Errorf(
            "cross-org violation: caller tid=%s, tenant expects tid=%s",
            callerTID, tenant.AzureADTenantID,
        )
    }
    return nil
}
```

This runs after authentication but before authorization. Blocks the request entirely if the caller's Azure AD tenant doesn't match.

---

## Token Validation Implementation

### Startup Sequence

```go
func main() {
    // 1. Initialize credential (Workload Identity)
    cred, err := azidentity.NewDefaultAzureCredential(nil)

    // 2. Fetch JWKS keys (blocks startup — readiness probe waits)
    validator, err := NewJWTValidator(JWTValidatorConfig{
        JWKSURL:       "https://login.microsoftonline.com/common/discovery/v2.0/keys",
        Audience:      os.Getenv("AZURE_AD_APP_ID"), // api://az-rotator-platform-prod
        ClockSkew:     30 * time.Second,
        RefreshPeriod: 24 * time.Hour,
    })

    // 3. Start JWKS background refresh
    go validator.BackgroundRefresh(ctx)

    // 4. Mark ready (readiness probe returns 200)
    ready.Store(true)
}
```

### Token Refresh in Long-Running Services

The Azure SDK handles caching and refresh internally:

```go
// azidentity.DefaultAzureCredential caches tokens automatically.
// GetToken() returns cached token if still valid (>5min to expiry).
// If within 5min of expiry → background refresh.
// If expired → blocking refresh.

token, err := cred.GetToken(ctx, policy.TokenRequestOptions{
    Scopes: []string{"https://servicebus.azure.net/.default"},
})
// token.Token is always valid when err == nil
// token.ExpiresOn tells you when it expires (but you don't need to track this)
```

**You never manually refresh.** The SDK's `TokenCredential` interface guarantees a valid token on every `.GetToken()` call.

---

## GitHub Actions OIDC

CI/CD uses federated identity — no stored secrets in GitHub.

### Setup

```hcl
# Terraform: App Registration for CI
resource "azuread_application" "github_ci" {
  display_name = "github-ci-az-rotator"
}

resource "azuread_service_principal" "github_ci" {
  client_id = azuread_application.github_ci.client_id
}

# Federated credential: trust GitHub's OIDC tokens
resource "azuread_application_federated_identity_credential" "github_main" {
  application_id = azuread_application.github_ci.id
  display_name   = "github-main-branch"

  issuer    = "https://token.actions.githubusercontent.com"
  subject   = "repo:your-org/go-az-secrets-rotator:ref:refs/heads/main"
  audiences = ["api://AzureADTokenExchange"]
}

# For PR builds (deploy to dev)
resource "azuread_application_federated_identity_credential" "github_pr" {
  application_id = azuread_application.github_ci.id
  display_name   = "github-pull-requests"

  issuer    = "https://token.actions.githubusercontent.com"
  subject   = "repo:your-org/go-az-secrets-rotator:pull_request"
  audiences = ["api://AzureADTokenExchange"]
}

# Role assignments
resource "azurerm_role_assignment" "ci_acr_push" {
  scope                = azurerm_container_registry.main.id
  role_definition_name = "AcrPush"
  principal_id         = azuread_service_principal.github_ci.object_id
}

resource "azurerm_role_assignment" "ci_contributor" {
  scope                = azurerm_resource_group.main.id
  role_definition_name = "Contributor"
  principal_id         = azuread_service_principal.github_ci.object_id
}
```

### In Workflow

```yaml
permissions:
  id-token: write   # Required for OIDC
  contents: read

steps:
  - uses: azure/login@v2
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}       # App Registration client ID
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      # No client-secret or certificate — OIDC handles it
```

The `azure/login` action:
1. Requests an OIDC token from GitHub (`ACTIONS_ID_TOKEN_REQUEST_TOKEN`)
2. Exchanges it with Azure AD using the federated credential
3. Gets an Azure AD access token
4. Sets up the Azure CLI session

**Subject claim matching:**
- `refs/heads/main` → only main branch workflows can use this credential
- `pull_request` → PR builds (scoped to dev only)
- You can further restrict with `environment:production` subjects for prod deploys

---

## Key Vault Access Model

Two distinct access patterns for Key Vault:

### Control Plane (ESO → K8s Secrets)

```
ESO ClusterSecretStore → Workload Identity (API SA) → Key Vault → K8s Secret
```

| Detail | Value |
|--------|-------|
| Who | External Secrets Operator (using API service's Workload Identity) |
| Role | `Key Vault Secrets User` (read-only) |
| Scope | Platform Key Vault (`kv-azrotator-prod`) |
| What it reads | `job-signing-key`, HMAC keys, non-DB secrets |
| Sync interval | Every 1h |
| Key Vault SKU | Premium (HSM-backed keys, private endpoint) |

### Worker (Direct Access → Target Vaults)

```
Worker MI → Key Vault Secrets Officer → Tenant's Key Vault → Write rotated secret
```

| Detail | Value |
|--------|-------|
| Who | Worker Managed Identity |
| Role | `Key Vault Secrets Officer` (read + write) |
| Scope | Tenant's own Key Vault (not the platform vault) |
| What it does | Writes newly rotated secrets, reads current values for validation |
| Granted by | Tenant admin (assigns role to worker MI in their subscription) |

**Key isolation:** Workers never access the platform Key Vault. The platform never accesses tenant Key Vaults. Complete separation.

---

## Dynamic RBAC for Workers

When a worker is approved, the platform assigns its MI the necessary Azure RBAC. When deactivated, it removes it.

### Assignment Flow

```
Worker approved
│
├─ API Service calls Azure Resource Manager:
│   POST /providers/Microsoft.Authorization/roleAssignments
│
├─ Assigns: Azure Service Bus Data Receiver
│   Scope: /subscriptions/.../topics/{tenant-topic}
│   Principal: worker MI object ID
│
└─ Stores role assignment ID in workers table
```

### Go Implementation

```go
import "github.com/Azure/azure-sdk-for-go/sdk/resourcemanager/authorization/armauthorization/v2"

func assignServiceBusRole(ctx context.Context, workerOID, topicResourceID string) error {
    client, _ := armauthorization.NewRoleAssignmentsClient(subscriptionID, cred, nil)

    _, err := client.Create(ctx,
        topicResourceID,  // scope: specific topic
        uuid.New().String(),
        armauthorization.RoleAssignmentCreateParameters{
            Properties: &armauthorization.RoleAssignmentProperties{
                RoleDefinitionID: to.Ptr(
                    "/providers/Microsoft.Authorization/roleDefinitions/4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0",
                    // Azure Service Bus Data Receiver
                ),
                PrincipalID:   to.Ptr(workerOID),
                PrincipalType: to.Ptr(armauthorization.PrincipalTypeServicePrincipal),
            },
        },
        nil,
    )
    return err
}
```

### Lifecycle

| Event | RBAC Action |
|-------|-------------|
| Worker approved | Assign `Data Receiver` on tenant's topic |
| Worker suspended | Remove role assignment |
| Worker restored | Re-assign role |
| Worker deleted | Remove role assignment + delete subscription |

The API service's MI needs the `User Access Administrator` role on the Service Bus namespace to manage these assignments:

```hcl
resource "azurerm_role_assignment" "api_rbac_manager" {
  scope                = azurerm_servicebus_namespace.main.id
  role_definition_name = "User Access Administrator"
  principal_id         = azurerm_user_assigned_identity.api.principal_id
}
```

---

## Security Invariants

Rules that must always hold true. If any is violated, the system is compromised.

| # | Invariant | Enforced By |
|---|-----------|-------------|
| 1 | No request reaches the API pod without a valid `X-Gateway-Auth` from the known APIM MI | Gateway auth middleware + APIM_MI_OID env var |
| 2 | Every client JWT was issued by Azure AD for the correct audience | JWKS signature verification + audience check |
| 3 | A worker can only access its own tenant's Service Bus topic | Topic-scoped RBAC (Azure infra level) |
| 4 | A user can only access tenants where they have a `tenant_members` row | Authorization middleware + RLS |
| 5 | No cross-org access: JWT `tid` must match tenant's `azure_ad_tenant_id` | Cross-org guard middleware |
| 6 | Database queries from the API are always scoped to one tenant | PostgreSQL RLS (`app.current_tenant_id` session var) |
| 7 | No stored credentials anywhere in the system | MI + Workload Identity + OIDC everywhere |
| 8 | Token forgery is impossible without Azure AD's private key | RSA signature on all JWTs |
| 9 | Suspended workers lose Service Bus access immediately | Dynamic RBAC removal on suspension |
| 10 | CI cannot deploy to prod without a merge to the prod deploy branch | GitHub OIDC subject claim restricts credential scope |

---

## Summary

| Layer | Mechanism | Details |
|-------|-----------|---------|
| User → APIM | Azure AD JWT (auth code + PKCE) | App Roles: `Platform.Admin`, `Platform.User` |
| Worker → APIM | MI JWT (client_credentials via IMDS) | No App Roles; identified by `oid` + `workers` table |
| APIM → Backend | System-assigned MI JWT | Backend checks `X-Gateway-Auth` against known APIM OID |
| Pod → PostgreSQL | Workload Identity → Azure AD token as password | `BeforeConnect` hook, auto-refresh |
| Pod → Service Bus | Workload Identity → Azure AD token | SDK handles caching |
| Pod → Key Vault | Workload Identity → Azure AD token | ESO for reads; workers write to tenant vaults |
| Worker → Service Bus | User-Assigned MI → Azure AD token | Topic-scoped `Data Receiver` |
| Worker → Key Vault | User-Assigned MI → Azure AD token | `Key Vault Secrets Officer` on tenant vault |
| GitHub Actions → Azure | OIDC federation | Subject-restricted per branch/environment |
| AKS → ACR | Kubelet system MI | `AcrPull` role |
| Authorization | App Roles + DB tenant_members + RLS | Defence in depth: APIM → middleware → DB |
