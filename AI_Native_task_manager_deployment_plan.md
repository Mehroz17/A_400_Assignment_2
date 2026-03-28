# AI Native Task Manager — Deployment Plan

---

## Scenario Overview

An AI-powered task management system with 4 services:

| Service | Role |
|---|---|
| **UI Interface** | Frontend web UI; connects to Backend API and Todo Agent |
| **Backend API** | Core business logic + task/session storage (SQLite) |
| **Todo Agent** | AI agent (OpenAI SDK / Gemini API); manages tasks via Backend API |
| **Notification API** | Sends notifications; connects to UI Interface and Backend API |

**Tech Stack:** FastAPI · Python 3.12 · OpenAI SDK · Gemini API · SQLite

---

## Deployment Order (Chronological)

```
Step 1  →  Dockerfiles           (build & push container images)
Step 2  →  Namespace             (create isolated environment in cluster)
Step 3  →  RBAC                  (identities & permissions before anything runs)
Step 4  →  ConfigMaps            (non-sensitive config must exist before pods start)
Step 5  →  Secrets               (sensitive config must exist before pods start)
Step 6  →  PersistentVolumeClaim (storage must exist before Backend API pod starts)
Step 7  →  Deployments           (start the pods — all dependencies now ready)
Step 8  →  Services              (expose pods for internal & external traffic)
Step 9  →  HPA                   (autoscaler — requires deployments to exist first)
Step 10 →  PodDisruptionBudget   (availability guarantee — requires running pods)
```

> Each step depends on the previous one. Applying out of order causes pods to crash on startup
> (missing config/secrets) or fail scheduling (missing ServiceAccount).

---

## Step 1: Dockerfiles — Build & Push Container Images

**Why first:** Everything in Kubernetes runs from a container image. Images must be built and
pushed to a registry before any Kubernetes manifest can reference them. If the image doesn't
exist, the pod will stay in `ImagePullBackOff` state forever.

---

### 1.1 Project Directory Structure (before writing Dockerfiles)

```
.
├── ui-interface/
│   ├── main.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .dockerignore
├── backend-api/
│   ├── main.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .dockerignore
├── todo-agent/
│   ├── main.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .dockerignore
└── notification-api/
    ├── main.py
    ├── requirements.txt
    ├── Dockerfile
    └── .dockerignore
```

---

### 1.2 Dockerfile Architecture — 2-Stage Multi-Stage Build

All 4 services follow the same pattern. 2 stages keep the final image lean — build tools
are never shipped to production.

```dockerfile
# ─── Stage 1: Builder ───────────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /app

# Copy requirements first — Docker caches this layer
# Only re-runs pip install if requirements.txt changes
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ─── Stage 2: Runtime ───────────────────────────────────────────────
FROM python:3.12-slim AS runtime

WORKDIR /app

# Copy only installed packages from builder — no pip, no build tools
COPY --from=builder /install /usr/local

# Copy application source code
COPY . .

# Create a non-root user and switch to it — never run as root in production
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

# Expose the service port
EXPOSE <port>

# Start FastAPI with uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "<port>"]
```

---

### 1.3 Per-Service Dockerfile Details

#### Service 1: UI Interface
- **Port:** `8000`
- **No external API keys needed**
- Replace `<port>` with `8000` in the template above

#### Service 2: Backend API
- **Port:** `8001`
- **SQLite persistence:** Volume will be mounted at `/app/data` via Kubernetes (Step 6)
- DB path set via env var: `DATABASE_URL=sqlite:////app/data/tasks.db`
- Replace `<port>` with `8001`

#### Service 3: Todo Agent
- **Port:** `8002`
- **API keys (`OPENAI_API_KEY`, `GEMINI_API_KEY`) injected at runtime via K8s Secrets** — never in the image
- Replace `<port>` with `8002`

#### Service 4: Notification API
- **Port:** `8003`
- Lightweight service; no special volume or secrets needed
- Replace `<port>` with `8003`

---

### 1.4 .dockerignore (same for all 4 services)

```
__pycache__/
*.pyc
*.pyo
*.pyd
.env
.env.*
tests/
docs/
*.md
.git/
.gitignore
.pytest_cache/
htmlcov/
*.egg-info/
dist/
build/
```

---

### 1.5 Build & Push Commands

```bash
# Set your registry prefix (Docker Hub, ECR, GCR, etc.)
export REGISTRY=your-dockerhub-username   # e.g. myorg

# ── Build all 4 images ──────────────────────────────────────────────
docker build -t $REGISTRY/ui-interface:v1.0 ./ui-interface
docker build -t $REGISTRY/backend-api:v1.0 ./backend-api
docker build -t $REGISTRY/todo-agent:v1.0 ./todo-agent
docker build -t $REGISTRY/notification-api:v1.0 ./notification-api

# ── Push all 4 images to registry ───────────────────────────────────
docker push $REGISTRY/ui-interface:v1.0
docker push $REGISTRY/backend-api:v1.0
docker push $REGISTRY/todo-agent:v1.0
docker push $REGISTRY/notification-api:v1.0
```

---

### 1.6 Verification — Confirm Images Are Built & Pushed

```bash
# List all locally built images — confirm all 4 appear
docker images | grep -E "ui-interface|backend-api|todo-agent|notification-api"
```

**Expected output:**
```
your-dockerhub-username/ui-interface       v1.0    abc123def456   2 minutes ago   180MB
your-dockerhub-username/backend-api        v1.0    def456ghi789   2 minutes ago   195MB
your-dockerhub-username/todo-agent         v1.0    ghi789jkl012   2 minutes ago   210MB
your-dockerhub-username/notification-api   v1.0    jkl012mno345   2 minutes ago   178MB
```

