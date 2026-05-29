# APIM Gateway — Detailed Design

## Overview

APIM is the **single public entry point** for all external traffic to the platform. Two kinds of clients call through APIM:

| Client Type | Identity Source | Example Calls |
|-------------|----------------|---------------|
| **Users** (tenant admins / operators) | Normal Azure AD account (org account login) | Create tenants, approve workers, manage policies, view audit logs |
| **Workers** (Container App agents) | Azure Managed Identity | Register, send heartbeats, report rotation status |

APIM authenticates **every** inbound request (user or worker) by validating the caller's Azure AD JWT **before** any traffic reaches the backend. APIM then authenticates **itself** to the backend API using its own Managed Identity, creating a **dual-auth** model where every request that arrives at the API pod carries two verified identities.

### Core Responsibilities

1. **Client JWT validation** — verify signature, issuer, audience, expiry for both users and workers
2. **Caller-type detection** — distinguish user vs worker from JWT claims and route context
3. **Tenant isolation** — extract and enforce tenant scope from claims / headers
4. **Rate limiting & throttling** — per-tenant and per-caller-type limits
5. **APIM-to-backend authentication** — APIM acquires its own MI token and injects it as a second identity
6. **Header enrichment** — inject verified caller metadata for downstream consumption
7. **Secure forwarding** — route to AKS API service over private network

---

## Dual-Auth Model

The platform uses two layers of authentication on every request:

```
                      ┌────────────────────────┐
                      │ Client (User or Worker) │
                      │ Identity #1: Client JWT │
                      └───────────┬────────────┘
                                  │
                                  ▼
                      ┌────────────────────────┐
                      │       APIM Gateway      │
                      │                        │
                      │  1. Validate Client JWT │
                      │  2. Acquire APIM MI JWT │
                      │  3. Forward both to API │
                      └───────────┬────────────┘
                                  │
                         Two JWTs sent downstream:
                         • Authorization: Bearer {Client JWT}
                         • X-Gateway-Auth: Bearer {APIM MI JWT}
                                  │
                                  ▼
                      ┌────────────────────────┐
                      │   API Service (AKS)    │
                      │                        │
                      │  1. Verify APIM MI JWT │
                      │     (trusted gateway?) │
                      │  2. Verify Client JWT  │
                      │     (who is calling?)  │
                      │  3. Authorize request  │
                      └────────────────────────┘
```

**Why dual auth?**
- The API pod **never** accepts traffic that didn't come through APIM (verifies gateway identity)
- The API pod **always** knows the original caller identity (re-validates client JWT for defense in depth)
- If APIM is compromised or bypassed, the backend rejects requests missing a valid APIM MI token
- Full audit trail: every request is logged with both gateway OID and caller OID

---

## Client Types & Token Acquisition

### Users (Tenant Admins / Operators)

Users authenticate with their **normal Azure AD organizational accounts** (e.g. `admin@contoso.com`). No app registrations or service principals are needed for human users. The platform registers an Enterprise Application in Azure AD that users consent to, which allows Azure AD to issue tokens with the correct audience.

- **Interactive login**: Users sign in via standard browser-based Azure AD login (authorization code flow with PKCE). Tokens are obtained through `az login`, a CLI tool, or any OAuth2-capable client.
- **Audience**: The platform's registered application ID — same audience used by workers so APIM validates both with one policy.
- **App Roles**: Defined on the App Registration manifest. Assigned to users (or groups) via the Enterprise Application in Azure AD. Appear in the JWT `roles` claim.

**App Roles defined on the App Registration**:

| App Role Value | Display Name | Who Gets It | Effect |
|----------------|--------------|-------------|--------|
| `Platform.Admin` | Platform Administrator | Global admins | Bypasses per-tenant checks — full access to all tenants |
| `Platform.User` | Platform User | All regular users | Grants API access; per-tenant permissions come from `tenant_members` table |

> A user **must** have at least one App Role to call the API at all. Users without a role in their JWT are rejected at APIM. Per-tenant granularity (owner / admin / operator / viewer) is controlled in the database via the `tenant_members` table — not via Azure AD.

