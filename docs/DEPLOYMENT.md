# Deployment

End-to-end pipeline: build Go binaries → container images → Azure Container Registry → AKS.

---

## Container Images

### Multi-Stage Dockerfile (All Services)

Each service uses an identical Dockerfile pattern — only the `CMD` entry point differs.

```dockerfile
# ---- Build Stage ----
FROM golang:1.22-alpine AS build

RUN apk add --no-cache ca-certificates git

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download

COPY . .

ARG SERVICE_NAME
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -o /app ./cmd/${SERVICE_NAME}

# ---- Runtime Stage ----
FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=build /app /app

USER 65534:65534
ENTRYPOINT ["/app"]
```

**Key decisions:**
- `distroless/static` — no shell, no package manager; smallest attack surface
- `nonroot` tag — UID 65534, matches PSS `runAsNonRoot: true`
- `CGO_ENABLED=0` — pure Go binary, no libc dependency
- `-ldflags="-s -w"` — strip debug symbols (smaller image)

### Build Matrix

| Service | `SERVICE_NAME` arg | Image Tag Pattern |
|---------|-------------------|-------------------|
| API | `api` | `az-rotator-api:$(GIT_SHA)` |
| Scheduler | `scheduler` | `az-rotator-scheduler:$(GIT_SHA)` |
| System Tasks | `system-tasks` | `az-rotator-system-tasks:$(GIT_SHA)` |

**3 control plane images** deployed via Flux. System Tasks is a single binary with subcommands:

```bash
./app audit-cleanup     # run by CronJob (daily)
./app job-reconciliation # run by CronJob (every 10m)
./app retry-handler      # run by CronJob (every 5m)
```

The CronJobs all use the same image — only the `args` differ.