```bash
# Test each image runs correctly before pushing to cluster
docker run --rm -p 8000:8000 $REGISTRY/ui-interface:v1.0
docker run --rm -p 8001:8001 $REGISTRY/backend-api:v1.0
docker run --rm -p 8002:8002 -e OPENAI_API_KEY=test -e GEMINI_API_KEY=test $REGISTRY/todo-agent:v1.0
docker run --rm -p 8003:8003 $REGISTRY/notification-api:v1.0

# Hit the health endpoint of each running container to confirm it responds
curl http://localhost:8000/health   # → {"status": "ok"}
curl http://localhost:8001/health   # → {"status": "ok"}
curl http://localhost:8002/health   # → {"status": "ok"}
curl http://localhost:8003/health   # → {"status": "ok"}
```

**Step 1 complete when:** All 4 images appear in `docker images` and all `/health` endpoints return `200 OK`.

---

## Step 2: Namespace

**Why second:** The namespace must exist before any other Kubernetes resource is created.
All subsequent resources (RBAC, ConfigMaps, Secrets, Deployments, Services) are scoped to
this namespace. Applying any resource without a namespace first will either go into `default`
(wrong) or fail.

---

### 2.1 File

```
k8s/namespace.yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: task-manager
  labels:
    app: task-manager
    env: production
```

---

### 2.2 Apply Command

```bash
kubectl apply -f k8s/namespace.yaml
```

---

### 2.3 Verification — Confirm Namespace Exists

```bash
# List all namespaces — task-manager should appear
kubectl get namespaces
```

**Expected output:**
```
NAME              STATUS   AGE
default           Active   10d
kube-system       Active   10d
kube-public       Active   10d
task-manager      Active   5s      ← newly created
```

```bash
# Describe the namespace for full detail
kubectl describe namespace task-manager
```

**Expected output:**
```
Name:         task-manager
Labels:       app=task-manager
              env=production
Status:       Active
```

**Step 2 complete when:** `kubectl get namespaces` shows `task-manager` with `STATUS: Active`.

---

## Step 3: RBAC

**Why third:** ServiceAccounts must exist before Deployments reference them. If a Deployment
references a ServiceAccount that doesn't exist, the pod will fail to schedule. Roles and
RoleBindings must also be in place before pods start making K8s API calls.

RBAC enforces the **principle of least privilege** — every service gets only the minimum
permissions it actually needs. Nothing more.

---

### 3.1 File Structure

```
k8s/rbac/
├── serviceaccounts/
│   ├── ui-interface-sa.yaml
│   ├── backend-api-sa.yaml
│   ├── todo-agent-sa.yaml
│   └── notification-api-sa.yaml
├── roles/
│   ├── ui-interface-role.yaml
│   ├── backend-api-role.yaml
│   ├── todo-agent-role.yaml
│   ├── notification-api-role.yaml
│   └── cicd-deploy-role.yaml
└── rolebindings/
    ├── ui-interface-rolebinding.yaml
    ├── backend-api-rolebinding.yaml
    ├── todo-agent-rolebinding.yaml
    ├── notification-api-rolebinding.yaml
    └── cicd-deploy-rolebinding.yaml
```

---

### 3.2 Per-Service RBAC

#### Service 1: UI Interface

**Needs:** Read its own ConfigMap only.
**Does NOT need:** Secrets, PVCs, Pod management.

```yaml
# serviceaccounts/ui-interface-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ui-interface-sa
  namespace: task-manager
automountServiceAccountToken: false   # UI never calls K8s API — disable token mount
---
# roles/ui-interface-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ui-interface-role
  namespace: task-manager
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["ui-interface-config"]  # locked to its OWN configmap only
    verbs: ["get"]
---
# rolebindings/ui-interface-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ui-interface-rolebinding
  namespace: task-manager
subjects:
  - kind: ServiceAccount
    name: ui-interface-sa
    namespace: task-manager
roleRef:
  kind: Role
  name: ui-interface-role
  apiGroup: rbac.authorization.k8s.io
```

---

#### Service 2: Backend API

**Needs:** Read its own ConfigMap + check its PVC status.
**Does NOT need:** Secrets, other services' configs, Pod creation.

```yaml
# roles/backend-api-role.yaml
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["backend-api-config"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    resourceNames: ["backend-api-sqlite-pvc"]
    verbs: ["get", "list"]
```

> PVC verbs: `get` and `list` only. The pod mounts the volume via Deployment spec — it never creates or deletes PVCs at runtime.

---

#### Service 3: Todo Agent

**Needs:** Read its own ConfigMap + Read `todo-agent-secrets` (API keys).
**Does NOT need:** Any other secret, PVC, Pod access, other services' config.

```yaml
# roles/todo-agent-role.yaml
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["todo-agent-config"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["todo-agent-secrets"]   # ONLY this secret — nothing else
    verbs: ["get"]
```

> `resourceNames` is critical. Without it, `verbs: ["get"]` on `secrets` grants access to
> **every** secret in the namespace — including other services' credentials.

---

#### Service 4: Notification API

**Needs:** Read its own ConfigMap only.
**Does NOT need:** Secrets, volumes, Pod management.

```yaml
# roles/notification-api-role.yaml
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["notification-api-config"]
    verbs: ["get"]
```

> `automountServiceAccountToken: false` — same as UI Interface; no K8s API calls needed at runtime.

---

#### Service 5: CI/CD Deploy Bot

**Needs:** Apply and update Deployments, Services, ConfigMaps, HPAs during automated deploys.
**Does NOT need:** Secrets (Vault handles that), cluster-level access, node management.