**Key JWT claims for users**:
- `oid` — Azure AD object ID of the user
- `preferred_username` — user's email / UPN (e.g. `admin@contoso.com`)
- `roles` — platform-wide App Roles (`Platform.Admin` or `Platform.User`)
- `tid` — Azure AD tenant ID (used to cross-reference platform tenant's `azure_ad_tenant_id`)
- No `appid` claim — this is a user token, not a service principal

### Workers (Container App Agents)

Workers authenticate via **Managed Identity** (system-assigned or user-assigned on the Container App). The MI obtains a token from Azure AD using the client_credentials grant — no secrets, passwords, or certificates are stored or managed by the worker.

- **Token acquisition**: The Container App runtime automatically fetches a token from the Azure Instance Metadata Service (IMDS) when the worker requests one for the platform's audience.
- **Audience**: Same platform application ID as users.

**Key JWT claims for workers**:
- `oid` — MI object ID (unique worker identifier)
- `appid` — MI application ID
- `tid` — Azure AD tenant (used to cross-reference platform tenant)
- No `roles` claim — worker authorization is based on registration status in the database, not Azure AD roles

> **How APIM tells them apart**: Both client types target the **same audience**. APIM (and the backend) differentiate callers by checking for the `roles` claim. If `roles` is present (contains `Platform.Admin` or `Platform.User`) → user. If `roles` is absent and `X-Tenant-ID` header is provided → worker.

---

## APIM Inbound Policy Chain

All policies execute in the APIM **inbound** pipeline, in order. These are described as design behaviors — implementation uses standard APIM policy elements.

### Step 1: Validate Client JWT (Users & Workers)

APIM validates the `Authorization: Bearer` header on every request:

- Fetches signing keys automatically from Azure AD's OpenID Connect metadata endpoint (JWKS)
- Verifies JWT **signature** (cryptographic proof it was issued by Azure AD)
- Verifies **issuer** matches the expected Azure AD tenant
- Verifies **audience** matches the platform's registered application ID
- Verifies **expiry** (rejects expired tokens)
- Extracts `oid`, `tid`, and `roles` claims into APIM context variables for use in later policies

Both user and worker tokens are validated by the same policy — they share the same audience. No manual key management is needed; APIM handles JWKS rotation automatically.

### Step 2: Detect Caller Type & Extract Context

After JWT validation, APIM determines the caller type:

- **If `roles` claim exists** → caller is a **user**. Tenant ID is derived from the JWT `tid` claim.
- **If `roles` claim is absent** → caller is a **worker**. Tenant ID is read from the required `X-Tenant-ID` request header.

If a worker request is missing the `X-Tenant-ID` header, APIM returns `400 Bad Request` immediately.

### Step 3: Rate Limiting

APIM applies rate limits to protect the backend:

- **Per-tenant limit**: 100 requests/minute, keyed by resolved tenant ID. Prevents any single tenant from overwhelming the platform.
- **Per-caller registration limit**: 10 requests/minute on the `/workers/register` endpoint, keyed by caller OID. Prevents registration abuse.

### Step 4: APIM Acquires Its Own MI Token (Gateway Identity)

This is the key piece of the dual-auth model. APIM uses its own **system-assigned Managed Identity** to acquire a JWT token from Azure AD, targeting the platform's application audience. This token proves to the backend that the request came through the trusted APIM gateway.

APIM injects the following headers before forwarding:

| Header | Value | Source |
|--------|-------|--------|
| `Authorization` | `Bearer {client JWT}` | Unchanged — original caller token passed through |
| `X-Gateway-Auth` | `Bearer {APIM MI JWT}` | Freshly acquired by APIM using its own MI |
| `X-Caller-OID` | Caller's `oid` claim | Extracted from validated client JWT |
| `X-Caller-Type` | `user` or `worker` | Determined in step 2 |
| `X-Tenant-ID` | Platform tenant UUID | From JWT `tid` (user) or request header (worker) |

### Step 5: Forward to Backend

APIM routes the enriched request to the backend API service over the private network path.

---

## APIM → Backend: Network Path

```
APIM (Public Endpoint, VNET-integrated)
    ↓
    │  HTTPS (VNET-injected private traffic)
    ↓
Azure Internal Load Balancer (private IP in AKS VNET)
    ↓
    │  Routes to AKS node pool
    ↓
Ingress Controller (nginx, terminates TLS from ILB)
    ↓
    │  Plain HTTP within cluster
    │  Cilium encrypts pod-to-pod at kernel level (WireGuard)
    ↓
API Service Pod (az-rotator-api namespace, listens on :8080)
```

**Key points**:
- APIM is VNET-integrated — traffic goes directly into the VNET without public hops
- Internal LB has a private IP only — not internet-reachable
- Ingress controller terminates TLS from the load balancer
- Within the cluster, Cilium encrypts all pod traffic transparently (WireGuard)
- API pod runs plain HTTP on port 8080 — no app-level TLS burden
- Cilium network policies restrict ingress to the API pod to only the ingress-nginx namespace

---

## Backend Dual-Auth Verification (Defense in Depth)

The API service validates **both** identities on every request. It never trusts APIM-injected headers alone.

### Headers Arriving at the API Pod

| Header | Purpose |
|--------|---------|
| `Authorization: Bearer {Client JWT}` | Original caller token (user or worker) |
| `X-Gateway-Auth: Bearer {APIM MI JWT}` | APIM's own MI token — proves gateway origin |
| `X-Caller-OID` | Caller's object ID, extracted by APIM from client JWT |
| `X-Caller-Type` | `user` or `worker`, determined by APIM |
| `X-Tenant-ID` | Resolved tenant context |

### Verification Steps

The backend middleware performs these checks in order:

1. **Verify gateway identity** — Parse and validate the `X-Gateway-Auth` JWT against Azure AD. Confirm the `oid` claim matches the known APIM Managed Identity object ID. If this fails, reject with `401 Unauthorized` — the request did not come through the trusted gateway.

2. **Verify client identity** — Parse and validate the `Authorization` JWT against Azure AD. Check signature, issuer, audience, and expiry. This is defense in depth — APIM already validated it, but the backend re-checks independently.

3. **Anti-tampering check** — Compare the `oid` claim from the client JWT against the `X-Caller-OID` header injected by APIM. If they don't match, someone tampered with the headers in transit. Reject with `401 Unauthorized`.

4. **Caller-type authorization**:
   - **Worker**: Look up the worker by OID and tenant ID in PostgreSQL. Reject if the worker is not in `APPROVED` status (`403 Forbidden`).
   - **User (Platform.Admin)**: If `roles` contains `Platform.Admin`, authorize immediately — global admins bypass per-tenant checks.
   - **User (Platform.User)**: Look up `tenant_members` WHERE `user_oid = {oid}` AND `tenant_id = {requested-tenant}`. If no row → `403 Forbidden`. If row exists → check that the member's `role` (owner/admin/operator/viewer) has sufficient permission for the endpoint.

5. **Inject verified context** — Set the verified gateway OID, caller OID, caller type, and tenant ID into the request context for downstream handlers. Log the auth result with both identities for audit.

---

## End-to-End Request Flows

### Flow A: User Approves a Worker

```
┌──────────────────────────────────────────────────────┐
│ User (Tenant Admin, normal Azure AD account)         │
│  Token: Azure AD JWT (roles=["TenantAdmin"])         │
│                                                      │
│  POST /api/tenants/{tid}/workers/{wid}/approve       │
│  Authorization: Bearer {user-jwt}                    │
└─────────────────────┬────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────┐
│ APIM Gateway                                         │
│  1. validate-jwt → valid, roles=["Platform.User"]   │
│  2. caller_type = "user"                             │
│  3. tenant_id from JWT tid claim                     │
│  4. rate-limit (100/min per tenant)                  │
│  5. authentication-managed-identity → APIM MI JWT    │
│  6. Forward:                                         │
│       Authorization: Bearer {user-jwt}               │
│       X-Gateway-Auth: Bearer {apim-mi-jwt}           │
│       X-Caller-OID: {user-oid}                       │
│       X-Caller-Type: user                            │
│       X-Tenant-ID: {tid}                             │
└─────────────────────┬────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────┐
│ API Service (az-rotator-api)                         │
│  1. Verify X-Gateway-Auth → APIM MI is trusted      │
│  2. Verify Authorization → user JWT is valid         │
│  3. roles=["Platform.User"] → lookup tenant_members  │
│  4. tenant_members role = admin or owner → allowed   │
│  5. Approve worker → status = APPROVED               │
│  6. Create Service Bus subscription for worker       │
│  7. Audit log: who approved, when, what              │
└──────────────────────────────────────────────────────┘
```

### Flow B: Worker Registers

```
┌──────────────────────────────────────────────────────┐
│ Worker (Container App, Managed Identity)             │
│  Token: Azure AD JWT (MI, no roles claim)            │
│                                                      │
│  POST /api/workers/register                          │
│  Authorization: Bearer {worker-jwt}                  │
│  X-Tenant-ID: {platform-tenant-uuid}                 │
└─────────────────────┬────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────┐
│ APIM Gateway                                         │
│  1. validate-jwt → valid, no roles                   │
│  2. caller_type = "worker"                           │
│  3. tenant_id from X-Tenant-ID header                │
│  4. rate-limit (10/min for registration endpoint)    │
│  5. authentication-managed-identity → APIM MI JWT    │
│  6. Forward:                                         │
│       Authorization: Bearer {worker-jwt}             │
│       X-Gateway-Auth: Bearer {apim-mi-jwt}           │
│       X-Caller-OID: {worker-mi-oid}                  │
│       X-Caller-Type: worker                          │
│       X-Tenant-ID: {platform-tenant-uuid}            │
└─────────────────────┬────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────┐
│ API Service (az-rotator-api)                         │
│  1. Verify X-Gateway-Auth → APIM MI is trusted      │
│  2. Verify Authorization → worker JWT is valid       │
│  3. No roles → worker path                           │
│  4. Create worker record (status = PENDING_APPROVAL) │
│  5. Return registration_id                           │
│  6. Audit log: registration attempt                  │
└──────────────────────────────────────────────────────┘
```

### Flow C: Worker Reports Rotation Status

```
┌──────────────────────────────────────────────────────┐
│ Worker (Approved, processing a job)                  │
│                                                      │
│  POST /api/rotations/{jobId}/status                  │
│  Authorization: Bearer {worker-jwt}                  │
│  X-Tenant-ID: {platform-tenant-uuid}                 │
│  Body: { "status": "COMPLETED", ... }                │
└─────────────────────┬────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────┐
│ APIM → same policy chain → API Service               │
│                                                      │
│  1. Dual auth verified                               │
│  2. Worker is APPROVED for tenant                    │
│  3. Job belongs to tenant (cross-check)              │
│  4. Update job status in PostgreSQL                  │
│  5. Audit log: rotation result                       │
└──────────────────────────────────────────────────────┘
```

---

## APIM Route Map

Routes exposed by APIM and which caller types are allowed:

| Method | Path | Caller | Purpose |
|--------|------|--------|---------|
| POST | `/api/tenants/register` | user | Create a new tenant |
| GET | `/api/tenants/{tid}` | user | Get tenant details |
| POST | `/api/workers/register` | worker | Register a new worker |
| POST | `/api/tenants/{tid}/workers/{wid}/approve` | user | Approve a pending worker |
| POST | `/api/tenants/{tid}/jobs` | user | Create a rotation job |
| POST | `/api/rotations/{jobId}/status` | worker | Report rotation job status |
| POST | `/api/workers/heartbeat` | worker | Worker heartbeat |
| GET | `/api/tenants/{tid}/jobs` | user | List jobs for a tenant |
| GET | `/api/tenants/{tid}/audit` | user | View audit logs |
| GET | `/api/policies/*` | user | Manage rotation policies |
| PUT | `/api/policies/*` | user | Update rotation policies |

---

## Security Summary

| Layer | What | How |
|-------|------|-----|
| **Client → APIM** | Caller authenticates to APIM | Azure AD JWT (MI for workers, org account for users) over HTTPS |
| **APIM (inbound)** | Validate client identity | `validate-jwt` policy — signature, issuer, audience, expiry, claims |
| **APIM (inbound)** | Detect caller type | Presence of `roles` claim → user; absence → worker |
| **APIM (inbound)** | Rate limiting | Per-tenant and per-endpoint rate limits |
| **APIM → Backend** | Gateway authenticates itself | APIM MI acquires its own JWT, injected in `X-Gateway-Auth` |
| **APIM → Backend** | Network security | VNET-integrated APIM → Internal LB → AKS (private path) |
| **Backend** | Verify gateway | Re-validate `X-Gateway-Auth` JWT, check OID matches known APIM MI |
| **Backend** | Verify client | Re-validate `Authorization` JWT (defense in depth) |
| **Backend** | Anti-tampering | Compare JWT claims to APIM-injected headers |
| **Backend** | Authorization | Users: `Platform.Admin` → global access, `Platform.User` → per-tenant role from `tenant_members`; Workers: check `APPROVED` status in DB |
| **In-cluster** | Pod-to-pod encryption | Cilium WireGuard — transparent kernel-level encryption |
| **Audit** | Traceability | Both gateway OID and caller OID logged on every request |

### What This Guards Against

- **Bypassed gateway**: Backend rejects requests without a valid APIM MI token
- **Stolen client JWT**: Rate limiting + short expiry + no static secrets
- **Header tampering**: Backend compares JWT claims against APIM-injected headers
- **Cross-tenant access**: Tenant ID enforced at APIM layer and re-verified at API layer
- **Unauthorized worker**: Worker must be in `APPROVED` status; registration alone grants no access
- **Unauthorized user**: Must have App Role (`Platform.Admin` or `Platform.User`) + per-tenant role for the endpoint
- **Replay attacks**: JWT expiry + job leasing (covered in ARCHITECTURE.md)