The **worker binary** is built separately and distributed as a standalone release (see [Worker Distribution](#worker-distribution-tenant-managed)).

---

## Azure Container Registry (ACR)

### Terraform Resource

```hcl
# deployment/terraform/acr.tf

resource "azurerm_container_registry" "main" {
  name                = "azrotator${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Premium"
  admin_enabled       = false

  # Private endpoint — no public access
  public_network_access_enabled = false

  # Geo-replication (prod only)
  dynamic "georeplications" {
    for_each = var.environment == "prod" ? [var.secondary_location] : []
    content {
      location = georeplications.value
    }
  }
}

# Private endpoint for AKS → ACR pull
resource "azurerm_private_endpoint" "acr" {
  name                = "pe-acr-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = azurerm_subnet.private_endpoints.id

  private_service_connection {
    name                           = "acr-connection"
    private_connection_resource_id = azurerm_container_registry.main.id
    subresource_names              = ["registry"]
    is_manual_connection           = false
  }
}

# AKS pull access (no admin credentials needed)
resource "azurerm_role_assignment" "aks_acr_pull" {
  scope                = azurerm_container_registry.main.id
  role_definition_name = "AcrPull"
  principal_id         = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
}
```

**Key points:**
- Premium SKU for private endpoints + geo-replication
- `admin_enabled = false` — no username/password auth ever
- AKS pulls via Managed Identity (`AcrPull` role on kubelet identity)
- Private endpoint means images never traverse public internet

---

## CI/CD Pipeline (GitHub Actions + Flux)

GitOps model with **environment branches**:
1. PR opens → CI builds images and deploys to **dev cluster** for testing
2. PR merges to main → CI promotes images to **staging cluster**
3. Manual merge promotes to **prod cluster**

CI never touches any cluster directly. It only pushes images and commits tags to deploy branches. Flux (running in each cluster) watches its own branch and reconciles.

### Branch Model

```
main               ← code lives here (PRs merge here after testing)
deploy/dev         ← Flux (dev cluster) watches this branch
deploy/staging     ← Flux (staging cluster) watches this branch
deploy/prod        ← Flux (prod cluster) watches this branch
```

### Build & Deploy Workflow

```yaml
# .github/workflows/build-push.yaml

name: Build & Deploy

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
    paths:
      - 'cmd/**'
      - 'internal/**'
      - 'go.mod'
      - 'go.sum'
      - 'Dockerfile'

env:
  ACR_NAME: azrotatorprod
  REGISTRY: azrotatorprod.azurecr.io

permissions:
  id-token: write   # OIDC federation
  contents: write   # push tag commits to deploy branches

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service:
          - name: api
            path: api
          - name: scheduler
            path: scheduler
          - name: system-tasks
            path: system-tasks

    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: ACR Login
        run: az acr login --name ${{ env.ACR_NAME }}

      - name: Determine Image Tag
        id: tag
        run: |
          if [ "${{ github.event_name }}" == "pull_request" ]; then
            echo "tag=pr-${{ github.event.number }}" >> $GITHUB_OUTPUT
          else
            echo "tag=${{ github.sha }}" >> $GITHUB_OUTPUT
          fi

      - name: Build & Push
        run: |
          IMAGE="${{ env.REGISTRY }}/az-rotator-${{ matrix.service.name }}:${{ steps.tag.outputs.tag }}"
          docker build \
            --build-arg SERVICE_NAME=${{ matrix.service.path }} \
            --tag "$IMAGE" \
            .
          docker push "$IMAGE"

      - name: Scan Image (Trivy)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "${{ env.REGISTRY }}/az-rotator-${{ matrix.service.name }}:${{ steps.tag.outputs.tag }}"
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH

      - name: Upload Scan Results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif

  # ─── PR: Deploy to dev cluster for testing ───────────────────────────
  deploy-dev:
    needs: build
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: deploy/dev
          token: ${{ secrets.DEPLOY_TOKEN }}

      - name: Update Dev Image Tags
        run: |
          TAG="pr-${{ github.event.number }}"
          cd deployment/kubernetes/overlays/dev
          kustomize edit set image \
            azrotatorprod.azurecr.io/az-rotator-api=azrotatorprod.azurecr.io/az-rotator-api:${TAG} \
            azrotatorprod.azurecr.io/az-rotator-scheduler=azrotatorprod.azurecr.io/az-rotator-scheduler:${TAG} \
            azrotatorprod.azurecr.io/az-rotator-system-tasks=azrotatorprod.azurecr.io/az-rotator-system-tasks:${TAG}

      - name: Commit & Push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          git commit -m "dev: deploy PR #${{ github.event.number }}" || exit 0
          git push origin deploy/dev

  # ─── Merge to main: Auto-promote to staging ─────────────────────────
  deploy-staging:
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: deploy/staging
          token: ${{ secrets.DEPLOY_TOKEN }}

      - name: Update Staging Image Tags
        run: |
          cd deployment/kubernetes/overlays/staging
          kustomize edit set image \
            azrotatorprod.azurecr.io/az-rotator-api=azrotatorprod.azurecr.io/az-rotator-api:${{ github.sha }} \
            azrotatorprod.azurecr.io/az-rotator-scheduler=azrotatorprod.azurecr.io/az-rotator-scheduler:${{ github.sha }} \
            azrotatorprod.azurecr.io/az-rotator-system-tasks=azrotatorprod.azurecr.io/az-rotator-system-tasks:${{ github.sha }}

      - name: Commit & Push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          git commit -m "staging: deploy ${{ github.sha }}" || exit 0
          git push origin deploy/staging
```

### Promote to Production (Manual)

Production promotion is a deliberate action — merge `deploy/staging` → `deploy/prod`:

```bash
# After verifying staging
git checkout deploy/prod
git merge deploy/staging
git push origin deploy/prod
```

Or create a PR from `deploy/staging` → `deploy/prod` for an audit trail with approval.

**Pipeline features:**
- **OIDC federation** — no stored credentials; GitHub Actions federated identity to Azure AD
- **Matrix build** — all 3 images built in parallel
- **Trivy scanning** — vulnerabilities flagged before deployment
- **PR builds deploy to dev** — test your code in a real cluster before merging
- **Merge to main auto-promotes to staging** — no manual step needed
- **Prod promotion is manual** — deliberate merge from staging branch
- **No kubectl, no cluster credentials** — CI only pushes images and commits tags
- **Each cluster pulls its own state** — Flux watches its own deploy branch

---

## Flux GitOps Controller

Each AKS cluster runs its own Flux installation, watching its corresponding deploy branch. When CI commits a tag update (or you merge between branches), Flux detects the change and applies it.

### Flux Bootstrap (Per Cluster)

```bash
# Dev cluster
flux bootstrap github \
  --owner=your-org \
  --repository=go-az-secrets-rotator \
  --branch=deploy/dev \
  --path=deployment/kubernetes/overlays/dev \
  --personal=false

# Staging cluster
flux bootstrap github \
  --owner=your-org \
  --repository=go-az-secrets-rotator \
  --branch=deploy/staging \
  --path=deployment/kubernetes/overlays/staging \
  --personal=false

# Prod cluster
flux bootstrap github \
  --owner=your-org \
  --repository=go-az-secrets-rotator \
  --branch=deploy/prod \
  --path=deployment/kubernetes/overlays/prod \
  --personal=false
```

### Flux Source (GitRepository)

Each cluster points to its own branch:

```yaml
# deployment/flux/gitrepository.yaml (prod example)

apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: az-rotator
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/your-org/go-az-secrets-rotator
  ref:
    branch: deploy/prod      # ← each cluster watches its own branch
  secretRef:
    name: github-deploy-key
```

### Flux Kustomization (Reconciler)

```yaml
# deployment/flux/kustomization.yaml (prod example)

apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: az-rotator-prod
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: az-rotator
  path: ./deployment/kubernetes/overlays/prod
  prune: true              # delete resources removed from git
  wait: true               # wait for health before marking ready
  timeout: 3m
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: api
      namespace: az-rotator-api
    - apiVersion: apps/v1
      kind: Deployment
      name: scheduler
      namespace: az-rotator-scheduler
```

**What Flux gives you:**
- **Drift detection** — if someone `kubectl edit`s a resource manually, Flux reverts it within 5 min
- **Health gating** — Flux won't mark a release successful until Deployments are healthy
- **Prune** — delete a manifest from git → Flux deletes the resource from cluster
- **No cluster creds in CI** — Flux pulls from git; CI never talks to the cluster
- **Branch isolation** — changes only reach a cluster when merged to its deploy branch

---

## Kustomize Manifest Structure

```
deployment/kubernetes/
├── base/
│   ├── kustomization.yaml
│   ├── namespaces.yaml
│   ├── api-deployment.yaml
│   ├── scheduler-deployment.yaml
│   ├── cronjobs.yaml
│   ├── workload-identity.yaml
│   ├── external-secrets.yaml
│   ├── network-policies.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml      # patches: replica count, resource limits, DB host
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml      # ← Flux watches this path
```

### Base Kustomization

```yaml
# deployment/kubernetes/base/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespaces.yaml
  - api-deployment.yaml
  - scheduler-deployment.yaml
  - cronjobs.yaml
  - workload-identity.yaml
  - external-secrets.yaml
  - network-policies.yaml
  - configmap.yaml
```

### Prod Overlay

```yaml
# deployment/kubernetes/overlays/prod/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
images:
  - name: azrotatorprod.azurecr.io/az-rotator-api
    newTag: "abc123"    # ← updated by CI on each build
  - name: azrotatorprod.azurecr.io/az-rotator-scheduler
    newTag: "abc123"
  - name: azrotatorprod.azurecr.io/az-rotator-system-tasks
    newTag: "abc123"
patchesStrategicMerge:
  - configmap-patch.yaml
```

---

## Kubernetes Manifests (Base)

### Namespace Setup

```yaml
# deployment/kubernetes/base/namespaces.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: az-rotator-api
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
---
apiVersion: v1
kind: Namespace
metadata:
  name: az-rotator-scheduler
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
---
apiVersion: v1
kind: Namespace
metadata:
  name: az-rotator-system
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```

### API Service Deployment

```yaml
# deployment/kubernetes/api-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: az-rotator-api
  labels:
    app: az-rotator-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: az-rotator-api
  template:
    metadata:
      labels:
        app: az-rotator-api
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: az-rotator-api-sa
      automountServiceAccountToken: false
      containers:
        - name: api
          image: azrotatorprod.azurecr.io/az-rotator-api:${IMAGE_TAG}
          ports:
            - containerPort: 8080
              protocol: TCP
          env:
            - name: PORT
              value: "8080"
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: az-rotator-config
                  key: db-host
            - name: DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: az-rotator-config
                  key: db-name
            - name: SERVICE_BUS_NAMESPACE
              valueFrom:
                configMapKeyRef:
                  name: az-rotator-config
                  key: service-bus-namespace
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector.monitoring:4317"
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          securityContext:
            runAsNonRoot: true
            runAsUser: 65534
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: az-rotator-api
spec:
  selector:
    app: az-rotator-api
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  type: ClusterIP
```

### Scheduler Deployment

```yaml
# deployment/kubernetes/scheduler-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: scheduler
  namespace: az-rotator-scheduler
  labels:
    app: az-rotator-scheduler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: az-rotator-scheduler
  template:
    metadata:
      labels:
        app: az-rotator-scheduler
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: az-rotator-scheduler-sa
      automountServiceAccountToken: false
      containers:
        - name: scheduler
          image: azrotatorprod.azurecr.io/az-rotator-scheduler:${IMAGE_TAG}
          env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: az-rotator-config
                  key: db-host
            - name: DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: az-rotator-config
                  key: db-name
            - name: SERVICE_BUS_NAMESPACE
              valueFrom:
                configMapKeyRef:
                  name: az-rotator-config
                  key: service-bus-namespace
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector.monitoring:4317"
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          securityContext:
            runAsNonRoot: true
            runAsUser: 65534
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8081
            initialDelaySeconds: 5
            periodSeconds: 10
```

### System Tasks CronJobs

All three CronJobs use the **same image** — the subcommand is passed via `args`.

```yaml
# deployment/kubernetes/cronjobs.yaml

apiVersion: batch/v1
kind: CronJob
metadata:
  name: audit-cleanup
  namespace: az-rotator-system
spec:
  schedule: "0 3 * * *"    # Daily at 03:00 UTC
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 300
      template:
        metadata:
          labels:
            azure.workload.identity/use: "true"
        spec:
          serviceAccountName: az-rotator-system-sa
          restartPolicy: OnFailure
          containers:
            - name: system-tasks
              image: azrotatorprod.azurecr.io/az-rotator-system-tasks:${IMAGE_TAG}
              args: ["audit-cleanup"]
              env:
                - name: DB_HOST
                  valueFrom:
                    configMapKeyRef:
                      name: az-rotator-config
                      key: db-host
                - name: DB_NAME
                  valueFrom:
                    configMapKeyRef:
                      name: az-rotator-config
                      key: db-name
                - name: RETENTION_DAYS
                  value: "90"
              resources:
                requests:
                  cpu: 50m
                  memory: 64Mi
                limits:
                  cpu: 200m
                  memory: 128Mi
              securityContext:
                runAsNonRoot: true
                runAsUser: 65534
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                capabilities:
                  drop: ["ALL"]
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: job-reconciliation
  namespace: az-rotator-system
spec:
  schedule: "*/10 * * * *"   # Every 10 minutes
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 120
      template:
        metadata:
          labels:
            azure.workload.identity/use: "true"
        spec:
          serviceAccountName: az-rotator-system-sa
          restartPolicy: OnFailure
          containers:
            - name: system-tasks
              image: azrotatorprod.azurecr.io/az-rotator-system-tasks:${IMAGE_TAG}
              args: ["job-reconciliation"]
              env:
                - name: DB_HOST
                  valueFrom:
                    configMapKeyRef:
                      name: az-rotator-config
                      key: db-host
                - name: DB_NAME
                  valueFrom:
                    configMapKeyRef:
                      name: az-rotator-config
                      key: db-name
                - name: SERVICE_BUS_NAMESPACE
                  valueFrom:
                    configMapKeyRef:
                      name: az-rotator-config
                      key: service-bus-namespace
              resources:
                requests:
                  cpu: 50m
                  memory: 64Mi
                limits:
                  cpu: 200m
                  memory: 128Mi
              securityContext:
                runAsNonRoot: true
                runAsUser: 65534
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                capabilities:
                  drop: ["ALL"]
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: retry-handler
  namespace: az-rotator-system
spec:
  schedule: "*/5 * * * *"   # Every 5 minutes
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 120
      template:
        metadata:
          labels:
            azure.workload.identity/use: "true"
        spec:
          serviceAccountName: az-rotator-system-sa
          restartPolicy: OnFailure
          containers:
            - name: system-tasks
              image: azrotatorprod.azurecr.io/az-rotator-system-tasks:${IMAGE_TAG}
              args: ["retry-handler"]
              env:
                - name: DB_HOST
                  valueFrom:
                    configMapKeyRef:
                      name: az-rotator-config
                      key: db-host
                - name: DB_NAME
                  valueFrom:
                    configMapKeyRef:
                      name: az-rotator-config
                      key: db-name
                - name: SERVICE_BUS_NAMESPACE
                  valueFrom:
                    configMapKeyRef:
                      name: az-rotator-config
                      key: service-bus-namespace
              resources:
                requests:
                  cpu: 50m
                  memory: 64Mi
                limits:
                  cpu: 200m
                  memory: 128Mi
              securityContext:
                runAsNonRoot: true
                runAsUser: 65534
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                capabilities:
                  drop: ["ALL"]
```

---

## Workload Identity Binding

```yaml
# deployment/kubernetes/workload-identity.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: az-rotator-api-sa
  namespace: az-rotator-api
  annotations:
    azure.workload.identity/client-id: "${API_CLIENT_ID}"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: az-rotator-scheduler-sa
  namespace: az-rotator-scheduler
  annotations:
    azure.workload.identity/client-id: "${SCHEDULER_CLIENT_ID}"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: az-rotator-system-sa
  namespace: az-rotator-system
  annotations:
    azure.workload.identity/client-id: "${SYSTEM_CLIENT_ID}"
```

Each service gets its own User-Assigned Managed Identity with minimum RBAC:

| Service Account | Azure Roles |
|----------------|-------------|
| `az-rotator-api-sa` | Key Vault Secrets User, Service Bus Data Sender, PostgreSQL Flexible Server AAD Admin |
| `az-rotator-scheduler-sa` | Service Bus Data Sender, PostgreSQL Flexible Server AAD Admin |
| `az-rotator-system-sa` | Service Bus Data Sender, PostgreSQL Flexible Server AAD Admin |

All three authenticate to PostgreSQL via **Azure AD token** (Workload Identity → `azidentity.DefaultAzureCredential` → short-lived access token as password). No static DB credentials exist.

---

## External Secrets Operator (ESO)

```yaml
# deployment/kubernetes/external-secrets.yaml

apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: azure-keyvault
spec:
  provider:
    azurekv:
      authType: WorkloadIdentity
      vaultUrl: "https://kv-azrotator-${ENVIRONMENT}.vault.azure.net"
      serviceAccountRef:
        name: az-rotator-api-sa
        namespace: az-rotator-api
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: az-rotator-secrets
  namespace: az-rotator-api
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: azure-keyvault
    kind: ClusterSecretStore
  target:
    name: az-rotator-secrets
    creationPolicy: Owner
  data:
    - secretKey: signing-key
      remoteRef:
        key: job-signing-key
```

ESO syncs non-database secrets from Key Vault → Kubernetes Secrets every hour.

> **No DB password in Key Vault.** PostgreSQL auth uses Azure AD tokens acquired at runtime via Workload Identity. The Go services call `azidentity.DefaultAzureCredential` to get a short-lived token, which is passed as the PostgreSQL password. Token refresh is handled automatically by the Azure SDK.

---

## Cilium Network Policies

```yaml
# deployment/kubernetes/network-policies.yaml

apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-policy
  namespace: az-rotator-api
spec:
  endpointSelector:
    matchLabels:
      app: az-rotator-api
  ingress:
    - fromEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: apim-ingress
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
  egress:
    - toFQDNs:
        - matchName: "*.postgres.database.azure.com"
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
    - toFQDNs:
        - matchName: "*.servicebus.windows.net"
      toPorts:
        - ports:
            - port: "5671"
              protocol: TCP
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: monitoring
      toPorts:
        - ports:
            - port: "4317"
              protocol: TCP
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: scheduler-policy
  namespace: az-rotator-scheduler
spec:
  endpointSelector:
    matchLabels:
      app: az-rotator-scheduler
  ingress: []   # No inbound traffic
  egress:
    - toFQDNs:
        - matchName: "*.postgres.database.azure.com"
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
    - toFQDNs:
        - matchName: "*.servicebus.windows.net"
      toPorts:
        - ports:
            - port: "5671"
              protocol: TCP
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: monitoring
      toPorts:
        - ports:
            - port: "4317"
              protocol: TCP
```

**Scheduler has zero ingress** — it only polls outward (DB read, Service Bus publish, OTel export).

---

## Terraform Infrastructure

### Module Layout

```
deployment/terraform/
├── main.tf              # Provider config, backend (Azure Storage)
├── variables.tf         # Input variables
├── outputs.tf           # Cluster endpoint, ACR URL, etc.
├── vnet.tf              # VNET + subnets + private endpoints
├── aks.tf               # AKS cluster (Cilium, Workload Identity)
├── acr.tf               # Container Registry + private endpoint
├── apim.tf              # API Management + JWT policy
├── service-bus.tf       # Namespace + topics + RBAC
├── postgresql.tf        # Flexible Server + private endpoint
├── keyvault.tf          # Key Vault + access policies
├── log-analytics.tf     # Workspace + diagnostic settings
├── monitoring.tf        # Managed Grafana + App Insights
└── environments/
    ├── dev.tfvars
    ├── staging.tfvars
    └── prod.tfvars
```

### AKS Cluster

```hcl
# deployment/terraform/aks.tf

resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-azrotator-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "azrotator-${var.environment}"

  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D2s_v5"
    node_count          = 2
    vnet_subnet_id      = azurerm_subnet.aks.id
    os_disk_size_gb     = 30
    temporary_name_for_rotation = "tempsys"
  }

  identity {
    type = "SystemAssigned"
  }

  oidc_issuer_enabled       = true
  workload_identity_enabled = true

  network_profile {
    network_plugin    = "none"   # Cilium (installed via Helm)
    network_policy    = "none"
    service_cidr      = "10.1.0.0/16"
    dns_service_ip    = "10.1.0.10"
  }

  azure_active_directory_role_based_access_control {
    azure_rbac_enabled = true
    managed            = true
  }

  key_vault_secrets_provider {
    secret_rotation_enabled  = true
    secret_rotation_interval = "2m"
  }

  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  }

  tags = var.tags
}
```

### Environment Strategy

| Environment | AKS Nodes | ACR SKU | PostgreSQL SKU | Purpose |
|-------------|-----------|---------|----------------|---------|
| **dev** | 2 × D2s_v5 | Basic | Burstable B1ms | Local development, feature testing |
| **staging** | 2 × D2s_v5 | Standard | GP Standard_D2s_v3 | Integration tests, pre-prod validation |
| **prod** | 3 × D4s_v5 | Premium | GP Standard_D4s_v3 | Production workloads, geo-replicated ACR |

Deploy per environment:

```bash
cd deployment/terraform
terraform workspace select prod
terraform apply -var-file="environments/prod.tfvars"
```

---

## Deployment Flow (End to End)

```
┌───────────────────────────────────────────────────────────────────────┐
│  1. Developer opens PR (feature branch → main)                        │
└──────────────────┬────────────────────────────────────────────────────┘
                   │
                   ▼
┌───────────────────────────────────────────────────────────────────────┐
│  CI: Build & Deploy to Dev                                            │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ 1. Build 3 images (tagged :pr-42)                              │   │
│  │ 2. Push to ACR                                                 │   │
│  │ 3. Trivy scan                                                  │   │
│  │ 4. Commit tag update to deploy/dev branch                      │   │
│  └────────────────────────────────────────────────────────────────┘   │
└──────────────────┬────────────────────────────────────────────────────┘
                   │
                   ▼
┌───────────────────────────────────────────────────────────────────────┐
│  Flux (dev cluster) picks up deploy/dev commit → deploys PR code      │
│  Developer tests in real environment                                  │
└──────────────────┬────────────────────────────────────────────────────┘
                   │ tests pass, PR approved
                   ▼
┌───────────────────────────────────────────────────────────────────────┐
│  2. PR merged to main                                                 │
└──────────────────┬────────────────────────────────────────────────────┘
                   │
                   ▼
┌───────────────────────────────────────────────────────────────────────┐
│  CI: Build & Promote to Staging                                       │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ 1. Build 3 images (tagged :git-sha)                            │   │
│  │ 2. Push to ACR                                                 │   │
│  │ 3. Trivy scan                                                  │   │
│  │ 4. Commit tag update to deploy/staging branch                  │   │
│  └────────────────────────────────────────────────────────────────┘   │
└──────────────────┬────────────────────────────────────────────────────┘
                   │
                   ▼
┌───────────────────────────────────────────────────────────────────────┐
│  Flux (staging cluster) picks up deploy/staging commit → deploys      │
│  Integration tests / final validation                                 │
└──────────────────┬────────────────────────────────────────────────────┘
                   │ verified in staging
                   ▼
┌───────────────────────────────────────────────────────────────────────┐
│  3. Manual: merge deploy/staging → deploy/prod (PR with approval)     │
└──────────────────┬────────────────────────────────────────────────────┘
                   │
                   ▼
┌───────────────────────────────────────────────────────────────────────┐
│  Flux (prod cluster) picks up deploy/prod commit → deploys            │
│  ┌─────────────┐  ┌─────────────────┐  ┌──────────────────────────┐  │
│  │ az-rotator- │  │ az-rotator-     │  │ az-rotator-system        │  │
│  │ api         │  │ scheduler       │  │ (CronJobs × 3)           │  │
│  │ Deployment  │  │ Deployment      │  │                          │  │
│  └─────────────┘  └─────────────────┘  └──────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
```

**Key points:**
- Code is tested in a real cluster (dev) **before** merging to main
- CI never has cluster credentials — each cluster pulls its own state from its branch
- Promotion between environments = merging between deploy branches
- Every promotion is a git commit — fully auditable, fully reversible

---

## Rollback Strategy

### Automatic (Failed Rollout)

Kubernetes deployment strategy:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  minReadySeconds: 10
  progressDeadlineSeconds: 120
```

If the new pod fails readiness probe within 120s, the rollout stalls. The old pod remains serving. Flux will report the Kustomization as `not ready`.

### Rollback via Git (Primary Method)

Since Flux reconciles from git, rollback = revert the commit:

```bash
# Revert the image tag update commit
git revert HEAD
git push

# Flux detects the revert within 1 minute
# Re-applies the previous image tags → old pods roll out
```

This is fully auditable — the rollback appears in git history.

### Rollback to Specific Version

```bash
# Manually set the image tag to a known-good SHA
cd deployment/kubernetes/overlays/prod
kustomize edit set image \
  azrotatorprod.azurecr.io/az-rotator-api=azrotatorprod.azurecr.io/az-rotator-api:known-good-sha \
  azrotatorprod.azurecr.io/az-rotator-scheduler=azrotatorprod.azurecr.io/az-rotator-scheduler:known-good-sha \
  azrotatorprod.azurecr.io/az-rotator-system-tasks=azrotatorprod.azurecr.io/az-rotator-system-tasks:known-good-sha

git add . && git commit -m "rollback: revert to known-good-sha" && git push
```

### Emergency (Suspend Flux)

If you need to stop Flux from reconciling while you debug:

```bash
# Pause reconciliation
flux suspend kustomization az-rotator-prod

# Manual kubectl fix if needed
kubectl rollout undo deployment/api -n az-rotator-api

# Resume when ready
flux resume kustomization az-rotator-prod
```

---

## Health Checks

| Service | Liveness | Readiness | Notes |
|---------|----------|-----------|-------|
| API | `GET /healthz` (process alive) | `GET /readyz` (DB + Service Bus connected) | Readiness gates traffic from APIM |
| Scheduler | `GET /healthz` (process alive) | — | No ingress, no readiness needed |
| CronJobs | — | — | Success = exit 0; failure = backoffLimit retries |

---

## Configuration Management

### ConfigMap (Non-Sensitive)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: az-rotator-config
  namespace: az-rotator-api
data:
  db-host: "pg-azrotator-prod.postgres.database.azure.com"
  db-name: "azrotator"
  service-bus-namespace: "sb-azrotator-prod.servicebus.windows.net"
  log-level: "info"
  otel-service-name: "az-rotator-api"
```

### Secrets (ESO-Managed)

Non-database secrets live in Azure Key Vault and are synced by ESO:
- `job-signing-key` — HMAC key for signed job payloads

### Database Auth (Azure AD — No Secrets)

PostgreSQL authentication uses **Azure AD tokens**, not passwords:
1. Pod starts with Workload Identity (federated service account)
2. Go service calls `azidentity.DefaultAzureCredential` → gets short-lived Azure AD token
3. Token is used as the PostgreSQL password (`host=... user=<client-id> password=<token> sslmode=require`)
4. Token auto-refreshes (~1h lifetime, SDK handles renewal)

No DB password exists anywhere — not in Key Vault, not in environment variables, not in ConfigMaps.

---

## Local Development

### Build Locally

```bash
# Build a single service
docker build --build-arg SERVICE_NAME=api -t az-rotator-api:local .

# Run locally (with .env file for config)
docker run --env-file .env.local -p 8080:8080 az-rotator-api:local
```

### Skaffold (Optional — Hot Reload)

```yaml
# skaffold.yaml
apiVersion: skaffold/v4beta6
kind: Config
build:
  artifacts:
    - image: az-rotator-api
      docker:
        dockerfile: Dockerfile
        buildArgs:
          SERVICE_NAME: api
deploy:
  kubectl:
    manifests:
      - deployment/kubernetes/api-deployment.yaml
```

```bash
skaffold dev --port-forward
```

---

## Branch Strategy & CI Workflow Triggers

### Branch Purposes

| Branch | Contains | Who writes to it | Cluster |
|--------|----------|------------------|---------|
| `main` | Go code, Terraform, base manifests, tests | Developers (via PR) | None directly |
| `deploy/dev` | Kustomize overlay + image tags for dev | CI bot (on PR) | Dev cluster |
| `deploy/staging` | Kustomize overlay + image tags for staging | CI bot (on merge to main) | Staging cluster |
| `deploy/prod` | Kustomize overlay + image tags for prod | Manual merge from staging | Prod cluster |

### Preventing Infinite Loops

The build workflow only triggers on **code path** changes to `main`. Tag commits go to deploy branches (not main), so they never re-trigger CI:

```yaml
on:
  pull_request:
    branches: [main]           # build on every PR
  push:
    branches: [main]
    paths:                     # only re-build on code changes to main
      - 'cmd/**'
      - 'internal/**'
      - 'go.mod'
      - 'go.sum'
      - 'Dockerfile'
```

CI commits go to `deploy/dev` or `deploy/staging` — never to main — so no loop is possible.

### Workflow Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│  PR opened (feature branch → main)                                       │
│    → CI builds :pr-42 images → commits tag to deploy/dev                 │
│    → Flux (dev cluster) deploys → developer tests                        │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │ PR approved + merged to main
                               ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  Push to main                                                            │
│    → CI builds :sha images → commits tag to deploy/staging               │
│    → Flux (staging cluster) deploys → integration tests                  │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │ manual merge deploy/staging → deploy/prod
                               ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  Flux (prod cluster) deploys                                             │
└──────────────────────────────────────────────────────────────────────────┘
```

### Pipeline Details

#### 1. PR Opened → Deploy to Dev (test before merge)

```
Developer → feature branch → opens PR
  → CI builds 3 images (:pr-42), pushes to ACR, Trivy scans
  → CI commits tag to deploy/dev branch
  → Flux (dev cluster) picks up within 1 min → deploys
  → Developer tests in real environment
  → Satisfied → requests review → PR merged to main
```

**Code is always tested in a real cluster before it reaches main.**

#### 2. PR Merged to Main → Auto-Promote to Staging

```
PR merged to main
  → CI builds 3 images (:sha), pushes to ACR, Trivy scans
  → CI commits tag to deploy/staging branch
  → Flux (staging cluster) picks up within 1 min → deploys
  → Integration tests / validation
```

Automatic — no human action needed after merge.

#### 3. Staging Verified → Manual Promote to Prod

```
Team verifies staging is healthy
  → Merge deploy/staging → deploy/prod (PR with approval or direct merge)
  → Flux (prod cluster) picks up within 1 min → deploys
```

Deliberate. Prod never gets code that hasn't been tested in staging.

#### 4. Terraform Changes (separate pipeline, manual approval)

```
Developer → feature branch → PR (Terraform changes) → merge to main
  → CI runs `terraform plan` (shows what will change)
  → Requires manual approval (GitHub environment protection)
  → CI runs `terraform apply`
```

```yaml
# .github/workflows/terraform.yaml

name: Terraform

on:
  push:
    branches: [main]
    paths:
      - 'deployment/terraform/**'

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Terraform Plan
        run: |
          cd deployment/terraform
          terraform init
          terraform plan -var-file="environments/prod.tfvars" -out=plan.tfplan
      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: deployment/terraform/plan.tfplan

  apply:
    needs: plan
    runs-on: ubuntu-latest
    environment: production    # ← requires manual approval click
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: tfplan
          path: deployment/terraform/
      - name: Terraform Apply
        run: |
          cd deployment/terraform
          terraform init
          terraform apply plan.tfplan
```

Infra changes are riskier — that's why apply needs a human to click "approve."

#### 5. K8s Manifest / Base Changes (tested via promotion)

If you change `deployment/kubernetes/base/` (e.g., add an env var, new CronJob):

```
Developer → feature branch → PR (includes base/ change)
  → CI builds images → commits to deploy/dev (base change is in that branch too)
  → Dev cluster gets the new manifest + new code
  → Test → merge to main → staging gets it → promote to prod
```

Base changes are **tested in dev first** because they live in the same branch. They don't leak to prod until you promote.

---

## Worker Distribution (Tenant-Managed)

The worker is **not** deployed via Flux or the control plane CI pipeline. It's a standalone Go binary published as a GitHub Release. Tenant teams download it and deploy it in their own environment (Container App, VM, etc.).

### Why separate

| | Control Plane | Worker |
|---|---|---|
| **Runs on** | AKS (your cluster) | Tenant's Container App / VM |
| **Deployed by** | Flux (deploy branches) | Tenant team (their own IaC) |
| **Instances** | 1 per service | 1 per tenant |
| **Managed by** | Platform team | Tenant team |

You ship the binary. They run it. Clean separation.

### Release Pipeline

```yaml
# .github/workflows/release-worker.yaml

name: Release Worker Binary

on:
  push:
    tags:
      - 'worker/v*'    # triggered by: git tag worker/v1.3.0 && git push --tags

permissions:
  contents: write      # create GitHub Release

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Build Binary
        run: |
          CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
            go build -ldflags="-s -w" -o az-rotator-worker ./cmd/worker

      - name: Create Release
        uses: softprops/action-gh-release@v2
        with:
          files: az-rotator-worker
          body: |
            ## Worker Binary ${{ github.ref_name }}

            Download and run:
            ```bash
            chmod +x az-rotator-worker
            export TENANT_ID="your-tenant-uuid"
            export CONTROL_PLANE_URL="https://api.azrotator.yourdomain.com"
            ./az-rotator-worker
            ```

            Required environment variables:
            - `TENANT_ID` — your tenant UUID (from registration)
            - `CONTROL_PLANE_URL` — platform API endpoint
            - `AZURE_CLIENT_ID` — Managed Identity client ID (for auth + resource access)
```

Triggered by tagging a release:

```bash
git tag worker/v1.3.0
git push --tags
# → CI builds binary → attaches to GitHub Release
```

### Versioning

| Tag | Example | Use |
|-----|---------|-----|
| Semver | `worker/v1.3.0` | Stable release, tenant pins to this |
| Pre-release | `worker/v1.4.0-rc.1` | Testing before stable |

Tenant teams upgrade on their own schedule. They pick a version, test it, and roll it out.

### What the tenant deploys

The tenant downloads the binary and wraps it in a minimal container:

```dockerfile
# Tenant's Dockerfile (2 lines)
FROM gcr.io/distroless/static-debian12:nonroot
COPY az-rotator-worker /app
USER 65534:65534
ENTRYPOINT ["/app"]
```

### Worker startup (automatic)

Once deployed, the binary handles everything:

```
1. Worker starts
2. Gets Azure AD token via Managed Identity
3. Calls control plane: POST /api/workers/register
4. Waits for admin approval (polls or webhook)
5. Once approved → subscribes to Service Bus → processes rotation jobs
```

The tenant team doesn't configure rotation logic — they just deploy and register. The platform handles the rest.

### What you provide to tenant teams

| Artifact | Where | Purpose |
|----------|-------|---------|
| Binary | GitHub Release (`worker/v1.x.x`) | The compiled worker |
| Docs | ONBOARDING.md | How to deploy, required env vars, registration flow |
| Example Terraform | `examples/tenant-deployment/` | Reference Container App config |
| Example Dockerfile | `examples/tenant-deployment/Dockerfile` | 2-line wrapper |

---

## Summary

| Concern | Solution |
|---------|----------|
| Image build | Multi-stage Dockerfile, distroless runtime, nonroot user |
| Binaries | 3 control plane images + 1 worker binary (separate release) |
| Registry | ACR Premium, private endpoint, AKS pulls via Managed Identity |
| CI/CD | GitHub Actions builds & pushes images, commits tags to deploy branches |
| GitOps | Flux per cluster watches its own deploy branch |
| Testing | PR → deploy to dev cluster → test → then merge to main |
| Promotion | dev → staging (auto on merge) → prod (manual merge) |
| Worker | Standalone binary, GitHub Release, tenant-managed deployment |
| Manifests | Kustomize base + per-environment overlays |
| DB Auth | Azure AD tokens via Workload Identity (no passwords) |
| Secrets | Key Vault → ESO for non-DB secrets (signing keys) |
| Network | Cilium policies, private endpoints, zero public exposure |
| Rollback | `git revert` on deploy branch → Flux restores previous state |
| Drift | Flux auto-corrects manual cluster changes within 5 min |
| Environments | Terraform workspaces + deploy branches + Kustomize overlays |
