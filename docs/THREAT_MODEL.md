# Threat Model

## 1. Scope & Trust Boundaries

### In Scope

- Control plane (API Service, Scheduler, System Tasks)
- Data store (PostgreSQL)
- Messaging (Azure Service Bus)
- Worker agents (Container Apps in tenant VNETs)
- Secret storage (Key Vault — platform and tenant-owned)
- Gateway (APIM)
- Identity (Azure AD, Managed Identity, Workload Identity)

### Trust Boundaries

| Boundary | What Crosses It | Controls |
|----------|----------------|----------|
| Internet → APIM | User JWTs, Worker JWTs | |
| APIM → AKS (API Service) | Validated requests + gateway MI token | |
| AKS → PostgreSQL | SQL over private endpoint | |
| Scheduler → Service Bus | Rotation job messages | |
| Service Bus → Worker | Job messages (subscription) | |
| Worker → APIM | Status callbacks, heartbeats | |
| ESO → Key Vault | Secret sync | |
| Worker → Tenant Key Vault | Rotated credential writes | |
| Worker → Target Resources | New credential application | |

### Assumptions

- Azure fabric is trusted (hypervisor, physical network, Azure AD token service)
- Managed Identity tokens cannot be forged outside Azure
- TLS is enforced on all connections (no plaintext paths)
- AKS node pool is not shared with other workloads
-

---

## 2. Threat Actors

| Actor | Motivation | Capabilities |
|-------|-----------|--------------|
| **External attacker** | Data theft, disruption | Internet access, public endpoint probing |
| **Malicious tenant admin** | Access other tenants' data | Valid JWT, API access scoped to their tenant |
| **Compromised worker** | Lateral movement, data exfil | MI token, Service Bus access, network position in tenant VNET |
| **Malicious insider (platform operator)** | Abuse privileges | kubectl access, Azure portal, deployment pipeline |
| **Supply chain** | Backdoor, crypto mining | Compromised base image or dependency |

---

## 3. STRIDE Analysis Per Component

### 3.1 APIM Gateway

| Threat | Category | Description | Mitigation | Residual Risk |
|--------|----------|-------------|------------|---------------|
| | Spoofing | | | |
| | Tampering | | | |
| | Repudiation | | | |
| | Info Disclosure | | | |
| | DoS | | | |
| | Elevation | | | |

### 3.2 API Service

| Threat | Category | Description | Mitigation | Residual Risk |
|--------|----------|-------------|------------|---------------|
| | Spoofing | | | |
| | Tampering | | | |
| | Repudiation | | | |
| | Info Disclosure | | | |
| | DoS | | | |
| | Elevation | | | |

### 3.3 Scheduler

| Threat | Category | Description | Mitigation | Residual Risk |
|--------|----------|-------------|------------|---------------|
| | Spoofing | | | |
| | Tampering | | | |
| | Repudiation | | | |
| | Info Disclosure | | | |
| | DoS | | | |
| | Elevation | | | |

### 3.4 System Tasks

| Threat | Category | Description | Mitigation | Residual Risk |
|--------|----------|-------------|------------|---------------|
| | Spoofing | | | |
| | Tampering | | | |
| | Repudiation | | | |
| | Info Disclosure | | | |
| | DoS | | | |
| | Elevation | | | |

### 3.5 Service Bus

| Threat | Category | Description | Mitigation | Residual Risk |
|--------|----------|-------------|------------|---------------|
| | Spoofing | | | |
| | Tampering | | | |
| | Repudiation | | | |
| | Info Disclosure | | | |
| | DoS | | | |
| | Elevation | | | |

### 3.6 PostgreSQL

| Threat | Category | Description | Mitigation | Residual Risk |
|--------|----------|-------------|------------|---------------|
| | Spoofing | | | |
| | Tampering | | | |
| | Repudiation | | | |
| | Info Disclosure | | | |
| | DoS | | | |
| | Elevation | | | |

### 3.7 Worker Agent

| Threat | Category | Description | Mitigation | Residual Risk |
|--------|----------|-------------|------------|---------------|
| | Spoofing | | | |
| | Tampering | | | |
| | Repudiation | | | |
| | Info Disclosure | | | |
| | DoS | | | |
| | Elevation | | | |

### 3.8 Key Vault / ESO

| Threat | Category | Description | Mitigation | Residual Risk |
|--------|----------|-------------|------------|---------------|
| | Spoofing | | | |
| | Tampering | | | |
| | Repudiation | | | |
| | Info Disclosure | | | |
| | DoS | | | |
| | Elevation | | | |

---

## 4. Data Flow Threats

### 4.1 APIM → API Service

**Question**: What if someone bypasses the gateway and hits the API directly?

- Threat:
- Mitigation:
- Residual:

### 4.2 Scheduler → Service Bus (Job Publishing)

**Question**: What if a rogue process publishes fake rotation jobs?

- Threat:
- Mitigation:
- Residual:

### 4.3 Service Bus → Worker (Job Consumption)

**Question**: What if a worker subscribes to another tenant's topic?

- Threat:
- Mitigation:
- Residual:

### 4.4 Worker → APIM (Status Callbacks)

**Question**: What if a compromised worker sends fake COMPLETED statuses?

- Threat:
- Mitigation:
- Residual:

### 4.5 Worker → Tenant Key Vault (Secret Write)

**Question**: What if the worker writes incorrect/malicious values?

- Threat:
- Mitigation:
- Residual:

### 4.6 Worker → Target Resource (Credential Application)

**Question**: What if a compromised worker uses its MI to access resources beyond rotation scope?

- Threat:
- Mitigation:
- Residual:

---

## 5. Tenant Isolation Threats

### 5.1 RLS Bypass

**Question**: Under what conditions could tenant A read tenant B's data?

- Scenarios:
- Mitigations:
- Residual:

### 5.2 Cross-Tenant Topic Access

**Question**: Can a worker subscribe to or peek another tenant's Service Bus topic?

- Scenarios:
- Mitigations:
- Residual:

### 5.3 Shared Infrastructure Side-Channels

**Question**: What shared resources exist and could they leak information?

- Shared resources:
- Mitigations:
- Residual:

---

## 6. Mitigations Summary

| # | Threat | Control | Layer |
|---|--------|---------|-------|
| M1 | | | Edge |
| M2 | | | Network |
| M3 | | | Application |
| M4 | | | Data |
| M5 | | | Operational |

---

## 7. Residual Risks & Accepted Risks

| # | Risk | Severity | Likelihood | Acceptance Reason |
|---|------|----------|-----------|-------------------|
| R1 | | | | |
| R2 | | | | |
| R3 | | | | |

---

## 8. Future Hardening (Not in MVP)

| Enhancement | Threat Addressed | Priority |
|-------------|-----------------|----------|
| | | |
| | | |
| | | |
| | | |