```yaml
# roles/cicd-deploy-role.yaml
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "create", "update", "patch"]
```

> Zero Secret access — a compromised CI/CD pipeline token cannot leak API credentials.

---

### 3.3 Permission Matrix

| ServiceAccount | ConfigMap | Secret | PVC | Deployments | Services | HPA |
|---------------|-----------|--------|-----|-------------|----------|-----|
| `ui-interface-sa` | get (own) | — | — | — | — | — |
| `backend-api-sa` | get (own) | — | get/list (own) | — | — | — |
| `todo-agent-sa` | get (own) | get (own) | — | — | — | — |
| `notification-api-sa` | get (own) | — | — | — | — | — |
| `cicd-deploy-sa` | get/create/update | — | — | get/create/update | get/create/update | get/create/update |

> `(own)` = `resourceNames` restricts access to only that service's specific named resource.

---

### 3.4 Apply Commands

```bash
# Apply in order: ServiceAccounts → Roles → RoleBindings
kubectl apply -f k8s/rbac/serviceaccounts/ -n task-manager
kubectl apply -f k8s/rbac/roles/ -n task-manager
kubectl apply -f k8s/rbac/rolebindings/ -n task-manager
```

---

### 3.5 Verification — Confirm RBAC Is Correct

```bash
# List all ServiceAccounts in the namespace
kubectl get serviceaccounts -n task-manager
```

**Expected output:**
```
NAME                   SECRETS   AGE
backend-api-sa         0         10s
cicd-deploy-sa         0         10s
default                0         2m
notification-api-sa    0         10s
todo-agent-sa          0         10s
ui-interface-sa        0         10s
```

```bash
# List all Roles
kubectl get roles -n task-manager
```

**Expected output:**
```
NAME                      CREATED AT
backend-api-role          2026-03-28T...
cicd-deploy-role          2026-03-28T...
notification-api-role     2026-03-28T...
todo-agent-role           2026-03-28T...
ui-interface-role         2026-03-28T...
```

```bash
# List all RoleBindings
kubectl get rolebindings -n task-manager
```

```bash
# Test permissions: confirm Todo Agent CAN read its own secret
kubectl auth can-i get secret/todo-agent-secrets \
  --as=system:serviceaccount:task-manager:todo-agent-sa \
  -n task-manager
# Expected: yes

# Test permissions: confirm UI Interface CANNOT read the Todo Agent secret
kubectl auth can-i get secret/todo-agent-secrets \
  --as=system:serviceaccount:task-manager:ui-interface-sa \
  -n task-manager
# Expected: no

# Test permissions: confirm Todo Agent CANNOT read UI's configmap
kubectl auth can-i get configmap/ui-interface-config \
  --as=system:serviceaccount:task-manager:todo-agent-sa \
  -n task-manager
# Expected: no
```

**Step 3 complete when:** All 5 ServiceAccounts, 5 Roles, and 5 RoleBindings are listed, and `kubectl auth can-i` tests return expected yes/no results.

---

## Step 4: ConfigMaps

**Why fourth:** ConfigMaps must exist before pods start. If a Deployment references a ConfigMap
via `envFrom` and it doesn't exist, the pod crashes immediately with `CreateContainerConfigError`.

---

### 4.1 What ConfigMaps Store

Non-sensitive configuration — safe in plaintext, safe to commit to Git.

**Rule:** If leaking the value causes no harm (a log level, an internal service URL), it belongs
in a ConfigMap. If it's a password or API key, it belongs in a Secret (Step 5).

---

### 4.2 Files — 4 total

```
k8s/configmaps/
├── ui-interface-configmap.yaml
├── backend-api-configmap.yaml
├── todo-agent-configmap.yaml
└── notification-api-configmap.yaml
```

---

### 4.3 ConfigMap Contents

#### UI Interface ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ui-interface-config
  namespace: task-manager
data:
  BACKEND_API_URL: "http://backend-api.task-manager.svc.cluster.local:8001"
  TODO_AGENT_URL: "http://todo-agent.task-manager.svc.cluster.local:8002"
  LOG_LEVEL: "INFO"
```

#### Backend API ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-api-config
  namespace: task-manager
data:
  DATABASE_URL: "sqlite:////app/data/tasks.db"
  NOTIFICATION_API_URL: "http://notification-api.task-manager.svc.cluster.local:8003"
  LOG_LEVEL: "INFO"
```

#### Todo Agent ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: todo-agent-config
  namespace: task-manager
data:
  BACKEND_API_URL: "http://backend-api.task-manager.svc.cluster.local:8001"
  LOG_LEVEL: "INFO"
```

#### Notification API ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: notification-api-config
  namespace: task-manager
data:
  BACKEND_API_URL: "http://backend-api.task-manager.svc.cluster.local:8001"
  UI_INTERFACE_URL: "http://ui-interface.task-manager.svc.cluster.local:80"
  LOG_LEVEL: "INFO"
```

---

### 4.4 How ConfigMaps Are Injected Into Pods

In each Deployment spec, ConfigMaps are injected as environment variables via `envFrom`:

```yaml
containers:
  - name: todo-agent
    image: your-registry/todo-agent:v1.0
    envFrom:
      - configMapRef:
          name: todo-agent-config   # all keys become env vars automatically
```

---

### 4.5 Apply Command

```bash
kubectl apply -f k8s/configmaps/ -n task-manager
```

---

### 4.6 Verification — Confirm ConfigMaps Are Created

```bash
# List all ConfigMaps in the namespace
kubectl get configmaps -n task-manager
```

