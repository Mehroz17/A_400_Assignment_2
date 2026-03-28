# AI Personal Employee — Kubernetes Deployment Plan

---

## Scenario Overview

A Personal AI Employee built on the **OpenAI Agents SDK** that acts as a digital assistant
capable of sending emails via Gmail, posting to LinkedIn, and messaging via WhatsApp on behalf
of the user.

| Service | Role |
|---|---|
| **AI Employee** | Single FastAPI service; runs the OpenAI Agent; integrates with Gmail, LinkedIn, WhatsApp |

**Tech Stack:** FastAPI · Python 3.12 · OpenAI Agents SDK · SQLite · Docker Desktop Kubernetes

**External API Integrations:**

| External Service | Purpose | Secret Required |
|---|---|---|
| OpenAI API | LLM inference (GPT-4o) | `OPENAI_API_KEY` |
| Google / Gmail API | Send emails on user's behalf | `GOOGLE_API_KEY` |
| LinkedIn API | Post updates, send messages | `LINKEDIN_API_TOKEN` |
| WhatsApp Business API | Send WhatsApp messages | `WHATSAPP_API_TOKEN` |
| SQLite (internal) | Persist agent memory + conversation history | `DATABASE_PASSWORD` |

---

## Task 2 Questions Answered Up Front

### Q1: How many ConfigMaps and Secrets?

| Resource | Count | Name | Keys |
|---|---|---|---|
| **ConfigMap** | 1 | `ai-employee-config` | `LOG_LEVEL`, `DB_PATH`, `OPENAI_MODEL`, `GMAIL_BASE_URL`, `LINKEDIN_BASE_URL`, `WHATSAPP_BASE_URL` |
| **Secret** | 1 | `ai-employee-secrets` | `OPENAI_API_KEY`, `GOOGLE_API_KEY`, `DATABASE_PASSWORD`, `LINKEDIN_API_TOKEN`, `WHATSAPP_API_TOKEN` |

> Rule: ConfigMaps hold **non-sensitive** config (safe to commit). Secrets hold **sensitive** credentials
> (never commit — managed via External Secrets Operator + HashiCorp Vault).

---

### Q2: RBAC — Will We Use It?

**Yes.** RBAC is mandatory even for a single-service deployment. Here is why:

- Without RBAC, any pod in the namespace can read any Secret — including your API keys
- The AI agent calls **external APIs only** (OpenAI, Gmail, etc.), not the Kubernetes API, so it
  needs a ServiceAccount with `automountServiceAccountToken: false`
- The CI/CD deploy bot needs permission to apply manifests but must **never** be able to read Secrets

**RBAC resources created:**

| Resource | Name | Purpose |
|---|---|---|
| ServiceAccount | `ai-employee-sa` | Identity for the AI Employee pod |
| Role | `ai-employee-role` | Read-only access to its own ConfigMap + Secret |
| RoleBinding | `ai-employee-rb` | Binds the role to the ServiceAccount |
| ServiceAccount | `deploy-bot-sa` | CI/CD pipeline identity |
| Role | `deploy-bot-role` | Apply deployments/services but **no Secret access** |
| RoleBinding | `deploy-bot-rb` | Binds deploy role to CI/CD bot |

---

### Q3: What Happens When a Secret Expires, Is Compromised, or Agent Access Is Needed?

#### Secret Expiry (Normal Rotation)
1. HashiCorp Vault issues secrets with a **TTL** (e.g., 24h for API tokens)
2. External Secrets Operator (ESO) polls Vault on a `refreshInterval: 1h` schedule
3. ESO automatically updates the K8s Secret before it expires
4. The pod picks up the new value on next restart **or** via a sidecar reload — zero downtime

#### Secret Compromise (Emergency Response Runbook)
```
1. Immediately revoke the compromised key at the source (OpenAI dashboard, Google Cloud, etc.)
2. kubectl delete secret ai-employee-secrets -n ai-employee   # purge from cluster immediately
3. Rotate the key in HashiCorp Vault:
      vault kv put secret/ai-employee/prod OPENAI_API_KEY=<new-key>
4. ESO will detect the change within refreshInterval and re-sync to the cluster
5. kubectl rollout restart deployment/ai-employee -n ai-employee   # force pod restart
6. kubectl rollout status deployment/ai-employee -n ai-employee    # confirm rollout complete
7. Audit Vault access logs to determine blast radius
```

#### Agent Access to Secrets
- The AI Employee pod accesses secrets **only via environment variables** injected by Kubernetes
- The pod never mounts the raw Secret file; it uses `env.valueFrom.secretKeyRef`
- RBAC `resourceNames` ensures the ServiceAccount can only read **its own named secret**,
  not any other secret in the namespace
- The OpenAI Agents SDK code reads `os.getenv("OPENAI_API_KEY")` — never hardcodes keys

---

## Deployment Order (Chronological)

```
Step 1  →  Dockerfiles           (build & push container image)
Step 2  →  Namespace             (create isolated environment in cluster)
Step 3  →  RBAC                  (identities & permissions before anything runs)
Step 4  →  ConfigMaps            (non-sensitive config must exist before pod starts)
Step 5  →  Secrets               (sensitive config must exist before pod starts)
Step 6  →  PersistentVolumeClaim (SQLite storage must exist before pod starts)
Step 7  →  Deployments           (start the pod — all dependencies now ready)
Step 8  →  Services              (expose pod for internal & external traffic)
Step 9  →  HPA                   (autoscaler — requires deployment to exist first)
Step 10 →  PodDisruptionBudget   (availability guarantee — requires running pod)
```

> Each step depends on the previous one. Applying out of order causes pods to crash on startup
> (missing config/secrets) or fail scheduling (missing ServiceAccount).

---

