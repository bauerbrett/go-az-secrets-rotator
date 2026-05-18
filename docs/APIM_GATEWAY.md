# APIM Gateway - Detailed Design

## Overview
APIM acts as the **public entry point** for all worker and user requests. It handles:
1. **JWT validation** (worker Managed Identity tokens)
2. **Tenant isolation** enforcement (routing by tenant claim)
3. **Rate limiting & throttling**
4. **Request transformation** (headers, tenant context injection)
5. **Secure forwarding** to AKS API service

---

## JWT Token Flow

### Step 1: Worker Obtains JWT from Azure AD
```
Worker Container App (with Managed Identity)
    ↓
1. Calls: GET https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token
   Headers: 
     - client_id: {worker_managed_identity_client_id}
   Body:
     - grant_type: client_credentials
     - scope: https://{apim-resource-id}/.default
     - client_assertion: (signed by worker's MI key)
    ↓
2. Azure AD returns JWT:
   {
     "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
     "expires_in": 3600,
     "token_type": "Bearer"
   }
    ↓
3. JWT Payload (decoded):
   {
     "iss": "https://login.microsoftonline.com/{tenantId}/v2.0",
     "aud": "https://{apim-resource-id}",
     "oid": "{worker-mi-object-id}",
     "sub": "{worker-mi-subject}",
     "tenant_id": "{azure-tenant-id}",
     "appid": "{application-id}",
     "exp": 1716384123  (3600 seconds from now)
   }
```

### Step 2: Worker Sends Request to APIM with JWT
```
Worker Container App
    ↓
POST https://{apim-endpoint}/api/workers/register
Headers:
  - Authorization: Bearer {jwt_token}
  - Content-Type: application/json
Body:
  {
    "worker_name": "worker-prod-01",
    "region": "eastus",
    "tags": {"env": "prod"}
  }
```

---

## APIM JWT Validation & Policy Chain

### Policy 1: Validate JWT Signature & Claims
```xml
<!-- APIM Inbound Policy -->
<policies>
  <inbound>
    <!-- Step 1: Extract JWT from Authorization header -->
    <validate-jwt 
        header-name="Authorization" 
        failed-validation-httpcode="401" 
        failed-validation-error-message="Invalid or missing JWT">
      
      <!-- Step 2: Define issuer (Azure AD) -->
      <issuer>https://login.microsoftonline.com/{tenant-id}/v2.0</issuer>
      
      <!-- Step 3: Define audience (APIM resource ID) -->
      <audiences>
        <audience>https://{apim-resource-id}</audience>
      </audiences>
      
      <!-- Step 4: Define signing keys (from Azure AD metadata) -->
      <signing-keys>
        <key>{key-from-azure-ad-jwks-endpoint}</key>
      </signing-keys>
      
      <!-- Step 5: Extract claims into context for later use -->
      <claim name="oid" />
      <claim name="sub" />
      <claim name="appid" />
    </validate-jwt>
    
    <!-- Step 6: Extract tenant claim and validate it exists -->
    <set-variable name="tenant_id" value="@{context.Request.Headers.GetValueOrDefault("X-Tenant-ID", "")}" />
    
    <!-- Step 7: Reject if tenant_id is missing or invalid -->
    <choose>
      <when condition="@(string.IsNullOrEmpty((string)context.Variables["tenant_id"]))">
        <return-response>
          <set-status code="400" reason="Bad Request" />
          <set-body>{ "error": "X-Tenant-ID header is required" }</set-body>
        </return-response>
      </when>
    </choose>
  </inbound>
</policies>
```