**Expected output:**
```
NAME                      DATA   AGE
backend-api-config        3      5s
kube-root-ca.crt          1      5m    ← system configmap, ignore
notification-api-config   3      5s
todo-agent-config         2      5s
ui-interface-config       3      5s
```

```bash
# Inspect a specific ConfigMap to confirm values are correct
kubectl describe configmap todo-agent-config -n task-manager
```

**Expected output:**
```
Name:         todo-agent-config
Namespace:    task-manager
Data
====
BACKEND_API_URL:
  http://backend-api.task-manager.svc.cluster.local:8001
LOG_LEVEL:
  INFO
```

```bash
# Inspect all 4 ConfigMaps in detail
kubectl describe configmap ui-interface-config -n task-manager
kubectl describe configmap backend-api-config -n task-manager
kubectl describe configmap notification-api-config -n task-manager
```

**Step 4 complete when:** All 4 ConfigMaps appear in `kubectl get configmaps` and `describe` shows the correct key-value pairs.

---

## Step 5: Secrets & Security

**Why fifth:** Like ConfigMaps, Secrets must exist before pods start. The Todo Agent pod will
fail with `CreateContainerConfigError` if it references a Secret that doesn't yet exist.

---

### 5.1 The Problem with Raw Kubernetes Secrets

Kubernetes Secrets are **base64-encoded, not encrypted**. Anyone with kubectl access can decode instantly:

```bash
kubectl get secret todo-agent-secrets -n task-manager \
  -o jsonpath='{.data.OPENAI_API_KEY}' | base64 -d
# Output: sk-abc123...  ← real API key fully exposed
```

Base64 is encoding, not encryption. Raw Secrets alone provide zero security.

---

### 5.2 Security Solution: Defense in Depth (4 Layers)

#### Layer 1 — Encryption at Rest (etcd-level)

AES-256 encrypts all Secret values before writing to `etcd`. Even a direct etcd dump reveals nothing.

```yaml
# /etc/kubernetes/encryption-config.yaml  (applied on the API server node)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

Start API server with: `--encryption-provider-config=/etc/kubernetes/encryption-config.yaml`

After enabling, force-encrypt all existing Secrets:
```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

#### Layer 2 — RBAC Scoping

Only `todo-agent-sa` can read `todo-agent-secrets`. All other ServiceAccounts denied.
*(Defined in Step 3)*

#### Layer 3 — External Secrets Operator + HashiCorp Vault

**Never store actual values in Kubernetes at all.**

```
HashiCorp Vault (external)
       ↓  ESO fetches at runtime
External Secrets Operator (in cluster)
       ↓  injects real value into pod env
Todo Agent Pod
```

```yaml
# k8s/secrets/todo-agent-external-secret.yaml  ← safe to commit, no real values
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: todo-agent-external-secret
  namespace: task-manager
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: todo-agent-secrets
  data:
    - secretKey: OPENAI_API_KEY
      remoteRef:
        key: secret/task-manager/todo-agent
        property: openai_api_key
    - secretKey: GEMINI_API_KEY
      remoteRef:
        key: secret/task-manager/todo-agent
        property: gemini_api_key
```

- Vault logs every read — full audit trail
- Key rotation in Vault auto-propagates on next `refreshInterval: 1h`

#### Layer 4 — Never Commit Secrets to Git

```
# .gitignore
k8s/secrets/*-raw.yaml
.env
.env.*
*-secrets-values.yaml
```

---

### 5.3 Files

```
k8s/secrets/
├── todo-agent-external-secret.yaml   ← reference only, safe to commit
└── vault-secret-store.yaml           ← Vault connection config, safe to commit
```

---

### 5.4 Apply Command

```bash
kubectl apply -f k8s/secrets/ -n task-manager
```

---

### 5.5 Verification — Confirm Secrets Are Created & Secured

```bash
# List Secrets in namespace — todo-agent-secrets should appear
kubectl get secrets -n task-manager
```

**Expected output:**
```
NAME                   TYPE     DATA   AGE
todo-agent-secrets     Opaque   2      5s
```

```bash
# Confirm Secret has the expected keys (values stay hidden)
kubectl describe secret todo-agent-secrets -n task-manager
```

**Expected output:**
```
Name:         todo-agent-secrets
Namespace:    task-manager
Type:  Opaque

Data
====
GEMINI_API_KEY:   40 bytes   ← length shown, value hidden
OPENAI_API_KEY:   51 bytes   ← length shown, value hidden
```

```bash
# Confirm encryption at rest is active
kubectl get secret todo-agent-secrets -n task-manager -o yaml
# The 'data' values should be base64 strings but NOT decodable to real keys
# if encryption at rest is enabled on etcd

# Confirm Todo Agent SA can read the secret
kubectl auth can-i get secret/todo-agent-secrets \
  --as=system:serviceaccount:task-manager:todo-agent-sa \
  -n task-manager
# Expected: yes

# Confirm no other SA can read it
kubectl auth can-i get secret/todo-agent-secrets \
  --as=system:serviceaccount:task-manager:ui-interface-sa \
  -n task-manager
# Expected: no
```

**Step 5 complete when:** Secret appears in `kubectl get secrets`, `describe` shows keys with byte lengths (not values), and RBAC permission checks pass.

---

## Step 6: PersistentVolumeClaim (Backend API)

**Why sixth:** The PVC must be provisioned and bound before the Backend API pod starts.
If the pod starts before the PVC exists, the volume mount fails and the pod goes into
`Pending` or `CrashLoopBackOff`.

---

### 6.1 File

```
k8s/storage/backend-api-pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backend-api-sqlite-pvc
  namespace: task-manager
spec:
  accessModes:
    - ReadWriteOnce        # only 1 pod mounts at a time — correct for SQLite
  storageClassName: standard
  resources:
    requests:
      storage: 1Gi
```

