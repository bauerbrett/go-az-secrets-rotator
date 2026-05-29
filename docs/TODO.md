# Documentation & Design TODO

Track progress on detailed design docs for each component of the platform. Each doc should focus on the design, contracts, IaC, security posture, and implementation guidance for its area.

> **Legend**: `[ ]` not started | `[~]` in progress | `[x]` complete

---

## Core Architecture

- [x] [ARCHITECTURE.md](ARCHITECTURE.md) — High-level architecture, phases, service layout, infra plan
- [x] [APIM_GATEWAY.md](APIM_GATEWAY.md) — Dual-auth design (users + workers), APIM MI to backend, policy chain, middleware
- [x] [DATABASE.md](DATABASE.md) — Schema, tables, tenant isolation (RLS), access patterns, migrations

### Gaps in APIM doc to close out
- [ ] API versioning strategy (URI path vs header)
- [ ] OpenAPI / Swagger integration with APIM
- [ ] APIM diagnostics & logging to Log Analytics
- [ ] Error response contract (standardized error envelope)
- [ ] APIM Named Values & Policy Fragments for reuse
- [ ] Custom domain & TLS cert setup
- [ ] Backend circuit breaker / retry policies
- [ ] CORS policy (if admin UI or external clients)

---

## Control Plane Services

- [x] **API_SERVICE.md** — REST API design
  - [x] Endpoint catalog (routes, methods, request/response schemas)
  - [x] Middleware chain (auth, logging, tenant context, rate limiting)
  - [x] Worker registration flow (detailed sequence diagram)
  - [x] Admin approval workflow
  - [x] Job creation endpoint contract
  - [x] Health / readiness endpoints
  - [x] Error handling & response envelope
  - [x] Go project structure (`cmd/api`, `internal/api`, handlers, middleware)

- [x] **SCHEDULER.md** — Scheduler service design
  - [x] Job polling strategy (interval, batch size, backoff)
  - [x] Service Bus publishing logic
  - [x] Job state machine transitions (who moves what state)
  - [x] Worker health sync / heartbeat consumer
  - [x] Leader election (if scaling beyond 1 replica)
  - [x] Failure handling & dead-letter promotion
  - [x] Go project structure (`cmd/scheduler`)

- [x] **SYSTEM_TASKS.md** — CronJob / background task design
  - [x] Audit log cleanup (retention policy, batch delete)
  - [x] Job reconciliation (stale job recovery, orphan detection)
  - [x] Retry handler (exponential backoff, max attempts, dead-letter)
  - [x] Worker heartbeat expiry / mark-unhealthy
  - [x] CronJob manifests & schedules
  - [x] Go project structure (`cmd/system-tasks/`)

---

## Worker Agent

- [x] **WORKER_AGENT.md** — Worker design & lifecycle
  - [x] Startup flow (MI auth → register → subscribe)
  - [x] Runtime loop (poll/subscribe → rotate → report)
  - [x] Rotation plugin interface (how plugins are loaded/called)
  - [x] Secret validation after rotation
  - [x] Heartbeat reporting
  - [x] Graceful shutdown
  - [x] Error reporting to control plane
  - [x] Container App deployment model (VNET integration, MI setup)
  - [x] Go project structure (`cmd/worker`, `internal/worker`)

---

## Data & Messaging

- [ ] **DATABASE.md** — PostgreSQL schema & access patterns
  - Full schema (CREATE TABLE statements)
  - Table relationships & indexes
  - Tenant isolation strategy (row-level, schema-per-tenant, or shared)
  - Migration strategy (golang-migrate, atlas, or manual)
  - Connection pooling (pgbouncer or app-level)
  - Private endpoint setup
  - Backup & disaster recovery
  - Query patterns per service (API reads, scheduler reads, system writes)

- [x] **SERVICE_BUS.md** — Azure Service Bus design
  - [x] Topic/subscription model (per-tenant topics vs shared with filters)
  - [x] Message schema (rotation job payload, signed envelope)
  - [x] Dead-letter queue handling
  - [x] Retry policies & TTL
  - [x] Ordering guarantees (sessions if needed)
  - [x] Worker subscription lifecycle (create on approve, delete on deregister)
  - [x] Managed Identity RBAC (sender vs receiver roles)
  - Terraform for Service Bus namespace, topics, RBAC

