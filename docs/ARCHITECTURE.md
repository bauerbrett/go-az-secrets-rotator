# Plan: AKS Control Plane + Distributed Worker Architecture

## TL;DR
Build a **centralized AKS control plane** in a dedicated VNET behind **APIM** (public endpoint) for external API access. **Container Apps workers** (VNET-integrated, customer-owned infrastructure) authenticate via **Azure Managed Identity → JWT tokens** validated by APIM. Workers communicate with control plane over public internet initially, designed for future private connectivity. Job distribution via **Azure Service Bus** topics/subscriptions. State in **PostgreSQL** (managed). Single-instance control plane (MVP). Monitoring via **Log Analytics** + **App Insights**. Admin approval required for worker onboarding.

---

## Architecture Layers

### 1. Network & Infrastructure
- **AKS Cluster**: VNET-injected in dedicated Azure VNET (e.g., 10.0.0.0/16)
  - **Subnet for AKS**: 10.0.1.0/24 (pods & nodes)
  - **Three namespaces** for control plane services: `az-rotator-api`, `az-rotator-scheduler`, `az-rotator-system`
  - Network policies enabled via **Cilium CNI** for micro-segmentation
  
- **APIM**: **Public endpoint** fronting the AKS service (dual-auth gateway)
  - Authenticates **both** client types: users (Azure AD App Reg / interactive) and workers (Managed Identity)
  - Validates client JWT up front (signature, issuer, audience, expiry, claims)
  - Detects caller type (user vs worker) based on JWT `roles` claim presence
  - Acquires its own **Managed Identity JWT** and injects it as `X-Gateway-Auth` so the backend can verify traffic came through the trusted gateway
  - Rate limiting, tenant context injection, header enrichment
  - Exposes endpoints: `/api/workers/register`, `/api/policies/*`, `/api/rotations/*`, `/api/tenants/*`, etc.
  - See [APIM_GATEWAY.md](APIM_GATEWAY.md) for full design
  - **Future**: Can transition to private endpoint + VNet integration when needed
  
- **Worker Container Apps**: VNET-integrated, customer-owned infrastructure
  - Authenticate via **Azure Managed Identity → JWT token** → present to APIM
  - APIM validates JWT issuer (Azure AD) + audience + tenant scope
  - **Initial**: Public communication via APIM
  - **Future**: Private connectivity via private endpoint or direct VNet routing

- **Users (Tenant Admins / Operators)**: Authenticate with normal Azure AD org accounts
  - Standard Azure AD interactive login (authorization code flow with PKCE)
  - JWT includes `roles` claim for platform RBAC via Azure AD App Roles (TenantAdmin, Operator)
  - Manage tenants, approve workers, create jobs, view audit logs
  
### 2. Data & Persistence
- **PostgreSQL (Azure Database for PostgreSQL)** — single instance MVP
  - Tables: Tenants, Tenant Members, Workers, Rotation Jobs, Policies, Audit Logs
  - A **platform tenant** = logical team/project/environment (not an Azure AD tenant). Multiple platform tenants can exist within one Azure AD org.
  - Accessed via **private endpoint** in AKS subnet for security
  
- **Alternative: Azure Cosmos DB** (if multi-region replication needed later)
  
### 3. Job Distribution & Messaging
- **Azure Service Bus** (recommended over Event Hubs for this use case):
  - Topics per tenant (e.g., `tenant-a-rotations`, `tenant-b-rotations`)
  - Each worker subscribes to its tenant topic
  - Control plane publishes rotation jobs as messages
  - Offers dead-letter queues, retry policies, ordering guarantees
  - Simpler SDK, better for request/reply patterns

- **Alternative: Event Hubs** (if extremely high throughput >10K msgs/sec needed)

### 4. Secrets & Identity
- **Azure Key Vault** (in customer VNETs or private to control plane):
  - Workers access their tenant secrets via Managed Identity RBAC (least privilege)
  - Control plane uses Managed Identity (AKS pod identity) for Azure service auth
  