## Step 1: Dockerfiles — Build & Push Container Image

**Why first:** Everything in Kubernetes runs from a container image. The image must be built and
pushed to a registry before any Kubernetes manifest can reference it. If the image doesn't exist,
the pod will stay in `ImagePullBackOff` state forever.

---

### 1.1 Project Directory Structure (before writing Dockerfile)

```
.
└── ai-employee/
    ├── main.py
    ├── requirements.txt
    ├── Dockerfile
    └── .dockerignore
```

---

### 1.2 Dockerfile Architecture — 2-Stage Multi-Stage Build

The 2-stage build keeps the final image lean — build tools, pip caches, and compile artifacts
are **never shipped to production**.

```dockerfile
# ─── Stage 1: Builder ───────────────────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /app

# Copy requirements first — Docker caches this layer.
# pip install only re-runs when requirements.txt actually changes.
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ─── Stage 2: Runtime ───────────────────────────────────────────────────────
FROM python:3.12-slim AS runtime

WORKDIR /app

# Copy only the installed packages from the builder — no pip, no build tools
COPY --from=builder /install /usr/local

# Copy application source code
COPY . .

# Create a non-root user and switch to it — never run as root in production
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

# Expose the FastAPI service port
EXPOSE 8000

# Start FastAPI with uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

### 1.3 Per-Service Dockerfile Details

#### Service: AI Employee
- **Port:** `8000`
- **API keys** (`OPENAI_API_KEY`, `GOOGLE_API_KEY`, `LINKEDIN_API_TOKEN`, `WHATSAPP_API_TOKEN`) injected
  at runtime via K8s Secrets — **never baked into the image**
- **SQLite persistence:** Volume will be mounted at `/app/data` via Kubernetes (Step 6)
- DB path set via env var: `DB_PATH=/app/data/employee.db`

---

### 1.4 Key Decisions

| Decision | Why |
|---|---|
| `python:3.12-slim` not Alpine | Slim avoids Alpine's musl libc incompatibilities with many Python packages |
| 2-stage build | Build tools (gcc, pip cache) never reach production — reduces attack surface and image size |
| Non-root user (`appuser`) | Containers running as root can escape to the host if the container runtime is compromised |
| `--prefix=/install` | Clean separation of installed packages from the builder layer |

---

### 1.5 .dockerignore

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
data/
```

> `data/` is excluded because the SQLite database lives on a PersistentVolume at runtime —
> it must never be baked into the image.

---

### 1.6 Build & Push Commands

```bash
# Set your registry prefix (Docker Hub, ECR, GCR, etc.)
export REGISTRY=your-dockerhub-username   # e.g. myorg

# ── Build the image ──────────────────────────────────────────────────────────
docker build -t $REGISTRY/ai-employee:v1.0 ./ai-employee

# ── Push the image to registry ───────────────────────────────────────────────
docker push $REGISTRY/ai-employee:v1.0
```

---

### 1.7 Verification — Confirm Image Is Built & Pushed

```bash
# List locally built image — confirm it appears
docker images | grep ai-employee
```

**Expected output:**
```
your-dockerhub-username/ai-employee   v1.0   abc123def456   2 minutes ago   195MB
```

```bash
# Smoke test: run the container locally (pass dummy secrets so the app starts)
docker run --rm -p 8000:8000 \
  -e OPENAI_API_KEY=test \
  -e GOOGLE_API_KEY=test \
  -e LINKEDIN_API_TOKEN=test \
  -e WHATSAPP_API_TOKEN=test \
  -e DATABASE_PASSWORD=test \
  $REGISTRY/ai-employee:v1.0

# Confirm health endpoint responds
curl http://localhost:8000/health   # → {"status": "ok"}
```

**Step 1 complete when:** Image appears in `docker images` and `/health` returns `200 OK`.

---

## Step 2: Namespace

**Why second:** The namespace must exist before any other Kubernetes resource is created.
All subsequent resources (RBAC, ConfigMaps, Secrets, Deployments, Services) are scoped to
this namespace. Without it, resources either land in `default` (wrong) or fail to apply.

---

### 2.1 File

```
k8s/namespace.yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ai-employee
  labels:
    app: ai-employee
    env: production
```

---

### 2.2 Apply Command

```bash
kubectl apply -f k8s/namespace.yaml
```

---

### 2.3 Verification

```bash
kubectl get namespaces
```

**Expected output:**
```
NAME              STATUS   AGE
default           Active   10d
kube-system       Active   10d
kube-public       Active   10d
ai-employee       Active   5s      ← newly created
```

```bash
kubectl describe namespace ai-employee
```

**Expected output:**
```
Name:         ai-employee
Labels:       app=ai-employee
              env=production
Status:       Active
```

**Step 2 complete when:** `kubectl get namespaces` shows `ai-employee` with `STATUS: Active`.

---

## Step 3: RBAC

**Why third:** ServiceAccounts must exist before Deployments reference them. If a Deployment
references a ServiceAccount that doesn't exist, the pod will fail to schedule. Roles and
RoleBindings must also be in place before pods start.

RBAC enforces the **principle of least privilege** — every identity gets only the minimum
permissions it actually needs. Nothing more.

---

### 3.1 File Structure

```
k8s/rbac/
├── serviceaccounts/
│   ├── ai-employee-sa.yaml
│   └── deploy-bot-sa.yaml
├── roles/
│   ├── ai-employee-role.yaml
│   └── deploy-bot-role.yaml
└── rolebindings/
    ├── ai-employee-rb.yaml
    └── deploy-bot-rb.yaml
```

---

### 3.2 ServiceAccount — AI Employee