**Why `ReadWriteOnce`:** SQLite is a single-file database. Multiple pods writing to the same
SQLite file simultaneously causes corruption. `ReadWriteOnce` ensures only one pod mounts
the volume at a time.

**Why `1Gi`:** Sufficient for task and session storage at this scale. Can be expanded later
via `kubectl edit pvc` without downtime on most storage classes.

---

### 6.2 How the PVC Is Mounted in the Backend API Deployment (Step 7 preview)

```yaml
# In backend-api-deployment.yaml
volumes:
  - name: sqlite-storage
    persistentVolumeClaim:
      claimName: backend-api-sqlite-pvc
containers:
  - name: backend-api
    volumeMounts:
      - name: sqlite-storage
        mountPath: /app/data       # SQLite file lives at /app/data/tasks.db
```

---

### 6.3 Apply Command

```bash
kubectl apply -f k8s/storage/backend-api-pvc.yaml -n task-manager
```

---

### 6.4 Verification — Confirm PVC Is Bound

```bash
# Check PVC status — must be Bound before proceeding
kubectl get pvc -n task-manager
```

**Expected output:**
```
NAME                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
backend-api-sqlite-pvc  Bound    pvc-a1b2c3d4-e5f6-7890-abcd-ef1234567890   1Gi        RWO            standard       10s
```

> `STATUS` must be `Bound`. If it stays `Pending`, the StorageClass may not have a provisioner.
> On Docker Desktop use `storageClassName: hostpath`. On Minikube use `standard`.

```bash
# Get full PVC details
kubectl describe pvc backend-api-sqlite-pvc -n task-manager
```

**Expected output includes:**
```
Status:        Bound
Capacity:      1Gi
Access Modes:  RWO
StorageClass:  standard
```

**Step 6 complete when:** `kubectl get pvc` shows `STATUS: Bound`.

---

## Step 7: Deployments

**Why seventh:** All dependencies are now ready — images built (Step 1), namespace exists
(Step 2), RBAC set (Step 3), ConfigMaps ready (Step 4), Secrets ready (Step 5), PVC bound
(Step 6). Pods can now start without any missing dependency.

**Deploy Backend API first** — UI Interface, Todo Agent, and Notification API all depend on
it being reachable. Deploying it first gives it time to become ready before the others start.

---

### 7.1 Files — 4 total

```
k8s/deployments/
├── backend-api-deployment.yaml       ← deploy first
├── todo-agent-deployment.yaml
├── notification-api-deployment.yaml
└── ui-interface-deployment.yaml      ← deploy last
```

---

### 7.2 Replica Count

| Service | Replicas | Reason |
|---------|----------|--------|
| UI Interface | 1 | Lightweight; HPA not needed |
| Backend API | 2 | All services depend on it; 2 replicas for baseline high availability |
| Todo Agent | 1 | HPA scales it under load (Step 9) |
| Notification API | 1 | Lightweight pass-through |

---

### 7.3 Resource Requests & Limits

| Service | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---------|------------|-----------|----------------|-------------|
| UI Interface | 100m | 300m | 128Mi | 256Mi |
| Backend API | 200m | 500m | 256Mi | 512Mi |
| Todo Agent | 300m | 1000m | 512Mi | 1Gi |
| Notification API | 100m | 300m | 128Mi | 256Mi |

> Todo Agent gets the highest limits — LLM API calls (OpenAI/Gemini) are I/O heavy and
> responses can consume significant memory during parsing.

---

### 7.4 Liveness & Readiness Probes

All 4 FastAPI services expose a `/health` endpoint used by Kubernetes for health checking:

**Readiness Probe** — pod receives traffic only when truly ready:
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8001
  initialDelaySeconds: 5    # wait 5s after container starts
  periodSeconds: 10         # check every 10s
  failureThreshold: 3       # mark unready after 3 failures
```

**Liveness Probe** — pod auto-restarted if it becomes permanently unresponsive:
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8001
  initialDelaySeconds: 30   # give app 30s to fully start first
  periodSeconds: 15         # check every 15s
  failureThreshold: 3       # restart after 3 consecutive failures
```

---

### 7.5 Rolling Update Strategy

Zero-downtime on every image update:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # bring up 1 new pod before killing old one
    maxUnavailable: 0  # never drop below desired replica count during update
```

**How it works:**
1. New image pushed → Kubernetes starts 1 new pod
2. Readiness probe confirms new pod is healthy
3. Old pod is terminated
4. Repeat for each replica
5. Traffic never drops — `maxUnavailable: 0` guarantees it

---

### 7.6 Full Backend API Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: task-manager
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      serviceAccountName: backend-api-sa
      containers:
        - name: backend-api
          image: your-registry/backend-api:v1.0
          ports:
            - containerPort: 8001
          envFrom:
            - configMapRef:
                name: backend-api-config
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8001
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 8001
            initialDelaySeconds: 30
            periodSeconds: 15
            failureThreshold: 3
          volumeMounts:
            - name: sqlite-storage
              mountPath: /app/data
      volumes:
        - name: sqlite-storage
          persistentVolumeClaim:
            claimName: backend-api-sqlite-pvc
```

---

### 7.7 Apply Commands (in dependency order)