- **JWT Authentication (Dual-Auth at APIM)**:
  - **Workers**: Container App has Managed Identity assigned → calls Azure AD for JWT (client_credentials)
  - **Users**: Authenticate via Azure AD App Registration → get JWT with `roles` claim
  - Both token types target the same audience (platform App Registration)
  - APIM validates caller JWT (issuer, audience, expiry, claims) up front
  - APIM acquires its own MI token and injects as `X-Gateway-Auth` header
  - Backend verifies **both** the APIM MI token (trusted gateway) and the client JWT (caller identity) — defense in depth
  
### 5. Monitoring & Observability
- **Azure Log Analytics**: Central logging for control plane pod logs, Service Bus metrics
- **Application Insights**: APM for control plane (latency, errors, dependencies)
- **Prometheus + Grafana** (optional): In-cluster metrics collection

---

## Control Plane Service Architecture

Three independent Go services deployed in AKS, each with dedicated namespace and strict pod security:

| Service | Namespace | Responsibility | Replicas | Security Posture |
|---------|-----------|-----------------|----------|------------------|
| **API** | `az-rotator-api` | REST endpoints (registration, approvals, job creation, audit queries); worker registry sync | 1 (MVP) | Workload Identity, ESO, no root, dropped caps, no priv esc |
| **Scheduler** | `az-rotator-scheduler` | Polls DB for unpublished jobs → publishes to Service Bus; worker health sync | 1 | Workload Identity, ESO, no root, dropped caps, no priv esc |
| **System Tasks** | `az-rotator-system` | Kubernetes CronJobs (audit cleanup, job reconciliation, retry handling) | 1 each | Workload Identity, ESO, no root, dropped caps, no priv esc |

### Pod Security & Compliance (All Services)
- **Workload Identity**: Azure AD pod identity (OIDC federation via AKS) — replaces legacy pod identity
- **External Secrets Operator (ESO)**: Secrets synced from Azure Key Vault (read-only, auto-refresh)
- **Cilium CNI**: Network policies for pod-to-pod + pod-to-external service communication
- **Pod Security Standards**:
  - `runAsNonRoot: true` (no root execution)
  - `allowPrivilegeEscalation: false`
  - `capabilities.drop: ["ALL"]`
  - `readOnlyRootFilesystem: true` (/tmp writable if needed)
  - Resource requests/limits enforced
  - Non-root user (UID 1000+) in all containers

### Service Communication Pattern
- **API ↔ PostgreSQL**: Private endpoint 
- **Scheduler ↔ Database**: Direct DB read (no API calls initially)
- **Scheduler ↔ Service Bus**: Managed Identity RBAC
- **API ↔ Service Bus**: Managed Identity RBAC  
- **External (APIM) → API**: Dual-auth — APIM validates client JWT + injects its own MI token; API re-validates both

---

## Steps

### Phase 1: Network & Core Infrastructure Setup
1. Create Azure VNET (10.0.0.0/16) with subnets:
   - AKS subnet (10.0.1.0/24)
   - APIM subnet (10.0.2.0/24, if APIM in same VNET)
   
2. Create **AKS cluster** with Cilium CNI, VNET injection, network policies enabled, Workload Identity enabled
   - Create three namespaces: `az-rotator-api`, `az-rotator-scheduler`, `az-rotator-system`
   
3. Create **APIM** instance (Premium or Standard tier)
   - Configure JWT validation policy (issuer = Azure AD, audience = APIM resource ID)
   - Set up policies for throttling, logging, tenant routing
   
4. Create **Azure Database for PostgreSQL** (Flexible Server)
   - Place in same VNET (private endpoint in AKS subnet)
   - Create initial schema (Tenants, Workers, Jobs, Policies, Audit Logs)

### Phase 2: Messaging & Worker Identity
5. Create **Azure Service Bus** namespace
   - Create tenant topic templates (can be auto-provisioned on worker registration)
   
