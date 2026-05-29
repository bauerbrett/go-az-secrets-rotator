# go-az-secrets-rotator

A distributed secret rotation platform built in Go. A centralized control plane (AKS) orchestrates rotation jobs while tenant-owned worker agents (Container Apps) perform the actual secret rotations in their own environments.

## How It Works

Teams deploy lightweight **worker agents** in their own VNETs with scoped Managed Identities. Each worker only has access to the secrets it needs to rotate — no single over-permissioned service principal. Teams create **platform tenants** (a logical grouping per team, project, or environment) and register workers against them.

Workers register with the **control plane** through an APIM gateway. APIM authenticates every request (users and workers) via Azure AD JWT validation, then authenticates itself to the backend API using its own Managed Identity — a dual-auth model where the API pod always verifies both the gateway identity and the original caller.

Once a worker is approved by an admin, the control plane publishes rotation jobs to tenant-scoped **Azure Service Bus** topics. Workers subscribe to their own queue, lease a job, rotate the secret locally, validate it, and report status back.

The control plane **never** directly accesses tenant secrets. It orchestrates, schedules, enforces policies, and maintains a full audit trail. All state lives in **PostgreSQL**. All cluster traffic is encrypted by **Cilium** at the kernel level.

## Key Properties

- **No static credentials** — Managed Identity and Workload Identity everywhere
- **Tenant isolation** — scoped workers, per-tenant queues, row-level data separation
- **Defense in depth** — dual-auth at the gateway, JWT re-validation at the API, Cilium network policies
- **Pluggable rotation** — Key Vault, App Registrations, PostgreSQL, MySQL, GitHub PATs, Databricks, Storage Accounts, K8s Secrets
- **Observable** — OpenTelemetry, Prometheus, Grafana, Log Analytics, App Insights

## Documentation

Detailed design docs live in [`docs/`](docs/):

- [ARCHITECTURE.md](docs/ARCHITECTURE.md) — full architecture, service layout, infra phases
- [APIM_GATEWAY.md](docs/APIM_GATEWAY.md) — APIM dual-auth design, policy chain, backend middleware
- [TODO.md](docs/TODO.md) — documentation progress tracker

