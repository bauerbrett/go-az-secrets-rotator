# go-az-secrets-rotator
This is going to be a Go app that has a control plane and workers to rotate secrets in Azure. The goal is to be able to set up different scoped workers and onboard the onto the control plane, the workers can live in different VNETS and have their creds scoped individually to what they need to rotate. This allows us to centrally track and log rotations through the control plane but not have the overly permissive SP that needs access to every environment.

Users/teams should be able to onboard a worker and register it with the control plane and then start onboarding all their secrets that need rotating.

# Distributed Secret Rotation Platform

## Overview

A centralized Kubernetes-hosted control plane orchestrates secret rotation jobs for tenant-owned worker agents.

Workers:
- register with the platform
- subscribe to tenant-scoped queues
- rotate secrets locally
- report status and audit results

The platform never directly accesses tenant secrets.

---

# High-Level Architecture

```text
                    +----------------------------------+
                    | Kubernetes Control Plane         |
                    |----------------------------------|
                    | API Gateway                      |
                    | Scheduler                        |
                    | Worker Registry                  |
                    | Policy Engine                    |
                    | Audit Service                    |
                    | Rotation Coordinator             |
                    +----------------+-----------------+
                                     |
                                     v
                        +-------------------------+
                        | PostgreSQL              |
                        |-------------------------|
                        | Tenants                 |
                        | Rotation Policies       |
                        | Job Metadata            |
                        | Audit Logs              |
                        | Worker Registry         |
                        +-------------------------+

                                     |
                                     v

                        +-------------------------+
                        | Queue/Event Bus         |
                        |-------------------------|
                        | tenant-a-rotations      |
                        | tenant-b-rotations      |
                        | tenant-c-rotations      |
                        +-------------------------+

                     /               |                 \
                    /                |                  \

         +----------------+  +----------------+  +----------------+
         | Worker Agent   |  | Worker Agent   |  | Worker Agent   |
         | Team A         |  | Team B         |  | Team C         |
         |----------------|  |----------------|  |----------------|
         | Azure CA       |  | Azure CA       |  | VM/K8s/etc     |
         | Managed ID     |  | Managed ID     |  | Scoped Access  |
         | Local Rotation |  | Local Rotation |  | Local Rotation |
         +----------------+  +----------------+  +----------------+
```

---

# Core Components

## Control Plane (Kubernetes)

### Responsibilities
- tenant management
- worker registration
- scheduling rotations
- queue publishing
- policy enforcement
- audit logging
- retry orchestration
- metrics

### Suggested Technologies
- Go
- Kubernetes
- Conatiner Apps
- Helm
- gRPC
- OpenTelemetry
- Prometheus
- Grafana

---

## PostgreSQL

### Stores
- tenants
- worker registrations
- job metadata
- rotation history
- policies
- audit events
- leases

### Example Tables

```sql
tenants
workers
rotation_jobs
rotation_policies
audit_logs
job_leases
worker_heartbeats
```

---

## Queue/Event Bus

- NATS
OR
- Azure Service Bus

### Queue Model

```text
tenant-a-rotation-jobs
tenant-b-rotation-jobs
tenant-c-rotation-jobs
```

Workers only subscribe to their queue.

---

## Worker Agent

### Deployment
Tenant deploys:
- Azure Container App
- Kubernetes Deployment
- VM Container

### Startup Flow
1. Authenticate using Managed Identity or OIDC
2. Register with control plane
3. Obtain queue assignment
4. Begin polling/subscribing

### Runtime Flow
1. Receive rotation job
2. Lease job
3. Authenticate locally
4. Rotate secret
5. Validate secret
6. Store in tenant Key Vault
7. Report status

---

# Security Design

## Key Trust Boundary

The worker:
- has privileged access

The control plane:
- does NOT

The control plane orchestrates but never directly rotates secrets.

---

## Authentication
- Azure Managed Identity
- OIDC Federation
- JWT validation

No static credentials.

---

## Signed Job Payloads

Protect against:
- queue tampering
- spoofing
- replay attacks

---

## Job Leasing

Example:

```text
job_id: 123
leased_to: worker-a
lease_expiry: 12:05 UTC
```

If a worker crashes:
- lease expires
- another worker retries

---

## RBAC

Tenants can:
- manage policies
- onboard workers
- view jobs
- manage approvals

---

# Rotation Plugins

### Example Plugins
- Azure Key Vault
- Azure App Registrations
- PostgreSQL
- MySQL
- GitHub PAT
- Kubernetes Secrets
- Databricks PATs
- Storage Account Keys

---

# Reliability Features

## Retry State Machine

```text
PENDING
RUNNING
VALIDATING
COMPLETED
FAILED
RETRYING
DEAD_LETTERED
```

---

## Worker Heartbeats

```text
worker_id
last_seen
status
version
capabilities
```

---

# Observability

## Metrics
- rotation latency
- failure rates
- retries
- queue lag
- worker health

---

# Threat Modeling

Document:
- compromised worker
- queue poisoning
- token theft
- replay attacks
- malicious tenant
- privilege escalation
- stale leases

---