6. Set up **Azure Workload Identity** for each service:
   - API service account (az-rotator-api namespace)
   - Scheduler service account (az-rotator-scheduler namespace)
   - System CronJob service accounts  (az-rotator-system namespace)
   - Each worker (external Container App or VM) gets tenant-scoped identity
   
7. Configure **External Secrets Operator**:
   - Install ESO CRDs and operator in cluster
   - Configure SecretStore for Key Vault (authenticate via Workload Identity)
   - Create ExternalSecret resources for each service's secrets

### Phase 3: Control Plane Service Deployment
8. Deploy three separate Go services in their respective namespaces:
   - **API Service** (`az-rotator-api`): REST server, registration, approvals, job creation, audit queries
   - **Scheduler** (`az-rotator-scheduler`): Polls DB for jobs, publishes to Service Bus, syncs worker health
   - **System CronJobs** (`az-rotator-system`): Background maintenance tasks (audit cleanup, reconciliation)
   
9. Apply Pod Security Policies (PSPs) or Pod Security Standards (PSS) to all namespaces:
   - Restricted profile: no root, dropped caps, read-only filesystem
   - All pods use non-root user
   
10. Configure **ingress** to APIM
    - Route `/api/workers/*`, `/api/policies/*`, `/api/rotations/*` to API service
    - APIM enforces JWT validation before routing

### Phase 4: Worker & Tenant Onboarding (Self-Service Setup)
11. Design **worker registration flow** (admin approval gate):
    - Tenant registers on control plane (creates Tenant record) → gets tenant ID + API key
    - Worker (Container App with MI) calls `POST /api/workers/register` with JWT token (from MI)
    - APIM policy validates JWT (issuer, audience, tenant claim)
    - Request routed to control plane API (az-rotator-api namespace)
    - Control plane creates Worker record in PostgreSQL with status `PENDING_APPROVAL`
    - API returns registration ID to worker
    - **Admin reviews** registration in control plane, approves
    - Upon approval, Worker status → `APPROVED`, subscribed to tenant topic in Service Bus
    - Tenant receives notification that worker is ready
    
12. **Self-service job creation (post-onboarding)**:
    - Tenant/admin can now call `POST /api/tenants/{tenantId}/jobs` (requires worker to be APPROVED)
    - Control plane validates: worker status = APPROVED for tenant
    - If validation passes, creates Rotation Job record in PostgreSQL
    - Publishes job message to Service Bus topic for tenant
    - Worker(s) subscribed to tenant topic receive and process job
    - Workers send status updates to `POST /api/rotations/{jobId}/status` (with JWT)

### Phase 5: Observability & Security
13. Enable **Azure Log Analytics** workspace integration:
    - Pod logs → Log Analytics via AKS integration
    - Service Bus logs → Log Analytics
    - APIM request logs → Log Analytics
    
14. Enable **Application Insights** for APM:
    - Instrument control plane with OpenTelemetry
    - Set up alerts for error rates, dependency timeouts, JWT validation failures

15. Implement **network policies** (Cilium):
    - API pod → PostgreSQL private endpoint only
    - Scheduler pod → PostgreSQL + Service Bus
    - Egress to Service Bus allowed (all pods)
    - Ingress from APIM layer only

### Phase 6: Validation & Hardening (Future)
16. Security validation:
    - MSFT Defender for Cloud scanning
    - Network policies audit
    - Managed identity RBAC audit

---