```yaml
# k8s/rbac/serviceaccounts/ai-employee-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ai-employee-sa
  namespace: ai-employee
  labels:
    app: ai-employee
automountServiceAccountToken: false   # Agent calls external APIs only, not the K8s API
```

> `automountServiceAccountToken: false` prevents the K8s API token from being auto-mounted
> into the pod. Since the AI agent only calls OpenAI/Gmail/LinkedIn/WhatsApp, it never needs
> to talk to the Kubernetes API. Removing the token eliminates one attack surface.

---

### 3.3 Role — AI Employee (read own ConfigMap + Secret only)

```yaml
# k8s/rbac/roles/ai-employee-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ai-employee-role
  namespace: ai-employee
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["ai-employee-config"]   # locked to this specific ConfigMap only
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["ai-employee-secrets"]  # locked to this specific Secret only
    verbs: ["get"]
```

> `resourceNames` is critical. Without it, a compromised pod could read **any** secret in the
> namespace (including other tenants' secrets in a shared cluster).

---

### 3.4 RoleBinding — AI Employee

```yaml
# k8s/rbac/rolebindings/ai-employee-rb.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ai-employee-rb
  namespace: ai-employee
subjects:
  - kind: ServiceAccount
    name: ai-employee-sa
    namespace: ai-employee
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: ai-employee-role
```

---

### 3.5 ServiceAccount — CI/CD Deploy Bot

```yaml
# k8s/rbac/serviceaccounts/deploy-bot-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deploy-bot-sa
  namespace: ai-employee
```

---

### 3.6 Role — CI/CD Deploy Bot (no Secret access)

```yaml
# k8s/rbac/roles/deploy-bot-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deploy-bot-role
  namespace: ai-employee
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "apply", "patch", "update"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "apply", "patch"]
  # Explicitly NO secrets access — deploy bot cannot read or write secrets
```

---

### 3.7 RoleBinding — CI/CD Deploy Bot

```yaml
# k8s/rbac/rolebindings/deploy-bot-rb.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deploy-bot-rb
  namespace: ai-employee
subjects:
  - kind: ServiceAccount
    name: deploy-bot-sa
    namespace: ai-employee
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: deploy-bot-role
```

---

### 3.8 Permission Matrix

| ServiceAccount | Can read ConfigMap | Can read Secret | Can apply Deployments | Can read other Secrets |
|---|---|---|---|---|
| `ai-employee-sa` | `ai-employee-config` only | `ai-employee-secrets` only | No | **No** |
| `deploy-bot-sa` | Yes (all) | **No** | Yes | **No** |

---

### 3.9 Apply Commands

```bash
# Apply in order: ServiceAccounts → Roles → RoleBindings
kubectl apply -f k8s/rbac/serviceaccounts/
kubectl apply -f k8s/rbac/roles/
kubectl apply -f k8s/rbac/rolebindings/
```

---

### 3.10 Verification

```bash
kubectl get serviceaccounts -n ai-employee
```

**Expected output:**
```
NAME               SECRETS   AGE
ai-employee-sa     0         10s
deploy-bot-sa      0         10s
default            0         1m
```

```bash
kubectl get roles -n ai-employee
```

**Expected output:**
```
NAME                 CREATED AT
ai-employee-role     2026-03-28T...
deploy-bot-role      2026-03-28T...
```

```bash
# ── Confirm ai-employee-sa CAN read its own secret ──────────────────────────
kubectl auth can-i get secret/ai-employee-secrets \
  --as=system:serviceaccount:ai-employee:ai-employee-sa \
  -n ai-employee
# Expected: yes

# ── Confirm ai-employee-sa CANNOT read any other secret ─────────────────────
kubectl auth can-i get secret/some-other-secret \
  --as=system:serviceaccount:ai-employee:ai-employee-sa \
  -n ai-employee
# Expected: no

# ── Confirm deploy-bot-sa CANNOT read secrets ────────────────────────────────
kubectl auth can-i get secret/ai-employee-secrets \
  --as=system:serviceaccount:ai-employee:deploy-bot-sa \
  -n ai-employee
# Expected: no
```

**Step 3 complete when:** All `can-i` checks return the expected `yes`/`no` results.

---

## Step 4: ConfigMaps

**Why fourth:** Pods reference ConfigMaps at startup. If the ConfigMap doesn't exist when the
pod starts, the pod will fail with `CreateContainerConfigError`.

**Rule: ConfigMaps contain non-sensitive values only.** Everything here is safe to commit to Git.
If a value would be dangerous to expose publicly, it belongs in a Secret (Step 5), not here.

---

### 4.1 File

```
k8s/configmaps/ai-employee-config.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ai-employee-config
  namespace: ai-employee
data:
  LOG_LEVEL: "INFO"
  DB_PATH: "/app/data/employee.db"
  OPENAI_MODEL: "gpt-4o"
  GMAIL_BASE_URL: "https://gmail.googleapis.com/gmail/v1"
  LINKEDIN_BASE_URL: "https://api.linkedin.com/v2"
  WHATSAPP_BASE_URL: "https://graph.facebook.com/v18.0"
```

> These values are safe to commit — no credentials, no tokens, only URLs, log levels, and paths.

---

### 4.2 How ConfigMap Is Injected Into the Pod

The Deployment (Step 7) will use `envFrom.configMapRef` to load **all** keys as environment variables:

```yaml
envFrom:
  - configMapRef:
      name: ai-employee-config
```

The FastAPI app then reads them with `os.getenv("LOG_LEVEL")`, `os.getenv("OPENAI_MODEL")`, etc.

---

### 4.3 Apply Command

```bash
kubectl apply -f k8s/configmaps/ai-employee-config.yaml
```

---

### 4.4 Verification

```bash
kubectl get configmaps -n ai-employee
```

**Expected output:**
```
NAME                   DATA   AGE
ai-employee-config     6      5s
kube-root-ca.crt       1      1m
```

```bash
kubectl describe configmap ai-employee-config -n ai-employee
```

**Expected output:**
```
Name:         ai-employee-config
Namespace:    ai-employee
Data
====
DB_PATH:              /app/data/employee.db
GMAIL_BASE_URL:       https://gmail.googleapis.com/gmail/v1
LINKEDIN_BASE_URL:    https://api.linkedin.com/v2
LOG_LEVEL:            INFO
OPENAI_MODEL:         gpt-4o
WHATSAPP_BASE_URL:    https://graph.facebook.com/v18.0
```

**Step 4 complete when:** `kubectl describe configmap ai-employee-config` shows all 6 keys.

---

## Step 5: Secrets & Security

**Why fifth:** Pods reference Secrets at startup. Secrets must exist before pods start, just
like ConfigMaps. However, Secrets require multiple security layers to protect them — this step
covers all four layers.

---

### 5.1 The Base64 Problem — Why Raw K8s Secrets Are Not Enough

Kubernetes Secrets are base64-encoded, **not encrypted**. Anyone with `kubectl get secret` access
can decode them instantly:

```bash
# The attack — one command reveals the plaintext key:
kubectl get secret ai-employee-secrets -n ai-employee -o jsonpath='{.data.OPENAI_API_KEY}' | base64 -d
# Output: sk-proj-abc123...  ← your real OpenAI key, in plaintext
```

This is why we apply **Defense in Depth** — 4 layers of protection.

---

### 5.2 Defense in Depth — 4 Security Layers

#### Layer 1: Encryption at Rest (AES-256 in etcd)

The Kubernetes API server can encrypt Secrets before writing them to etcd. Without this,
anyone with etcd access (or an etcd backup) reads your secrets in plaintext.

```yaml
# /etc/kubernetes/encryption-config.yaml  (on the API server node)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>   # generate: head -c 32 /dev/urandom | base64
      - identity: {}   # fallback for unencrypted secrets (remove after migration)
```

Enable on the API server with the flag:
```
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

> For Docker Desktop Kubernetes: encryption at rest is not configurable by default.
> Use Layer 3 (External Secrets Operator + Vault) as the primary protection in local dev.

---

#### Layer 2: RBAC Scoping (already done in Step 3)

- Only `ai-employee-sa` can read `ai-employee-secrets` (enforced by `resourceNames`)
- `deploy-bot-sa` has **zero** Secret access
- No wildcard `*` verbs on secrets anywhere

---

#### Layer 3: External Secrets Operator + HashiCorp Vault

This is the production-grade solution. Secrets live in Vault (with audit logging, rotation,
and access control). ESO syncs them into K8s Secrets automatically.

**Architecture:**
```
HashiCorp Vault
    │  (stores real plaintext secrets with TTL + audit logs)
    │
    ▼
External Secrets Operator (ESO)
    │  (polls Vault on refreshInterval, writes K8s Secret)
    │
    ▼
K8s Secret: ai-employee-secrets
    │  (encrypted at rest in etcd via Layer 1)
    │
    ▼
AI Employee Pod
    (reads OPENAI_API_KEY etc. as environment variables)
```

**Store the secrets in Vault first:**
```bash
vault kv put secret/ai-employee/prod \
  OPENAI_API_KEY="sk-proj-..." \
  GOOGLE_API_KEY="AIza..." \
  DATABASE_PASSWORD="securepassword123" \
  LINKEDIN_API_TOKEN="AQV..." \
  WHATSAPP_API_TOKEN="EAAb..."
```

**ExternalSecret manifest (safe to commit to Git — references only, no values):**

```yaml
# k8s/secrets/ai-employee-external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ai-employee-external-secret
  namespace: ai-employee
spec:
  refreshInterval: 1h          # ESO re-syncs from Vault every hour
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: ai-employee-secrets  # K8s Secret that ESO will create/update
    creationPolicy: Owner
  data:
    - secretKey: OPENAI_API_KEY
      remoteRef:
        key: secret/ai-employee/prod
        property: OPENAI_API_KEY
    - secretKey: GOOGLE_API_KEY
      remoteRef:
        key: secret/ai-employee/prod
        property: GOOGLE_API_KEY
    - secretKey: DATABASE_PASSWORD
      remoteRef:
        key: secret/ai-employee/prod
        property: DATABASE_PASSWORD
    - secretKey: LINKEDIN_API_TOKEN
      remoteRef:
        key: secret/ai-employee/prod
        property: LINKEDIN_API_TOKEN
    - secretKey: WHATSAPP_API_TOKEN
      remoteRef:
        key: secret/ai-employee/prod
        property: WHATSAPP_API_TOKEN
```

> **Why `refreshInterval: 1h`?** If a secret is rotated in Vault (e.g., due to compromise or
> scheduled rotation), ESO picks up the new value within 1 hour and updates the K8s Secret.
> Pair with `kubectl rollout restart deployment/ai-employee` to apply the new value immediately.

---

#### Layer 4: Never Commit Secrets to Git

Add to `.gitignore`:

```
.env
.env.*
k8s/secrets/*-values.yaml
*-secret.yaml
!k8s/secrets/*-external-secret.yaml   # ExternalSecret files are safe — no values
```

---

### 5.3 What to Commit vs. What to Never Commit

| File | Commit to Git? | Why |
|---|---|---|
| `k8s/secrets/ai-employee-external-secret.yaml` | **Yes** | Contains references only, no values |
| `k8s/configmaps/ai-employee-config.yaml` | **Yes** | Non-sensitive values |
| `.env` (local dev) | **Never** | Contains real secrets |
| `k8s/secrets/ai-employee-secrets.yaml` (raw Secret) | **Never** | Base64-encoded values are trivially decoded |
| `encryption-config.yaml` | **Never** | Contains the encryption key |

---

### 5.4 Secret Lifecycle — Expiry, Compromise, Agent Access

#### Secret Expiry (Scheduled Rotation)
```
Vault TTL expires for OPENAI_API_KEY
    → Vault auto-renews (if dynamic secrets) or alerts for manual rotation
    → ESO detects new version on next refreshInterval poll
    → ESO updates K8s Secret ai-employee-secrets
    → Pod reads new key on next restart (or use secret-reloader sidecar for hot reload)
```

#### Secret Compromise (Emergency Runbook)
```bash
# Step 1: Revoke immediately at the source
#   - OpenAI: https://platform.openai.com/api-keys → Delete key
#   - Google:  Google Cloud Console → API keys → Delete
#   - LinkedIn: Developer portal → revoke token
#   - WhatsApp: Meta Business → revoke token

# Step 2: Purge from cluster immediately
kubectl delete secret ai-employee-secrets -n ai-employee

# Step 3: Rotate in Vault with new credentials
vault kv put secret/ai-employee/prod \
  OPENAI_API_KEY="sk-proj-NEW..." \
  GOOGLE_API_KEY="AIzaNEW..."
  # ... (all 5 keys, even uncompromised ones — rotate all after a breach)

# Step 4: Force ESO re-sync (instead of waiting for refreshInterval)
kubectl annotate externalsecret ai-employee-external-secret \
  force-sync=$(date +%s) -n ai-employee --overwrite

# Step 5: Restart the pod to pick up new secrets
kubectl rollout restart deployment/ai-employee -n ai-employee
kubectl rollout status deployment/ai-employee -n ai-employee

# Step 6: Audit Vault access logs to find what accessed the compromised key
vault audit list
```

#### Agent Access to Secrets (How the Pod Reads Them)
The pod **never** reads secrets directly from the K8s API. Kubernetes injects them as
environment variables at pod startup:

```yaml
# In the Deployment (Step 7):
env:
  - name: OPENAI_API_KEY
    valueFrom:
      secretKeyRef:
        name: ai-employee-secrets
        key: OPENAI_API_KEY
  - name: GOOGLE_API_KEY
    valueFrom:
      secretKeyRef:
        name: ai-employee-secrets
        key: GOOGLE_API_KEY
  # ... repeat for remaining 3 secrets
```

The FastAPI app reads them with `os.getenv("OPENAI_API_KEY")` — the secret value never
appears in code, logs, or image layers.

---

### 5.5 Apply Command

```bash
kubectl apply -f k8s/secrets/ai-employee-external-secret.yaml
```

> ESO will create the `ai-employee-secrets` K8s Secret automatically after syncing with Vault.

---

### 5.6 Verification

```bash
kubectl get secrets -n ai-employee
```

**Expected output:**
```
NAME                    TYPE     DATA   AGE
ai-employee-secrets     Opaque   5      10s
```

```bash
# Confirm secret has 5 keys but values are NOT shown
kubectl describe secret ai-employee-secrets -n ai-employee
```

**Expected output:**
```
Name:         ai-employee-secrets
Namespace:    ai-employee
Type:         Opaque

Data
====
DATABASE_PASSWORD:      20 bytes
GOOGLE_API_KEY:         39 bytes
LINKEDIN_API_TOKEN:     28 bytes
OPENAI_API_KEY:         51 bytes
WHATSAPP_API_TOKEN:     32 bytes
```

```bash
# Confirm ai-employee-sa CAN read the secret
kubectl auth can-i get secret/ai-employee-secrets \
  --as=system:serviceaccount:ai-employee:ai-employee-sa \
  -n ai-employee
# Expected: yes

# Confirm deploy-bot-sa CANNOT read the secret
kubectl auth can-i get secret/ai-employee-secrets \
  --as=system:serviceaccount:ai-employee:deploy-bot-sa \
  -n ai-employee
# Expected: no
```

**Step 5 complete when:** Secret exists with 5 keys and `describe` shows byte lengths (not values).

---

## Step 6: PersistentVolumeClaim

**Why sixth:** The AI Employee uses SQLite to persist agent memory, conversation history,
and task state. Without a PersistentVolume, this data is lost every time the pod restarts.
The PVC must exist before the Deployment starts, or the pod will fail to start with a
`FailedMount` error.

---

### 6.1 File

```
k8s/storage/ai-employee-pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ai-employee-pvc
  namespace: ai-employee
spec:
  accessModes:
    - ReadWriteOnce    # SQLite is single-writer — only one pod writes at a time
  storageClassName: hostpath   # Docker Desktop's default storage class
  resources:
    requests:
      storage: 2Gi
```

> **Why `ReadWriteOnce`?** SQLite uses file locking and is designed for a single writer.
> `ReadWriteMany` would allow multiple pods to write simultaneously, corrupting the database.
> With `maxUnavailable: 0` in our rolling update strategy (Step 7), the new pod starts
> only after the old pod stops, preventing two pods from ever writing concurrently.

---

### 6.2 How PVC Is Mounted in the Deployment

```yaml
# In the Deployment spec (Step 7):
volumes:
  - name: employee-storage
    persistentVolumeClaim:
      claimName: ai-employee-pvc

containers:
  - name: ai-employee
    volumeMounts:
      - name: employee-storage
        mountPath: /app/data    # SQLite file lives at /app/data/employee.db
```

---

### 6.3 Apply Command

```bash
kubectl apply -f k8s/storage/ai-employee-pvc.yaml
```

---

### 6.4 Verification

```bash
kubectl get pvc -n ai-employee
```

**Expected output:**
```
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ai-employee-pvc     Bound    pvc-a1b2c3d4-e5f6-7890-abcd-ef1234567890   2Gi        RWO            hostpath       10s
```

> `STATUS` must be `Bound`. If it shows `Pending`, the storage class cannot provision the volume.

```bash
kubectl describe pvc ai-employee-pvc -n ai-employee
```

**Expected output:**
```
Name:          ai-employee-pvc
Namespace:     ai-employee
StorageClass:  hostpath
Status:        Bound
Volume:        pvc-a1b2c3...
Capacity:      2Gi
Access Modes:  RWO
```

**Step 6 complete when:** `kubectl get pvc` shows `STATUS: Bound`.

---

## Step 7: Deployments

**Why seventh:** All dependencies (namespace, RBAC, ConfigMap, Secret, PVC) are now in place.
The pod can start cleanly without any `CreateContainerConfigError` or `FailedMount` errors.

---

### 7.1 Resource Sizing

| Container | CPU Request | CPU Limit | Memory Request | Memory Limit | Justification |
|---|---|---|---|---|---|
| `ai-employee` | `500m` | `1000m` | `512Mi` | `1Gi` | AI/LLM inference is CPU-bound; OpenAI Agents SDK needs headroom for tool-call loops |

---

### 7.2 Deployment — AI Employee

```yaml
# k8s/deployments/ai-employee-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-employee
  namespace: ai-employee
  labels:
    app: ai-employee
spec:
  replicas: 2   # 2 replicas: one handles requests while the other is available during updates
  selector:
    matchLabels:
      app: ai-employee
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Start 1 new pod before terminating old ones
      maxUnavailable: 0    # Never have fewer than 2 pods serving traffic (critical for PVC single-writer)
  template:
    metadata:
      labels:
        app: ai-employee
    spec:
      serviceAccountName: ai-employee-sa
      automountServiceAccountToken: false

      containers:
        - name: ai-employee
          image: your-dockerhub-username/ai-employee:v1.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8000

          # ── Non-sensitive config from ConfigMap ──────────────────────────
          envFrom:
            - configMapRef:
                name: ai-employee-config

          # ── Sensitive secrets injected individually ──────────────────────
          env:
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: ai-employee-secrets
                  key: OPENAI_API_KEY
            - name: GOOGLE_API_KEY
              valueFrom:
                secretKeyRef:
                  name: ai-employee-secrets
                  key: GOOGLE_API_KEY
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ai-employee-secrets
                  key: DATABASE_PASSWORD
            - name: LINKEDIN_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: ai-employee-secrets
                  key: LINKEDIN_API_TOKEN
            - name: WHATSAPP_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: ai-employee-secrets
                  key: WHATSAPP_API_TOKEN

          # ── Resource limits ──────────────────────────────────────────────
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"

          # ── Readiness probe: only send traffic when app is ready ─────────
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10   # Wait 10s for app to boot
            periodSeconds: 10
            failureThreshold: 3

          # ── Liveness probe: restart pod if it stops responding ───────────
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 30   # Give more time before first liveness check
            periodSeconds: 15
            failureThreshold: 3

          # ── Volume mount for SQLite persistence ─────────────────────────
          volumeMounts:
            - name: employee-storage
              mountPath: /app/data

      volumes:
        - name: employee-storage
          persistentVolumeClaim:
            claimName: ai-employee-pvc
```

---

### 7.3 Apply Command

```bash
kubectl apply -f k8s/deployments/ai-employee-deployment.yaml

# Wait for the rollout to complete before moving on
kubectl rollout status deployment/ai-employee -n ai-employee
```

**Expected output:**
```
Waiting for deployment "ai-employee" rollout to finish: 0 of 2 updated replicas are available...
Waiting for deployment "ai-employee" rollout to finish: 1 of 2 updated replicas are available...
deployment "ai-employee" successfully rolled out
```

---

### 7.4 Verification

```bash
kubectl get pods -n ai-employee
```

**Expected output:**
```
NAME                           READY   STATUS    RESTARTS   AGE
ai-employee-7d6b8c9f4d-xk2pq   1/1     Running   0          30s
ai-employee-7d6b8c9f4d-mn7wz   1/1     Running   0          30s
```

> `RESTARTS: 0` is the key indicator — any restart means the pod is crashing (check logs).

```bash
kubectl get deployments -n ai-employee
```

**Expected output:**
```
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
ai-employee   2/2     2            2           1m
```

```bash
# Check logs from the deployment
kubectl logs deployment/ai-employee -n ai-employee

# If a pod is misbehaving, describe it for event details
kubectl describe pod <pod-name> -n ai-employee
```

**Step 7 complete when:** `kubectl get pods` shows `2/2 Running` with `0` restarts.

---

## Step 8: Services

**Why eighth:** Services create stable DNS names and load-balance traffic to pods. Without a
Service, pods are only reachable by their ephemeral IP addresses, which change on every restart.

---

### 8.1 Service Type Decision

| Service | Type | Reason |
|---|---|---|
| `ai-employee-svc` | `NodePort` | Docker Desktop maps NodePorts to `localhost` — enables external browser/curl access during local development |

> In production (AWS EKS, GKE): use `ClusterIP` + an **Ingress** with TLS termination instead
> of NodePort. NodePort is appropriate for local Docker Desktop development only.

---

### 8.2 Service — AI Employee

```yaml
# k8s/services/ai-employee-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: ai-employee-svc
  namespace: ai-employee
  labels:
    app: ai-employee
spec:
  type: NodePort
  selector:
    app: ai-employee
  ports:
    - name: http
      protocol: TCP
      port: 80          # Service port (cluster-internal)
      targetPort: 8000  # Container port (FastAPI)
      nodePort: 30800   # External port on localhost (Docker Desktop)
```

---

### 8.3 Port Mapping Table

| Layer | Port | Description |
|---|---|---|
| Container | `8000` | FastAPI uvicorn listens here |
| Service (ClusterIP) | `80` | Internal cluster DNS: `ai-employee-svc.ai-employee.svc.cluster.local:80` |
| NodePort | `30800` | External access: `http://localhost:30800` on Docker Desktop |

---

### 8.4 Inter-Service DNS

Since this is a single-service deployment, the DNS table is simple:

| From | To | URL |
|---|---|---|
| External user / browser | AI Employee | `http://localhost:30800` |
| AI Employee | OpenAI API | `https://api.openai.com` (external, no K8s DNS) |
| AI Employee | Gmail API | `https://gmail.googleapis.com` (external) |
| AI Employee | LinkedIn API | `https://api.linkedin.com` (external) |
| AI Employee | WhatsApp API | `https://graph.facebook.com` (external) |

---

### 8.5 Apply Command

```bash
kubectl apply -f k8s/services/ai-employee-svc.yaml
```

---

### 8.6 Verification

```bash
kubectl get services -n ai-employee
```

**Expected output:**
```
NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
ai-employee-svc    NodePort   10.96.145.32    <none>        80:30800/TCP   10s
```

```bash
# Endpoints must show pod IPs — if <none>, the selector is wrong
kubectl get endpoints -n ai-employee
```

**Expected output:**
```
NAME               ENDPOINTS                       AGE
ai-employee-svc    10.1.0.5:8000,10.1.0.6:8000    10s
```

```bash
# Test inter-service access from inside the cluster
kubectl exec -it deployment/ai-employee -n ai-employee -- \
  curl http://ai-employee-svc.ai-employee.svc.cluster.local/health
# Expected: {"status": "ok"}

# Test external access from your machine
curl http://localhost:30800/health
# Expected: {"status": "ok"}
```

**Step 8 complete when:** Endpoints show pod IPs and `curl http://localhost:30800/health` returns `200 OK`.

---

## Step 9: HPA (Horizontal Pod Autoscaler)

**Why ninth:** The HPA references the Deployment — it must exist first. The HPA scales the
AI Employee based on CPU load. OpenAI Agents SDK calls involve multiple LLM round trips
(tool calls, reasoning steps) which are CPU-intensive.

---

### 9.1 Prerequisite — Metrics Server

```bash
# Confirm Metrics Server is running (Docker Desktop includes it by default)
kubectl get deployment metrics-server -n kube-system
```

**Expected output:**
```
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           10d
```

---

### 9.2 HPA — AI Employee

```yaml
# k8s/hpa/ai-employee-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-employee-hpa
  namespace: ai-employee
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-employee
  minReplicas: 2    # Always keep 2 running — maintains high availability
  maxReplicas: 6    # Cap at 6 to control OpenAI API costs during spikes
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60   # Scale up when any pod exceeds 60% CPU
```

> **Why 60% CPU threshold?** AI agent workloads spike sharply during multi-step tool calls.
> At 60%, there is enough headroom to handle a burst before the new pod is Ready
> (typically 30-60 seconds for a new pod to pass readiness probes).

---

### 9.3 Apply Command

```bash
kubectl apply -f k8s/hpa/ai-employee-hpa.yaml
```

---

### 9.4 Verification

```bash
kubectl get hpa -n ai-employee
```

**Expected output:**
```
NAME               REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
ai-employee-hpa    Deployment/ai-employee   12%/60%   2         6         2          30s
```

> `TARGETS` must show a real CPU percentage (e.g., `12%/60%`), not `<unknown>/60%`.
> `<unknown>` means Metrics Server cannot reach the pods — check `kubectl top pods -n ai-employee`.

```bash
kubectl describe hpa ai-employee-hpa -n ai-employee
```

**Expected output (abbreviated):**
```
Name:                      ai-employee-hpa
Namespace:                 ai-employee
Reference:                 Deployment/ai-employee
Metrics:                   ( current / target )
  resource cpu on pods:    12% / 60%
Min replicas:              2
Max replicas:              6
Deployment pods:           2 current / 2 desired
```

**Step 9 complete when:** TARGETS shows a real CPU% value.

---

## Step 10: PodDisruptionBudget

**Why tenth:** The PDB constrains how many pods Kubernetes is allowed to evict at once during
voluntary disruptions (node drains, cluster upgrades). It must be applied after pods are running
so Kubernetes can validate the `minAvailable` math.

---

### 10.1 Disruption Math

With `replicas: 2` and `minAvailable: 1`:
- Allowed disruptions = 2 − 1 = **1 pod at a time**
- Kubernetes can evict 1 pod (for a node drain or upgrade) while the other keeps serving traffic
- The second pod is only evicted after the first is rescheduled and passes readiness probes

---

### 10.2 PDB — AI Employee

```yaml
# k8s/pdb/ai-employee-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ai-employee-pdb
  namespace: ai-employee
spec:
  minAvailable: 1   # At least 1 pod must remain running at all times
  selector:
    matchLabels:
      app: ai-employee
```

---

### 10.3 Apply Command

```bash
kubectl apply -f k8s/pdb/ai-employee-pdb.yaml
```

---

### 10.4 Verification

```bash
kubectl get pdb -n ai-employee
```

**Expected output:**
```
NAME                MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
ai-employee-pdb     1               N/A               1                     10s
```

> `ALLOWED DISRUPTIONS` must be `> 0`. If it shows `0`, there are not enough running pods
> to satisfy `minAvailable` — the HPA may not have scaled up yet.

```bash
kubectl describe pdb ai-employee-pdb -n ai-employee
```

**Expected output:**
```
Name:             ai-employee-pdb
Namespace:        ai-employee
Min available:    1
Selector:         app=ai-employee
Status:
    Allowed disruptions:  1
    Current:              2
    Desired:              1
    Total:                2
Events:           <none>
```

**Step 10 complete when:** `ALLOWED DISRUPTIONS` is `1`.

---

## Final System Health Check

Run these 10 commands in order to confirm the entire system is working end-to-end:

```bash
# 1. All resources in the namespace
kubectl get all -n ai-employee

# 2. All pods Running, 0 restarts
kubectl get pods -n ai-employee

# 3. Deployment READY count matches replicas
kubectl get deployments -n ai-employee

# 4. Endpoints have real pod IPs (not <none>)
kubectl get endpoints -n ai-employee

# 5. HPA shows real CPU% in TARGETS (not <unknown>)
kubectl get hpa -n ai-employee

# 6. PDB ALLOWED DISRUPTIONS > 0
kubectl get pdb -n ai-employee

# 7. PVC STATUS is Bound
kubectl get pvc -n ai-employee

# 8. Inter-service health check from inside the cluster
kubectl exec -it deployment/ai-employee -n ai-employee -- \
  curl http://ai-employee-svc.ai-employee.svc.cluster.local/health
# Expected: {"status": "ok"}

# 9. External access from your machine
curl http://localhost:30800/health
# Expected: {"status": "ok"}

# 10. Check for warnings or errors in recent cluster events
kubectl get events -n ai-employee --sort-by='.lastTimestamp'
```

---

## Full Deployment Execution Block

Copy and paste this entire block to deploy from scratch:

```bash
#!/bin/bash
set -e   # Stop on any error

export REGISTRY=your-dockerhub-username

# ── Step 1: Build & Push Image ───────────────────────────────────────────────
docker build -t $REGISTRY/ai-employee:v1.0 ./ai-employee
docker push $REGISTRY/ai-employee:v1.0
docker images | grep ai-employee   # verify

# ── Step 2: Namespace ────────────────────────────────────────────────────────
kubectl apply -f k8s/namespace.yaml
kubectl get namespaces | grep ai-employee   # verify: Active

# ── Step 3: RBAC ─────────────────────────────────────────────────────────────
kubectl apply -f k8s/rbac/serviceaccounts/
kubectl apply -f k8s/rbac/roles/
kubectl apply -f k8s/rbac/rolebindings/
kubectl get serviceaccounts -n ai-employee   # verify

# ── Step 4: ConfigMaps ───────────────────────────────────────────────────────
kubectl apply -f k8s/configmaps/ai-employee-config.yaml
kubectl get configmaps -n ai-employee   # verify: 1 configmap (+ kube-root-ca.crt)

# ── Step 5: Secrets (via External Secrets Operator) ──────────────────────────
kubectl apply -f k8s/secrets/ai-employee-external-secret.yaml
kubectl get secrets -n ai-employee   # verify: ai-employee-secrets exists

# ── Step 6: PersistentVolumeClaim ────────────────────────────────────────────
kubectl apply -f k8s/storage/ai-employee-pvc.yaml
kubectl get pvc -n ai-employee   # verify: STATUS Bound

# ── Step 7: Deployment ───────────────────────────────────────────────────────
kubectl apply -f k8s/deployments/ai-employee-deployment.yaml
kubectl rollout status deployment/ai-employee -n ai-employee   # wait for rollout
kubectl get pods -n ai-employee   # verify: 2/2 Running, 0 restarts

# ── Step 8: Services ─────────────────────────────────────────────────────────
kubectl apply -f k8s/services/ai-employee-svc.yaml
kubectl get services -n ai-employee   # verify: NodePort 30800
kubectl get endpoints -n ai-employee  # verify: pod IPs listed

# ── Step 9: HPA ──────────────────────────────────────────────────────────────
kubectl apply -f k8s/hpa/ai-employee-hpa.yaml
kubectl get hpa -n ai-employee   # verify: real CPU% in TARGETS

# ── Step 10: PodDisruptionBudget ─────────────────────────────────────────────
kubectl apply -f k8s/pdb/ai-employee-pdb.yaml
kubectl get pdb -n ai-employee   # verify: ALLOWED DISRUPTIONS > 0

# ── Final health check ───────────────────────────────────────────────────────
curl http://localhost:30800/health   # Expected: {"status": "ok"}
echo "Deployment complete."
```

---

## Final Project Directory Structure

```
.
├── ai-employee/                          # Service source
│   ├── main.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .dockerignore
│
└── k8s/                                  # Kubernetes manifests
    ├── namespace.yaml                    # 1 file
    ├── rbac/
    │   ├── serviceaccounts/
    │   │   ├── ai-employee-sa.yaml       # 2 files
    │   │   └── deploy-bot-sa.yaml
    │   ├── roles/
    │   │   ├── ai-employee-role.yaml     # 2 files
    │   │   └── deploy-bot-role.yaml
    │   └── rolebindings/
    │       ├── ai-employee-rb.yaml       # 2 files
    │       └── deploy-bot-rb.yaml
    ├── configmaps/
    │   └── ai-employee-config.yaml       # 1 file
    ├── secrets/
    │   └── ai-employee-external-secret.yaml   # 1 file (safe to commit — no values)
    ├── storage/
    │   └── ai-employee-pvc.yaml          # 1 file
    ├── deployments/
    │   └── ai-employee-deployment.yaml   # 1 file
    ├── services/
    │   └── ai-employee-svc.yaml          # 1 file
    ├── hpa/
    │   └── ai-employee-hpa.yaml          # 1 file
    └── pdb/
        └── ai-employee-pdb.yaml          # 1 file

Total: 4 source files + 14 Kubernetes manifests = 18 files
```