```bash
# 1. Deploy Backend API first — all other services depend on it
kubectl apply -f k8s/deployments/backend-api-deployment.yaml -n task-manager

# Wait for Backend API to be fully ready before deploying the rest
kubectl rollout status deployment/backend-api -n task-manager
# Waits and prints: "deployment "backend-api" successfully rolled out"

# 2. Deploy Todo Agent and Notification API next
kubectl apply -f k8s/deployments/todo-agent-deployment.yaml -n task-manager
kubectl apply -f k8s/deployments/notification-api-deployment.yaml -n task-manager

# 3. Deploy UI Interface last — depends on both Backend API and Todo Agent
kubectl apply -f k8s/deployments/ui-interface-deployment.yaml -n task-manager

# Wait for all deployments to finish rolling out
kubectl rollout status deployment/todo-agent -n task-manager
kubectl rollout status deployment/notification-api -n task-manager
kubectl rollout status deployment/ui-interface -n task-manager
```

---

### 7.8 Verification — Confirm All Pods Are Running

```bash
# List all pods — all should show Running status
kubectl get pods -n task-manager
```

**Expected output:**
```
NAME                                READY   STATUS    RESTARTS   AGE
backend-api-7d6b8c9f4-xr2pq        1/1     Running   0          60s
backend-api-7d6b8c9f4-mn4kl        1/1     Running   0          60s
todo-agent-5f8c7d6b9-ab3cd         1/1     Running   0          45s
notification-api-9b4c5d7f2-ef5gh   1/1     Running   0          45s
ui-interface-3a2b1c8d7-ij6kl       1/1     Running   0          30s
```

```bash
# Watch pods in real-time as they start up
kubectl get pods -n task-manager --watch

# Check detailed status of a specific pod
kubectl describe pod <pod-name> -n task-manager

# Check logs of a running pod
kubectl logs deployment/backend-api -n task-manager
kubectl logs deployment/todo-agent -n task-manager
kubectl logs deployment/ui-interface -n task-manager
kubectl logs deployment/notification-api -n task-manager

# Check logs of a previous crashed pod (if troubleshooting)
kubectl logs deployment/backend-api -n task-manager --previous

# Confirm all deployments show desired = ready
kubectl get deployments -n task-manager
```

**Expected output for deployments:**
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
backend-api        2/2     2            2           2m
notification-api   1/1     1            1           90s
todo-agent         1/1     1            1           90s
ui-interface       1/1     1            1           60s
```

**Step 7 complete when:** All pods show `STATUS: Running`, `READY: 1/1` (or `2/2` for Backend API), and `RESTARTS: 0`.

---

## Step 8: Services

**Why eighth:** Services route traffic to pods using label selectors. Applying after Deployments
lets you confirm the selector labels match the pod labels before traffic is routed. Services
can be applied before Deployments, but the DNS name won't resolve until at least one matching
pod is running.

---

### 8.1 Files — 4 total

```
k8s/services/
├── ui-interface-service.yaml      ← NodePort (external access)
├── backend-api-service.yaml       ← ClusterIP (internal only)
├── todo-agent-service.yaml        ← ClusterIP (internal only)
└── notification-api-service.yaml  ← ClusterIP (internal only)
```

---

### 8.2 Service Type Decisions

| Service | Type | External? | Port | Reason |
|---------|------|-----------|------|--------|
| UI Interface | **NodePort** | Yes | 30080 | Only entry point for users; exposes UI without needing a cloud LB |
| Backend API | **ClusterIP** | No | 8001 | Internal only — called by UI, Todo Agent, Notification API |
| Todo Agent | **ClusterIP** | No | 8002 | Internal only — must never be publicly reachable |
| Notification API | **ClusterIP** | No | 8003 | Internal only — triggered by Backend API and UI |

---

### 8.3 Service YAML Examples

#### UI Interface — NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ui-interface
  namespace: task-manager
spec:
  type: NodePort
  selector:
    app: ui-interface
  ports:
    - protocol: TCP
      port: 80           # service port inside cluster
      targetPort: 8000   # container port
      nodePort: 30080    # external port on the node
```

#### Backend API — ClusterIP
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-api
  namespace: task-manager
spec:
  type: ClusterIP
  selector:
    app: backend-api
  ports:
    - protocol: TCP
      port: 8001
      targetPort: 8001
```

---

### 8.4 Inter-Service DNS (how services talk to each other)

Kubernetes automatically assigns each ClusterIP service a stable internal DNS name:

| From | To | Internal URL |
|------|----|-------------|
| UI Interface | Backend API | `http://backend-api.task-manager.svc.cluster.local:8001` |
| UI Interface | Todo Agent | `http://todo-agent.task-manager.svc.cluster.local:8002` |
| Todo Agent | Backend API | `http://backend-api.task-manager.svc.cluster.local:8001` |
| Backend API | Notification API | `http://notification-api.task-manager.svc.cluster.local:8003` |

**External user access:**
`http://<node-ip>:30080` → UI Interface pod on port 8000

---

### 8.5 Apply Command

```bash
kubectl apply -f k8s/services/ -n task-manager
```

---

### 8.6 Verification — Confirm Services Are Running

```bash
# List all services in namespace
kubectl get services -n task-manager
```

**Expected output:**
```
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
backend-api        ClusterIP   10.96.45.123     <none>        8001/TCP       10s
notification-api   ClusterIP   10.96.67.234     <none>        8003/TCP       10s
todo-agent         ClusterIP   10.96.89.012     <none>        8002/TCP       10s
ui-interface       NodePort    10.96.12.345     <none>        80:30080/TCP   10s
```

```bash
# Describe a service to confirm selector matches deployment labels
kubectl describe service backend-api -n task-manager
```

**Expected output includes:**
```
Selector:   app=backend-api
Endpoints:  10.244.0.5:8001,10.244.0.6:8001   ← 2 pod IPs (2 replicas)
```