## Relevant Files (To Be Created)
- `deployment/kubernetes/namespaces.yaml` — Three namespaces: `az-rotator-api`, `az-rotator-scheduler`, `az-rotator-system`
- `deployment/kubernetes/pod-security-standards.yaml` — PSS Restricted profile
- `deployment/kubernetes/network-policies.yaml` — Cilium NetworkPolicy ingress/egress rules
- `deployment/kubernetes/workload-identity.yaml` — Workload Identity service accounts + OIDC binding
- `deployment/kubernetes/external-secrets.yaml` — ESO SecretStore + ExternalSecret resources
- `deployment/terraform/vnet.tf` — VNET, subnets, private endpoints
- `deployment/terraform/aks.tf` — AKS cluster config (Cilium CNI, Workload Identity enabled)
- `deployment/terraform/apim.tf` — APIM backend setup with JWT validation policy
- `deployment/terraform/service-bus.tf` — Service Bus namespace + RBAC assignments
- `deployment/terraform/postgresql.tf` — Database private endpoint + firewall
- `cmd/api/main.go` — Control plane API server
- `cmd/scheduler/main.go` — Job scheduler daemon
- `cmd/system-tasks/audit-cleanup/main.go` — CronJob: audit log cleanup
- `internal/worker/registry.go` — Worker registration logic + approval workflow
- `internal/queue/service_bus.go` — Service Bus client wrapper (MI auth)
- `internal/auth/workload_identity.go` — Workload Identity token retrieval
- `internal/audit/logger.go` — Audit logging
- `migrations/schema.sql` — Database schema (Tenants, Workers, Jobs, Policies, Audit)

---

## Key Decisions & Rationale

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Service Architecture** | Three separate services (API, Scheduler, System CronJobs) | Allows independent scaling, deployment, and resource management; separation of concerns |
| **Pod Security** | Restricted PSS, Workload Identity, ESO, Cilium | Defense-in-depth: no root, no privilege escalation, secrets in Key Vault, network isolation |
| **CNI** | Cilium | Superior network policies, encryption, observability; replaces Azure CNI for security |
| **Secrets Management** | Workload Identity + ESO | No secrets in pods; Azure AD-signed tokens; automatic rotation |
| **Messaging** | Azure Service Bus with MI auth | Topic/subscription model, tenant isolation, MI RBAC (no keys distributed) |
| **Database** | Azure Database for PostgreSQL (private endpoint) | GermanY, ACID compliance, cost-effective, private connectivity |
| **Worker Auth** | Managed Identity → JWT | No secrets; Azure AD-signed; APIM + control plane validation |
| **Onboarding Phase** | Tenant + Worker registration/approval → then jobs self-service | Ensures worker ready; prevents orphaned jobs; self-service after setup |

---

## Verification Checklist

1. **Workload Identity**: Pod can fetch JWT token from AAD OIDC endpoint
2. **ESO Integration**: Secrets synced from Key Vault into pod environment
3. **Cilium Policies**: Pod-to-pod communication validated; egress to Service Bus allowed
4. **Pod Security**: No container runs as root; capabilities dropped; no privilege escalation
5. **Network connectivity**: Pods reach PostgreSQL private endpoint, Service Bus
6. **APIM JWT validation**: APIM blocks invalid/expired tokens; routes valid ones to API
7. **Tenant Registration**: POST `/api/tenants/register` → tenant created with isolation
8. **Worker Registration**: Worker calls API with JWT → worker created with PENDING_APPROVAL status
9. **Admin Approval**: Admin approves worker → subscribed to Service Bus topic
10. **Self-Service Job Creation**: Tenant creates job → published to Service Bus → worker receives
11. **Observability**: Pod logs in Log Analytics; API traces in App Insights
12. **Scaling simulation**: Deploy 5 workers (5 tenants) → approve all → each creates 20 jobs → workers receive only their tenant jobs

---

## Implementation Roadmap

**Phase 1 (Infrastructure & Security)**: Terraform (VNET, AKS + Cilium/Workload Identity, APIM, Service Bus, PostgreSQL)
**Phase 2 (Kubernetes Setup)**: Namespaces, RBAC, PSS, Workload Identity, ESO integration
**Phase 3 (Control Plane Services)**: Three Go services (API, Scheduler, System CronJobs) with proper pod security
**Phase 4 (Observability)**: Log Analytics + App Insights + OpenTelemetry instrumentation
**Phase 5 (Worker Example)**: Reference Container App worker with Service Bus consumer + MI token handling
**Phase 6 (E2E Testing)**: Automated test suite (registration → approval → job → processing)