**What This Does**:
1. ✅ Validates JWT signature (ensures it's signed by Azure AD)
2. ✅ Validates issuer (ensures it came from Azure AD)
3. ✅ Validates audience (ensures it's intended for APIM)
4. ✅ Validates expiry (rejects expired tokens)
5. ✅ Extracts claims (`oid`, `sub`, `appid`) for downstream use
6. ✅ Requires tenant context header

---

### Policy 2: Tenant Isolation - Extract & Validate Tenant Claim
```xml
<policies>
  <inbound>
    <!-- After JWT validation above -->
    
    <!-- Extract worker MI's object ID from JWT -->
    <set-variable name="worker_oid" value="@{(string)context.Request.Headers.GetValueOrDefault("Authorization").Split(' ')[1]}" />
    
    <!-- Look up worker in database via API backend to get tenant -->
    <!-- OR: Worker provides tenant in X-Tenant-ID header + we validate it matches their MI -->
    
    <!-- Option A: Simple - trust X-Tenant-ID (less secure, assumes client honesty) -->
    <set-variable name="tenant_from_header" value="@{context.Request.Headers.GetValueOrDefault("X-Tenant-ID", "")}" />
    
    <!-- Option B: Secure - validate worker MI ownership of tenant -->
    <!-- Call internal API to verify: {worker_oid} ∈ {tenant.approved_workers} -->
    <send-request mode="new" response-variable-name="worker-validation-response">
      <set-url>@{new Uri(new Uri("https://api-service:8080"), "/internal/validate-worker").AbsoluteUri}</set-url>
      <set-method>POST</set-method>
      <set-header name="Content-Type" value="application/json" />
      <set-body>@{
        new JObject(
          new JProperty("worker_oid", (string)context.Variables["worker_oid"]),
          new JProperty("tenant_id", context.Request.Headers.GetValueOrDefault("X-Tenant-ID", ""))
        ).ToString()
      }</set-body>
    </send-request>
    
    <!-- Check validation response -->
    <choose>
      <when condition="@{((IResponse)context.Variables["worker-validation-response"]).StatusCode != 200}">
        <return-response>
          <set-status code="403" reason="Forbidden" />
          <set-body>{ "error": "Worker not approved for this tenant" }</set-body>
        </return-response>
      </when>
    </choose>
    
    <!-- If validation passed, inject tenant context for downstream services -->
    <set-header name="X-Tenant-ID" value="@{context.Request.Headers.GetValueOrDefault("X-Tenant-ID", "")}" />
    <set-header name="X-Worker-OID" value="@{(string)context.Variables["worker_oid"]}" />
    
  </inbound>
</policies>
```

**What This Does**:
1. ✅ Extracts worker MI's object ID (`oid`) from JWT
2. ✅ Validates that worker is registered for the requested tenant
3. ✅ Injects tenant & worker context into headers for API service to consume
4. ✅ Rejects if worker not authorized for tenant (403 Forbidden)

---

### Policy 3: Rate Limiting & Throttling (Optional)
```xml
<policies>
  <inbound>
    <!-- Rate limit per tenant to prevent single tenant hogging APIM -->
    <rate-limit-by-key 
        calls="100" 
        renewal-period="60" 
        counter-key="@{context.Request.Headers.GetValueOrDefault("X-Tenant-ID", "unknown")}" 
        increment-condition="@{context.Response.StatusCode < 400}">
    </rate-limit-by-key>
  </inbound>
</policies>
```

---

## APIM → API Service: Secure Forwarding (mTLS + Cilium)

### Architecture: APIM → Internal LB (mTLS) → AKS Cluster (Cilium mTLS internal)

```
APIM (Azure Managed Service)
    ↓ mTLS HTTPS (APIM has client cert)
Azure Internal Load Balancer (in VNET)
    - Frontend: HTTPS port 443 (server cert)
    - Backend pool: AKS nodes or service endpoints
    ↓ Routes to backend
AKS Cluster (Private Network)
    ↓ Ingress / Kubernetes Service (terminates LB HTTPS)
    ↓ Cilium network layer (encrypts all pod traffic)
API Pod (listens on plain HTTP:8080, Cilium encrypts automatically)
```

### Step 1: APIM mTLS Configuration to Internal Load Balancer

**APIM Backend** (points to Internal LB):
```json
{
  "type": "Microsoft.ApiManagement/service/backends",
  "name": "api-service-backend",
  "properties": {
    "url": "https://api-internal-lb.{vnet-name}.private:443",
    "protocol": "https",
    "tls": {
      "validateCertificateChain": true,
      "validateCertificateName": true
    },
    "credentials": {
      "clientCertificate": "{base64-pfx-with-apim-client-cert}"
    }
  }
}
```

**APIM Policy** (use the backend):
```xml
<set-backend-service 
    base-url="https://api-internal-lb.{vnet-name}.private:443" />
```

**Why**:
- ✅ mTLS between APIM (public) and Internal LB ensures encrypted, authenticated entry to private network
- ✅ APIM client cert validates APIM to LB
- ✅ Internal LB is in your VNET (private, not internet-exposed)

### Step 2: Internal Load Balancer Configuration

```hcl
# Terraform
resource "azurerm_lb" "internal" {
  name                = "api-internal-lb"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  load_balancer_sku   = "Standard"

  frontend_ip_configuration {
    name                          = "api-frontend"
    subnet_id                     = azurerm_subnet.aks_subnet.id
    private_ip_address_allocation = "Static"
    private_ip_address            = "10.0.1.10"
  }
}

resource "azurerm_lb_backend_address_pool" "api" {
  loadbalancer_id = azurerm_lb.internal.id
  name            = "api-backend-pool"
}

resource "azurerm_lb_probe" "api" {
  loadbalancer_id = azurerm_lb.internal.id
  name            = "api-health"
  port            = 8080
  protocol        = "http"
  request_path    = "/health"
}

resource "azurerm_lb_rule" "api" {
  loadbalancer_id            = azurerm_lb.internal.id
  name                       = "api-443"
  protocol                   = "Tcp"
  frontend_port              = 443
  backend_port               = 8080  # Ingress listens on 8080
  frontend_ip_configuration_name = "api-frontend"
  backend_address_pool_ids   = [azurerm_lb_backend_address_pool.api.id]
  probe_id                   = azurerm_lb_probe.api.id
}
```

### Step 3: AKS Ingress (Terminates LB HTTPS, Routes to API Pod)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: az-rotator-api
spec:
  ingressClassName: nginx
  # Ingress terminates HTTPS from LB
  tls:
    - hosts:
        - api-internal-lb.{vnet-name}.private
      secretName: api-tls-cert  # Server cert for LB
  rules:
    - host: api-internal-lb.{vnet-name}.private
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080  # API pod plain HTTP
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: az-rotator-api
spec:
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: api
```

### Step 4: API Pod (Plain HTTP, Cilium Handles Encryption)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
  namespace: az-rotator-api
  labels:
    app: api
spec:
  serviceAccountName: api-sa
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: api
      image: {registry}/api:latest
      ports:
        - name: http
          containerPort: 8080
          protocol: TCP
      env:
        - name: PORT
          value: "8080"
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
        readOnlyRootFilesystem: true
      resources:
        requests:
          memory: "256Mi"
          cpu: "100m"
        limits:
          memory: "512Mi"
          cpu: "500m"
```

**API Pod Code** (plain HTTP, no TLS):
```go
// main.go
func main() {
  server := &http.Server{
    Addr:    ":8080",  // Plain HTTP
    Handler: setupRoutes(),
  }
  
  if err := server.ListenAndServe(); err != nil {
    log.Fatalf("Server error: %v", err)
  }
}
```

### Step 5: Cilium mTLS for Pod-to-Pod Encryption

**Enable Cilium with mTLS** (at AKS cluster creation):

```hcl
# Terraform
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "aks-control-plane"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  
  # Use Cilium CNI
  network_profile {
    network_plugin = "cilium"
    ebpf_data_plane = "cilium"
  }
  
  # Enable Cilium mTLS for pod encryption
  azure_policy_enabled = true
}
```

**Or via Helm**:
```bash
helm repo add cilium https://helm.cilium.io
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set hubble.enabled=true \
  --set hubble.ui.enabled=true \
  --set encryption.enabled=true \
  --set encryption.type=wireguard
```

**Cilium NetworkPolicy** (enforces mTLS within cluster):
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-encryption
  namespace: az-rotator-api
spec:
  description: "Encrypt all API pod traffic with Cilium mTLS"
  endpointSelector:
    matchLabels:
      app: api
  ingress:
    - fromEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: ingress-nginx
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            l7:
              - rule: "HTTP"
  egress:
    - toEndpoints:
        - matchLabels:
            app: api
      toPorts:
        - ports:
            - port: "5432"  # PostgreSQL
              protocol: TCP
```

**Why Cilium mTLS is better**:
- ✅ No app-level TLS complexity (API pod = plain HTTP:8080)
- ✅ Automatic encryption for ALL pod traffic (not just API)
- ✅ Cilium manages certificates transparently (no key rotation burden)
- ✅ NetworkPolicy enforces encryption policies at network layer
- ✅ Works with any protocol (HTTP, gRPC, etc.)

---

## Request Flow: Two-Layer Authentication Model

### Layer 1: Client (Worker) → APIM
```
Worker (Managed Identity #1)
  ↓
1. Get JWT from Azure AD (scope = APIM resource ID)
  ↓
2. POST https://{apim-endpoint}/api/workers/register
   Authorization: Bearer {Worker JWT}
   X-Tenant-ID: {tenant-uuid}
```

### Layer 2: APIM → Backend
```
APIM validates Worker JWT + rate limits
  ↓
APIM uses its Managed Identity (MI #2) to authenticate to backend
  ↓
APIM forwards original Worker JWT in request (+tenant context)
  ↓
REQUEST TO BACKEND:
  Authorization: Bearer {Worker JWT}  ← Client identity (unchanged)
  X-Tenant-ID: {tenant-uuid}         ← Injected by APIM
  X-Worker-OID: {worker-oid}         ← Injected by APIM
  X-APIM-Auth: Bearer {APIM MI JWT}  ← Backend verifies APIM is trusted
```

### Layer 3: Backend (API Pod) Validates Both Identities

**Step 1: Verify APIM is trusted (connection source)**
```go
apimJWT := extractJWT(r.Header.Get("X-APIM-Auth"))
if !isValidJWT(apimJWT, expectedAPIMResourceID) {
  http.Error(w, "Untrusted gateway", 403)
  return
}
log.Infof("gateway_verified: apim_oid=%s", apimJWT.OID)
```

**Step 2: Verify Worker identity + check for tampering**
```go
workerJWT := extractJWT(r.Header.Get("Authorization"))
if !isValidJWT(workerJWT) {
  http.Error(w, "Invalid worker token", 401)
  return
}

workerOID := workerJWT.OID
if workerOID != r.Header.Get("X-Worker-OID") {
  http.Error(w, "Token mismatch (tampering detected)", 401)
  return
}
```

**Step 3: Verify worker is approved for tenant**
```go
tenantID := r.Header.Get("X-Tenant-ID")
worker, err := db.GetWorker(tenantID, workerOID)
if err != nil || worker.Status != "APPROVED" {
  http.Error(w, "Worker not approved", 403)
  return
}
```

**Step 4: Process request using Worker claims**
```go
// All actions scoped to: tenant_id + worker_oid
log.Infof("auth_success: apim_oid=%s, worker_oid=%s, tenant=%s",
          apimJWT.OID, workerOID, tenantID)
// Continue with business logic...
```

---

## API Service Authentication & Authorization

### Authentication Middleware (Validates Both Layers)
The API service **receives validated headers from APIM** and verifies both identities:

#### Check 1: Verify APIM Gateway Authentication
```go
// In API service middleware
func AuthenticationMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    // ========== IDENTITY #1: APIM (Gateway) ==========
    apimAuthHeader := r.Header.Get("X-APIM-Auth")
    if apimAuthHeader == "" {
      http.Error(w, "Missing gateway authentication", http.StatusUnauthorized)
      return
    }
    
    apimTokenString := strings.TrimPrefix(apimAuthHeader, "Bearer ")
    apimClaims := jwt.MapClaims{}
    apimToken, err := jwt.ParseWithClaims(apimTokenString, apimClaims, func(token *jwt.Token) (interface{}, error) {
      return getAzureADPublicKey(token)
    })
    
    if err != nil || !apimToken.Valid {
      log.Warnf("Invalid APIM token: %v", err)
      http.Error(w, "Invalid gateway token", http.StatusUnauthorized)
      return
    }
    
    // Verify APIM token is for the right resource (ILB/backend)
    apimAudience := apimClaims["aud"].(string)
    if apimAudience != expectedBackendResourceID {
      http.Error(w, "Invalid token audience", http.StatusUnauthorized)
      return
    }
    
    apimOID := apimClaims["oid"].(string)
    log.Infof("gateway_authenticated: apim_oid=%s", apimOID)
    
    // ========== IDENTITY #2: Worker (Client) ==========
    workerAuthHeader := r.Header.Get("Authorization")
    if workerAuthHeader == "" {
      http.Error(w, "Missing worker authorization", http.StatusUnauthorized)
      return
    }
    
    workerTokenString := strings.TrimPrefix(workerAuthHeader, "Bearer ")
    workerClaims := jwt.MapClaims{}
    workerToken, err := jwt.ParseWithClaims(workerTokenString, workerClaims, func(token *jwt.Token) (interface{}, error) {
      return getAzureADPublicKey(token)
    })
    
    if err != nil || !workerToken.Valid {
      log.Warnf("Invalid worker token: %v", err)
      http.Error(w, "Invalid worker token", http.StatusUnauthorized)
      return
    }
    
    workerOID := workerClaims["oid"].(string)
    log.Infof("worker_authenticated: oid=%s", workerOID)
    
    // ========== CONSISTENCY CHECK: Detect Tampering ==========
    tenantID := r.Header.Get("X-Tenant-ID")
    headerWorkerOID := r.Header.Get("X-Worker-OID")
    
    if tenantID == "" || headerWorkerOID == "" {
      http.Error(w, "Missing tenant context", http.StatusBadRequest)
      return
    }
    
    // Verify JWT claims match APIM-injected headers (catch header tampering)
    if workerOID != headerWorkerOID {
      log.Warnf("OID mismatch: jwt=%s, header=%s (tampering attempt)", workerOID, headerWorkerOID)
      http.Error(w, "Token/header mismatch", http.StatusUnauthorized)
      return
    }
    
    // ========== AUTHORIZATION: Is worker approved for tenant? ==========
    worker, err := db.GetWorker(tenantID, workerOID)
    if err != nil {
      log.Errorf("Worker lookup failed: %v", err)
      http.Error(w, "Authorization check failed", http.StatusInternalServerError)
      return
    }
    
    if worker == nil || worker.Status != "APPROVED" {
      log.Warnf("Worker not approved: tenant=%s, oid=%s, status=%s",
                tenantID, workerOID, worker.Status)
      http.Error(w, "Worker not approved for tenant", http.StatusForbidden)
      return
    }
    
    // ========== SUCCESS: Inject context for handlers ==========
    log.Infof("auth_success: gateway_oid=%s, worker_oid=%s, tenant_id=%s",
              apimOID, workerOID, tenantID)
    
    r.Header.Set("X-Auth-Status", "verified")
    r.Header.Set("X-APIM-OID", apimOID)
    r.Header.Set("X-Worker-OID", workerOID)
    r.Header.Set("X-Tenant-ID", tenantID)
    
    next.ServeHTTP(w, r)
  })
}
```

#### Check 2: Tenant-Scoped Authorization (in handler)
```go
func RegisterWorkerHandler(w http.ResponseWriter, r *http.Request) {
  tenantID := r.Header.Get("X-Tenant-ID")
  workerOID := r.Header.Get("X-Worker-OID")
  
  // Query: Is this worker already registered for this tenant?
  existingWorker, err := db.GetWorker(tenantID, workerOID)
  if err != nil && err != ErrNotFound {
    http.Error(w, "Database error", http.StatusInternalServerError)
    return
  }
  
  if existingWorker != nil {
    http.Error(w, "Worker already registered", http.StatusConflict)
    return
  }
  
  // Create worker record
  worker := &Worker{
    TenantID:  tenantID,
    WorkerOID: workerOID,
    Status:    "PENDING_APPROVAL",
    CreatedAt: time.Now(),
  }
  
  err = db.CreateWorker(worker)
  if err != nil {
    http.Error(w, "Failed to register worker", http.StatusInternalServerError)
    return
  }
  
  // Return registration ID
  w.Header().Set("Content-Type", "application/json")
  json.NewEncoder(w).Encode(map[string]string{
    "registration_id": worker.ID,
    "status": worker.Status,
  })
}
```

---

## Decision: Trust Model at API Service

### Option A: Trust APIM (Simplified)
```
API validates:
  ✓ X-Tenant-ID header presence
  ✓ X-Worker-OID header presence
API assumes:
  ✓ APIM already validated JWT
  ✓ APIM already verified worker belongs to tenant
```

**Pros**:
- Simpler code
- Faster (no JWT re-validation)

**Cons**:
- No defense if APIM is compromised
- Less audit trail

### Option B: Defense in Depth (Recommended)
```
API validates:
  ✓ Authorization header contains valid JWT
  ✓ JWT signature is valid (Azure AD keys)
  ✓ JWT claims (iss, aud, exp) are valid
  ✓ Claims match X-Tenant-ID, X-Worker-OID headers (anti-tampering)
  ✓ Worker is in "APPROVED" status for tenant
```

**Pros**:
- More secure (doesn't trust APIM alone)
- Better audit trail (original JWT retained)
- Detects header tampering

**Cons**:
- More complex code
- Slight performance hit (JWT validation on every request)

---

## Recommended Architecture (End-to-End Encryption)

```
┌─────────────────────────────────────────────────────────────────┐
│ WORKER (Container App with Managed Identity)                    │
│  1. Get JWT from Azure AD: GET /token (scope=APIM)             │
│  2. Call APIM: POST /api/workers/register (JWT in header)      │
└─────────────────────┬───────────────────────────────────────────┘
                      │ HTTPS Public Internet
                      │ Authorization: Bearer {JWT}
                      │ X-Tenant-ID: {uuid}
                      │
┌─────────────────────▼───────────────────────────────────────────┐
│ APIM GATEWAY (Public Endpoint)                                  │
│                                                                  │
│ Policies (Inbound):                                             │
│  1. validate-jwt                                                │
│     - Verify JWT signature (Azure AD keys)                      │
│     - Verify issuer, audience, expiry                           │
│     - Extract claims (oid, appid)                               │
│                                                                  │
│  2. send-request (internal API call)                            │
│     - POST /internal/validate-worker                            │
│     - Verify: worker_oid in tenant.approved_workers             │
│                                                                  │
│  3. set-header                                                  │
│     - Inject X-Tenant-ID, X-Worker-OID                          │
│     - Keep Authorization header (for audit)                     │
│                                                                  │
│  4. rate-limit-by-key                                           │
│     - Max 100 req/min per tenant                                │
│                                                                  │
│ Forward to Backend Service (mTLS)                               │
└─────────────────────┬───────────────────────────────────────────┘
                      │ mTLS HTTPS
                      │ Client Cert: APIM identity
                      │ Private Network (Internal LB)
                      │ X-Tenant-ID: {uuid}
                      │ X-Worker-OID: {oid}
                      │ Authorization: Bearer {JWT}
                      │
┌─────────────────────▼───────────────────────────────────────────┐
│ Azure Internal Load Balancer (Private, VNET)                    │
│  - Terminal Point for mTLS (server cert)
│  - Routes traffic to AKS backend pool
│  - Health checks on pod:8080/health
└─────────────────────┬───────────────────────────────────────────┘
                      │ HTTPS (from LB frontend → Ingress)
                      │ Plain text (Ingress backend → Pod)
                      │ But Cilium encrypts at kernel level
                      │
┌─────────────────────▼───────────────────────────────────────────┐
│ AKS CLUSTER (Private Network, Cilium CNI)                       │
│                                                                  │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ Ingress (nginx) - Terminates LB HTTPS                       │ │
│ │  Routes to Service: api-service:8080                        │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                     │                                           │
│ ┌─────────────────▼─────────────────────────────────────────┐ │
│ │ az-rotator-api Namespace                                  │ │
│ │    [Cilium mTLS Boundary - Encrypts all pod traffic]      │ │
│ │                                                           │ │
│ │  ┌──────────────────────────────────────────────────┐   │ │
│ │  │ API Service Pod (Workload Identity)              │   │ │
│ │  │ Listens: http://localhost:8080 (plain HTTP)     │   │ │
│ │  │                                                  │   │ │
│ │  │ Middleware: Authentication                       │   │ │
│ │  │  1. Extract X-Tenant-ID, X-Worker-OID           │   │ │
│ │  │  2. Re-validate JWT (defense in depth)           │   │ │
│ │  │  3. Compare JWT claims vs headers (tampering)   │   │ │
│ │  │  4. Check worker status in DB                    │   │ │
│ │  │  5. Inject tenant context                        │   │ │
│ │  │                                                  │   │ │
│ │  │ Handler: RegisterWorkerHandler                   │   │ │
│ │  │  1. Create Worker record (status=PENDING)       │   │ │
│ │  │  2. Return registration_id                       │   │ │
│ │  │  3. Log audit event                              │   │ │
│ │  │                                                  │   │ │
│ │  └──────────────────────────────────────────────────┘   │ │
│ │             │                                            │ │
│ │             ├─[Cilium mTLS encrypted]→ PostgreSQL       │ │
│ │             │                                            │ │
│ │             └─[Cilium mTLS encrypted]→ Service Bus      │ │
│ │                                                           │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                               │
│ [All pod-to-pod traffic encrypted by Cilium at kernel level] │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary: JWT Validation & Request Flow

| Stage | Component | What Happens | Security Control |
|-------|-----------|--------------|------------------|
| 1 | **Worker MI** | Obtains JWT from Azure AD (client_credentials flow) | ✅ Azure AD issues signed JWT |
| 2 | **Worker** | Sends JWT to APIM (Authorization header) | ✅ HTTPS + JWT in header |
| 3 | **APIM** | Validates JWT signature, issuer, audience, expiry | ✅ Cryptographic validation |
| 4 | **APIM** | Validates worker belongs to tenant (API call) | ✅ Database lookup via internal API |
| 5 | **APIM** | Injects tenant context headers | ✅ Custom headers added by APIM |
| 6 | **APIM** | Forwards to API service (private network + mTLS optional) | ✅ Private connectivity |
| 7 | **API Service** | (Option A) Trusts APIM headers | ⚠️ Simplified trust model |
| 7 | **API Service** | (Option B) Re-validates JWT + verifies headers | ✅ Defense in depth |
| 8 | **API Service** | Creates Worker record in DB (status=PENDING_APPROVAL) | ✅ Audit logged |
| 9 | **API Service** | Returns registration_id to worker | ✅ Ready for admin approval |

---

## Recommendation: **Go with Option B (Defense in Depth)**
- Re-validate JWT at API service (small performance cost, big security gain)
- Keep original JWT in Authorization header for audit trail
- Verify claims match injected headers (anti-tampering)
- Log all auth events (worker OID, tenant ID, success/failure)