```bash
# Test inter-service DNS resolution from inside the cluster
# Exec into the UI Interface pod and curl the Backend API
kubectl exec -it deployment/ui-interface -n task-manager -- \
  curl http://backend-api.task-manager.svc.cluster.local:8001/health
# Expected: {"status": "ok"}

# Test UI Interface is accessible externally
# Get the node IP first
kubectl get nodes -o wide
# Then curl the NodePort
curl http://<node-ip>:30080/health
# Expected: {"status": "ok"}

# On Docker Desktop, node IP is localhost
curl http://localhost:30080/health
```

**Step 8 complete when:** All 4 services appear in `kubectl get services`, `Endpoints` shows pod IPs (not `<none>`), and external curl to `localhost:30080` returns `200 OK`.

---

## Step 9: Horizontal Pod Autoscaler (HPA)

**Why ninth:** HPA attaches to an existing Deployment. If applied before the Deployment exists,
it will error. HPA also needs the Metrics Server running to collect CPU usage data.

---

### 9.1 Prerequisite — Verify Metrics Server

```bash
# Confirm Metrics Server is running
kubectl get deployment metrics-server -n kube-system
```

**Expected output:**
```
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           5d
```

If not running on Minikube:
```bash
minikube addons enable metrics-server
```

---

### 9.2 Files — 2 total

```
k8s/hpa/
├── todo-agent-hpa.yaml
└── backend-api-hpa.yaml
```

---

### 9.3 HPA Configuration

#### Todo Agent HPA
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: todo-agent-hpa
  namespace: task-manager
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: todo-agent
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # scale up when CPU > 70%
```

#### Backend API HPA
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-api-hpa
  namespace: task-manager
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-api
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

| Service | Min | Max | Trigger | Why |
|---------|-----|-----|---------|-----|
| Todo Agent | 1 | 5 | CPU > 70% | LLM calls are CPU/memory intensive; concurrent requests spike load |
| Backend API | 2 | 6 | CPU > 70% | All 3 services funnel through it; must absorb spikes |

---

### 9.4 Apply Command

```bash
kubectl apply -f k8s/hpa/ -n task-manager
```

---

### 9.5 Verification — Confirm HPA Is Active

```bash
# List HPAs
kubectl get hpa -n task-manager
```

**Expected output:**
```
NAME             REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
backend-api-hpa  Deployment/backend-api  15%/70%   2         6         2          30s
todo-agent-hpa   Deployment/todo-agent   10%/70%   1         5         1          30s
```

> `TARGETS` shows current CPU% / threshold. If it shows `<unknown>/70%`, wait 60s for Metrics Server to collect initial data.

```bash
# Describe HPA for full detail including scaling events
kubectl describe hpa todo-agent-hpa -n task-manager
```

**Expected output includes:**
```
Metrics:  ( current / target )
  resource cpu on pods  (as a percentage of request):  10% (30m) / 70%
Min replicas:   1
Max replicas:   5
Deployment pods:  1 current / 1 desired
```

**Step 9 complete when:** Both HPAs show `TARGETS` with a real CPU% value (not `<unknown>`) and `REPLICAS` matches the current deployment replica count.

---

## Step 10: PodDisruptionBudget (PDB)

**Why last:** PDB is a protection policy that requires running pods to protect. It guarantees
a minimum number of pods stay available during voluntary disruptions (node drains, cluster
upgrades, rolling restarts). Without it, all Backend API replicas could be terminated
simultaneously during maintenance, taking down the entire application.

---

### 10.1 File

```
k8s/pdb/backend-api-pdb.yaml
```

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-api-pdb
  namespace: task-manager
spec:
  minAvailable: 1          # always keep at least 1 Backend API pod running
  selector:
    matchLabels:
      app: backend-api
```

**Why Backend API only:** It is the single dependency for all other services. UI Interface,
Todo Agent, and Notification API all fail if Backend API is down. The PDB ensures
Kubernetes never voluntarily takes down all replicas at once.

---

### 10.2 Apply Command

```bash
kubectl apply -f k8s/pdb/backend-api-pdb.yaml -n task-manager
```

---

### 10.3 Verification — Confirm PDB Is Active

```bash
# List PodDisruptionBudgets
kubectl get pdb -n task-manager
```

**Expected output:**
```
NAME              MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
backend-api-pdb   1               N/A               1                     10s
```

> `ALLOWED DISRUPTIONS: 1` means with 2 replicas running and `minAvailable: 1`,
> Kubernetes is allowed to disrupt 1 pod at a time safely.

```bash
# Describe for full detail
kubectl describe pdb backend-api-pdb -n task-manager
```

**Expected output:**
```
Min available:   1
Current number of healthy pods:  2
Minimum number of healthy pods:  1
Allowed disruptions:             1
Status:                          Ok
```

**Step 10 complete when:** `kubectl get pdb` shows `backend-api-pdb` with `ALLOWED DISRUPTIONS: 1` and `Status: Ok`.

---

## Final Verification — Full System Health Check

Run these commands after all 10 steps to confirm the entire system is working end-to-end:

```bash
# ── 1. Check all resources in the namespace ──────────────────────────
kubectl get all -n task-manager

# ── 2. Confirm all pods are Running with 0 restarts ──────────────────
kubectl get pods -n task-manager

# ── 3. Confirm all deployments are fully rolled out ──────────────────
kubectl get deployments -n task-manager

# ── 4. Confirm all services have endpoints (not <none>) ──────────────
kubectl get endpoints -n task-manager

# ── 5. Confirm both HPAs are tracking real CPU metrics ───────────────
kubectl get hpa -n task-manager

# ── 6. Confirm PDB is protecting Backend API ─────────────────────────
kubectl get pdb -n task-manager

# ── 7. Confirm PVC is Bound ──────────────────────────────────────────
kubectl get pvc -n task-manager

# ── 8. End-to-end health check — hit every service from inside cluster ─
kubectl exec -it deployment/ui-interface -n task-manager -- \
  curl http://backend-api.task-manager.svc.cluster.local:8001/health
# Expected: {"status": "ok"}

kubectl exec -it deployment/ui-interface -n task-manager -- \
  curl http://todo-agent.task-manager.svc.cluster.local:8002/health
# Expected: {"status": "ok"}

kubectl exec -it deployment/ui-interface -n task-manager -- \
  curl http://notification-api.task-manager.svc.cluster.local:8003/health
# Expected: {"status": "ok"}

# ── 9. External access test ──────────────────────────────────────────
curl http://localhost:30080/health      # Docker Desktop
# OR
curl http://$(minikube ip):30080/health # Minikube
# Expected: {"status": "ok"}

# ── 10. Check events for any warnings or errors ──────────────────────
kubectl get events -n task-manager --sort-by='.lastTimestamp'
```

---

## Full Deployment Execution (All Commands in Order)

```bash
# ── STEP 1: Build & Push Images ──────────────────────────────────────
export REGISTRY=your-dockerhub-username

docker build -t $REGISTRY/ui-interface:v1.0 ./ui-interface
docker build -t $REGISTRY/backend-api:v1.0 ./backend-api
docker build -t $REGISTRY/todo-agent:v1.0 ./todo-agent
docker build -t $REGISTRY/notification-api:v1.0 ./notification-api

docker push $REGISTRY/ui-interface:v1.0
docker push $REGISTRY/backend-api:v1.0
docker push $REGISTRY/todo-agent:v1.0
docker push $REGISTRY/notification-api:v1.0

# ── STEP 2: Namespace ─────────────────────────────────────────────────
kubectl apply -f k8s/namespace.yaml
kubectl get namespaces | grep task-manager   # verify

# ── STEP 3: RBAC ──────────────────────────────────────────────────────
kubectl apply -f k8s/rbac/serviceaccounts/ -n task-manager
kubectl apply -f k8s/rbac/roles/ -n task-manager
kubectl apply -f k8s/rbac/rolebindings/ -n task-manager
kubectl get serviceaccounts -n task-manager  # verify

# ── STEP 4: ConfigMaps ────────────────────────────────────────────────
kubectl apply -f k8s/configmaps/ -n task-manager
kubectl get configmaps -n task-manager       # verify

# ── STEP 5: Secrets ───────────────────────────────────────────────────
kubectl apply -f k8s/secrets/ -n task-manager
kubectl get secrets -n task-manager          # verify

# ── STEP 6: PVC ───────────────────────────────────────────────────────
kubectl apply -f k8s/storage/backend-api-pvc.yaml -n task-manager
kubectl get pvc -n task-manager              # verify STATUS=Bound

# ── STEP 7: Deployments ───────────────────────────────────────────────
kubectl apply -f k8s/deployments/backend-api-deployment.yaml -n task-manager
kubectl rollout status deployment/backend-api -n task-manager   # wait

kubectl apply -f k8s/deployments/todo-agent-deployment.yaml -n task-manager
kubectl apply -f k8s/deployments/notification-api-deployment.yaml -n task-manager
kubectl apply -f k8s/deployments/ui-interface-deployment.yaml -n task-manager

kubectl rollout status deployment/todo-agent -n task-manager
kubectl rollout status deployment/notification-api -n task-manager
kubectl rollout status deployment/ui-interface -n task-manager
kubectl get pods -n task-manager             # verify all Running

# ── STEP 8: Services ──────────────────────────────────────────────────
kubectl apply -f k8s/services/ -n task-manager
kubectl get services -n task-manager         # verify
kubectl get endpoints -n task-manager        # verify endpoints have pod IPs

# ── STEP 9: HPA ───────────────────────────────────────────────────────
kubectl apply -f k8s/hpa/ -n task-manager
kubectl get hpa -n task-manager              # verify TARGETS shows CPU%

# ── STEP 10: PDB ──────────────────────────────────────────────────────
kubectl apply -f k8s/pdb/backend-api-pdb.yaml -n task-manager
kubectl get pdb -n task-manager              # verify ALLOWED DISRUPTIONS=1

# ── FINAL CHECK ───────────────────────────────────────────────────────
kubectl get all -n task-manager
curl http://localhost:30080/health           # external access test
```

---

## Final Project Structure

```
.
├── ui-interface/
│   ├── main.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .dockerignore
├── backend-api/
│   ├── main.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .dockerignore
├── todo-agent/
│   ├── main.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .dockerignore
├── notification-api/
│   ├── main.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .dockerignore
└── k8s/
    ├── namespace.yaml
    ├── rbac/
    │   ├── serviceaccounts/        (4 files: one per service)
    │   ├── roles/                  (5 files: 4 services + cicd)
    │   └── rolebindings/           (5 files: 4 services + cicd)
    ├── configmaps/                 (4 files: one per service)
    ├── secrets/                    (2 files: external-secret + vault-store)
    ├── storage/
    │   └── backend-api-pvc.yaml
    ├── deployments/                (4 files: one per service)
    ├── services/                   (4 files: one per service)
    ├── hpa/                        (2 files: todo-agent + backend-api)
    └── pdb/
        └── backend-api-pdb.yaml
```

---

**Total Kubernetes manifest files: 31**
**Total Dockerfiles: 4**
**Total project files per service: Dockerfile + .dockerignore + main.py + requirements.txt**