---

## Security

- [ ] **AUTH_IDENTITY.md** — Authentication & identity design
  - Managed Identity setup (system vs user-assigned)
  - Workload Identity for AKS pods (OIDC federation)
  - JWT token flow (detailed claim mapping)
  - Token caching & refresh
  - External Secrets Operator setup (Key Vault → K8s secrets)
  - No static credentials policy enforcement

- [ ] **THREAT_MODEL.md** — Threat modeling
  - Trust boundaries diagram
  - Threat scenarios (compromised worker, queue poisoning, token theft, replay attacks, malicious tenant, privilege escalation, stale leases)
  - Mitigations per threat
  - Signed job payloads (signing key management, verification)
  - Job leasing as anti-replay
  - Network segmentation (Cilium policies)
  - RBAC audit checklist

---

## Rotation Engine

- [x] **ROTATION_PLUGINS.md** — Plugin system design
  - [x] Plugin interface definition (Go interface)
  - [x] Built-in plugins:
    - [x] Azure App Registration client secrets
    - [x] PostgreSQL passwords
    - [x] MySQL passwords
    - [x] GitHub PATs
    - [x] Databricks PATs
    - [x] Storage Account keys
  - [x] Plugin registration & discovery
  - [x] Validation step (post-rotate connectivity check)
  - [x] Plugin configuration schema (per-secret-type config)
  - [x] Adding custom plugins (extension guide)

- [x] **ROTATION_POLICIES.md** — Policy engine design
  - [x] Policy schema (rotation interval, retry config, notification prefs)
  - [x] Policy assignment (per-secret, per-tenant, global defaults)
  - [x] Policy evaluation in scheduler
  - [x] Compliance reporting (overdue rotations, failed policies)

---

## Operations & Observability

- [x] **OBSERVABILITY.md** — Monitoring & alerting
  - [x] OpenTelemetry instrumentation (traces, metrics, logs)
  - [x] Prometheus metrics catalog (rotation latency, failure rates, queue lag, worker health)
  - [x] Grafana dashboards (control plane health, rotation status, tenant overview)
  - [x] Log Analytics integration (pod logs, APIM logs, Service Bus logs)
  - [x] Application Insights APM
  - [x] Alerting rules (error rate spikes, heartbeat failures, queue depth)
  - [x] Audit log schema & retention

- [x] **DEPLOYMENT.md** — IaC & deployment
  - [x] Terraform modules (VNET, AKS, APIM, Service Bus, PostgreSQL, Key Vault, ACR)
  - [x] Kubernetes manifests (namespaces, PSS, network policies, workload identity, ESO)
  - [x] CI/CD pipeline design (GitHub Actions: build, scan, deploy)
  - [x] Environment strategy (dev, staging, prod)
  - [x] Container image build (multi-stage Dockerfile) & ACR registry
  - [x] Rollback strategy
  - [x] Health checks & readiness gates

---

## Tenant Experience

- [ ] **ONBOARDING.md** — Tenant & worker onboarding
  - Tenant registration flow
  - Worker onboarding (self-service + admin approval gate)
  - First rotation setup walkthrough
  - RBAC roles (tenant admin, operator, viewer)
  - API key / credential provisioning
  - Offboarding & cleanup

---

## Priority Order (Suggested)

1. **APIM_GATEWAY.md** — close out remaining gaps
2. **DATABASE.md** — schema drives everything else
3. **API_SERVICE.md** — core control plane endpoints
4. **SERVICE_BUS.md** — messaging contract
5. **WORKER_AGENT.md** — rotation execution
6. **SCHEDULER.md** — job orchestration
7. **AUTH_IDENTITY.md** — consolidated auth design
8. **ROTATION_PLUGINS.md** — plugin interface
9. **THREAT_MODEL.md** — security review
10. **OBSERVABILITY.md** — monitoring
11. **DEPLOYMENT.md** — IaC & CI/CD
12. **ONBOARDING.md** — tenant experience
13. **SYSTEM_TASKS.md** — background maintenance
14. **ROTATION_POLICIES.md** — policy engine
