# Session 3 Agenda — Full GitOps CD: ArgoCD + Helm

**Duration:** 1.5 hours | **Format:** Concept → Code walkthrough → Live demo

---

## Overview

| # | Topic | Type | Time |
|---|---|---|---|
| 1 | Why raw manifests don't scale | Discussion | 10 min |
| 2 | Helm chart anatomy | Concept | 10 min |
| 3 | Migration: raw deployment.yaml → Helm | Live walkthrough | 15 min |
| 4 | Full repo structure tour | Walkthrough | 15 min |
| 5 | Secrets in Helm | Concept + code | 10 min |
| 6 | NGINX Ingress in Helm | Concept + code | 5 min |
| 7 | Live demo: Git push → ArgoCD sync | Demo | 20 min |
| Q&A | — | — | 5 min |

---

## Section 1 (10 min): Why Raw Manifests Don't Scale

In Day 2 we wrote one `deployment.yaml` per service and applied it with `kubectl`.
That works fine for one service in one environment. It breaks down fast.

---

### The problem — copy-paste at scale

We have **9 services** and **3 environments** (dev / qa / prod).

Each service needs roughly 4–5 manifest files:
`deployment.yaml`, `service.yaml`, `configmap.yaml`, `ingress.yaml`, `serviceaccount.yaml`

```
9 services × 5 files = 45 files  (just for dev)
45 files   × 3 envs  = 135 files  (total)
```

Every one of those 135 files has the service name, namespace, image tag, and resource limits hardcoded.

---

### What happens when something needs to change

| Change | What you actually have to do |
|---|---|
| New image tag | Open and edit 9 `deployment.yaml` files per environment |
| Increase memory limit | Edit 9 files per environment |
| Add a staging environment | Copy all 45 files, rename every hardcoded value |
| Dev and prod configs drift | Nobody notices — until prod breaks |

---

### The fix — one template, many values files

Instead of 135 files with hardcoded values, Helm gives us:

- **1 template** — the structure, shared by all services
- **1 values file per service per environment** — just the differences

```
Before:  135 raw YAML files
After:   5 templates  +  25 values files  =  30 files total
```

Change the memory limit once in the template → all 9 services in all 3 environments pick it up.

---

## Section 2 (10 min): Helm Chart Anatomy

A Helm chart is just **three things working together**:

| File | Role |
|---|---|
| `Chart.yaml` | Who am I? (name, version) |
| `values.yaml` | What are my defaults? |
| `templates/*.yaml` | What do I render? |

---

### The minimal structure

```
helm-charts/
├── Chart.yaml       ← chart identity
├── values.yaml      ← default values (shared across all services)
└── templates/
    └── deployment.yaml   ← your k8s manifest with {{ .Values.xxx }} placeholders
```

That's it. A valid Helm chart needs only these three things.

---

### Chart.yaml — the four fields that matter

```yaml
apiVersion: v2
name: pharma-service
description: Common Helm chart for all Pharma microservices
version: 1.0.0
```

- `name` — what `helm install` calls this chart
- `version` — the chart's own version (bump when templates change)

Everything else (`keywords`, `maintainers`) is optional metadata.

---

### How the three files connect

```
Chart.yaml          → tells Helm the chart name
values.yaml         → supplies default values
templates/deployment.yaml → uses {{ .Values.xxx }} to read those values
                             ↓
                    helm template → renders final YAML → kubectl apply
```

When you run `helm install`, Helm reads `Chart.yaml`, merges `values.yaml` with any override file you pass (`-f envs/dev/values-api-gateway.yaml`), then renders every file in `templates/`.

---

### Our full chart — same structure, more templates

```
helm-charts/
├── Chart.yaml
├── values.yaml          ← shared defaults for all 9 services
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── ingress.yaml
    └── serviceaccount.yaml
```

**One chart. Nine services. Per-environment differences live only in the values files.**

```
envs/
├── dev/    ← 9 values files (one per service)
├── qa/     ← 8 values files
└── prod/   ← 8 values files
```

25 values files replace the ~126 raw manifest files from Day 2.

---

## Section 3 (15 min): Migration: api-gateway deployment.yaml → Helm

The idea is simple: **replace every hardcoded value with a `{{ .Values.xxx }}` placeholder.**
The values file then drives what gets rendered — per service, per environment.

### Step 1 — The raw manifest (Day 2)

Three things are hardcoded that we need to change per service or per environment:

```yaml
# lab2/manifests/api-gateway/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway                                                    # hardcoded
  namespace: dev                                                       # hardcoded
spec:
  replicas: 1                                                          # hardcoded
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
        - name: api-gateway
          image: 516209541629.dkr.ecr.us-east-1.amazonaws.com/api-gateway:sha-ef1ccc7  # hardcoded
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 60                                    # hardcoded
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

**The problem:** changing `replicas` or the image tag means editing this file for every service in every environment — 9 services × 3 environments = 27 edits for a single change.

---

### Step 2 — The Helm template (same file, values extracted)

Replace each hardcoded value with `{{ .Values.<key> }}`:

```yaml
# helm-charts/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.appName }}             # ← was: api-gateway
  namespace: {{ .Values.namespace }}      # ← was: dev
spec:
  replicas: {{ .Values.replicaCount }}    # ← was: 1
  selector:
    matchLabels:
      app: {{ .Values.appName }}
  template:
    metadata:
      labels:
        app: {{ .Values.appName }}
    spec:
      containers:
        - name: {{ .Values.appName }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"  # ← was: hardcoded ECR URL
          ports:
            - containerPort: {{ .Values.service.port }}
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: {{ .Values.service.port }}
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}  # ← was: 60
          resources:
            requests:
              cpu: {{ .Values.resources.requests.cpu }}
              memory: {{ .Values.resources.requests.memory }}
            limits:
              cpu: {{ .Values.resources.limits.cpu }}
              memory: {{ .Values.resources.limits.memory }}
```

The template is **identical in structure** to the raw manifest. The only change is swapping hardcoded strings for `{{ .Values.xxx }}` placeholders.

---

### Step 3 — The values file fills in the blanks

```yaml
# envs/dev/values-api-gateway.yaml
appName: api-gateway
namespace: dev
replicaCount: 1

image:
  repository: 516209541629.dkr.ecr.us-east-1.amazonaws.com/api-gateway
  tag: sha-ef1ccc7        # ← CI updates only this line on every build

service:
  port: 8080

livenessProbe:
  initialDelaySeconds: 60

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

---

### Step 4 — Preview what Helm renders

```bash
helm template api-gateway helm-charts/ -f envs/dev/values-api-gateway.yaml
```

The output is **byte-for-byte identical** to the Day 2 raw manifest. Helm just filled in the blanks.

---

### The payoff — one change, everywhere

| Scenario | Raw manifests | Helm |
|---|---|---|
| Update image tag | Edit 9 `deployment.yaml` files per env | Edit 1 values file (or CI does it) |
| Change probe timeout | Edit 9 files per env | Edit `values.yaml` default once |
| Add a new environment | Copy ~42 files, rename every field | Copy 9 values files |

**One template. Many values files. That's the whole idea.**

---

## Section 4 (15 min): Full Repo Structure Tour

There are three folders that matter. Everything else is supporting infrastructure.

```
zen-gitops/
├── helm-charts/   ← the ONE shared chart (templates live here)
├── envs/          ← the values files (one per service per environment)
└── argocd/        ← tells ArgoCD which chart + which values file to use
```

---

### helm-charts/ — the template

One chart, shared by all 9 services. The templates never change per service — only the values do.

```
helm-charts/
├── Chart.yaml
├── values.yaml          ← safe defaults (overridden by envs/ files)
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── ingress.yaml
    └── serviceaccount.yaml
```

---

### envs/ — the differences

One values file per service per environment. This is the only file CI touches on a new build.

```
envs/
├── dev/
│   ├── values-api-gateway.yaml
│   ├── values-auth-service.yaml
│   └── ... (9 files)
├── qa/    ← 8 files
└── prod/  ← 8 files
```

---

### argocd/apps/ — the wiring

Each ArgoCD `Application` CRD answers two questions: **which chart?** and **which values file?**

```yaml
# argocd/apps/dev/api-gateway-app.yaml
source:
  path: helm-charts                            # ← render this chart
  helm:
    valueFiles:
      - ../envs/dev/values-api-gateway.yaml   # ← with this values file

destination:
  namespace: dev                               # ← deploy here
```

That's the entire wiring. ArgoCD renders `helm template helm-charts/ -f envs/dev/values-api-gateway.yaml` and applies the result.

---

### The full flow — Git push to running pod

```
Developer pushes app code
        ↓
CI builds image → pushes to ECR → updates image.tag in envs/dev/values-api-gateway.yaml → git push
        ↓
ArgoCD detects zen-gitops changed
        ↓
helm template + kubectl apply
        ↓
New pod running in dev namespace
```

**CI only touches Git. ArgoCD owns the cluster. No one runs `kubectl apply` or `helm upgrade` by hand.**

---

## Section 5 (10 min): Secrets in Helm

### What the values file declares

```yaml
# envs/dev/values-api-gateway.yaml (envFrom section)
envFrom:
  - secretRef:
      name: db-credentials
  - secretRef:
      name: jwt-secret
```

The values file names the Secret — it never contains the secret value.

### What the template does with it

```yaml
# helm-charts/templates/deployment.yaml (envFrom block)
{{- if or .Values.configmap .Values.envFrom }}
envFrom:
  {{- if .Values.configmap }}
  - configMapRef:
      name: {{ include "pharma-service.fullname" . }}
  {{- end }}
  {{- with .Values.envFrom }}
  {{- toYaml . | nindent 12 }}
  {{- end }}
{{- end }}
```

Helm passes `envFrom` entries through verbatim. It only knows the Secret **name**, never the secret **value**. The rendered Deployment references `db-credentials` exactly as written in the values file.

### Where the K8s Secret actually comes from — ESO

```
AWS Secrets Manager: /pharma/dev/db-credentials
        │  (value: {"username": "pharmaadmin", "password": "..."})
        │  ESO polls every 1h via IRSA (no static AWS keys)
        ▼
ExternalSecret CRD (k8s/external-secrets/dev-external-secrets.yaml)
        │  creates/refreshes
        ▼
K8s Secret: db-credentials in dev namespace
        │  envFrom: secretRef
        ▼
api-gateway Pod — reads DB_PASSWORD as environment variable
```

### What never happens in this setup

```yaml
# WRONG — never put secret values in values files
env:
  - name: DB_PASSWORD
    value: "supersecret123"   # ← committed to Git permanently
```

Once a secret value is in Git history, it is compromised — even if you delete the file later.

### Verify the chain

```bash
kubectl get externalsecret -n dev
kubectl get secret db-credentials -n dev
kubectl exec -n dev deployment/api-gateway -- env | grep DB_USERNAME
```

---

## Section 6 (5 min): NGINX Ingress in Helm

### Which services expose an ingress

| Service | `ingress.enabled` | Reachable externally at |
|---|---|---|
| api-gateway | `true` | `/api` via NGINX |
| pharma-ui | `true` | `/` via NGINX |
| auth-service | `false` | `http://auth-service.dev.svc.cluster.local:8081` only |
| drug-catalog-service | `false` | internal only |
| notification-service | `false` | internal only |
| qc-service | `false` | internal only |
| (other services) | `false` | internal only |

### Values file ingress section (api-gateway)

```yaml
# envs/dev/values-api-gateway.yaml
ingress:
  enabled: true
  className: nginx
  annotations: {}
  host: ""
  path: /api
  pathType: Prefix
```

### The template conditional

```yaml
# helm-charts/templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "pharma-service.fullname" . }}
  labels:
    {{- include "pharma-service.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- toYaml .Values.ingress.tls | nindent 4 }}
  {{- end }}
  rules:
    - {{- if .Values.ingress.host }}
      host: {{ .Values.ingress.host | quote }}
      {{- end }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: {{ .Values.ingress.pathType }}
            backend:
              service:
                name: {{ include "pharma-service.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

When `ingress.enabled: false`, **no Ingress resource is created at all** — the entire file is skipped. `auth-service` is only reachable at `http://auth-service.dev.svc.cluster.local:8081` from within the cluster.

---

## Section 7 (20 min): Live Demo — Git Push → ArgoCD Sync

### Setup — verify current state

```bash
argocd app list
```

Expected output:

```
NAME                  CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  ...
api-gateway-dev       https://kubernetes.default.svc  dev        pharma   Synced  Healthy  Auto-Prune  ...
auth-service-dev      https://kubernetes.default.svc  dev        pharma   Synced  Healthy  Auto-Prune  ...
drug-catalog-dev      https://kubernetes.default.svc  dev        pharma   Synced  Healthy  Auto-Prune  ...
notification-dev      https://kubernetes.default.svc  dev        pharma   Synced  Healthy  Auto-Prune  ...
qc-service-dev        https://kubernetes.default.svc  dev        pharma   Synced  Healthy  Auto-Prune  ...
... (all 9 apps Synced/Healthy)
```

### Step 1: Simulate a CI image push — update image tag in Git

Edit `envs/dev/values-api-gateway.yaml`, change:

```yaml
image:
  tag: sha-ef1ccc7   # before
```

to:

```yaml
image:
  tag: sha-demo01    # simulated new CI build
```

Commit and push:

```bash
git add envs/dev/values-api-gateway.yaml
git commit -m "ci: update api-gateway dev image to sha-demo01"
git push
```

### Step 2: Watch ArgoCD detect the change

```bash
argocd app get api-gateway-dev --refresh
argocd app diff api-gateway-dev
```

Expected diff:

```diff
===== apps/Deployment dev/api-gateway ======
  image: 516209541629.dkr.ecr.us-east-1.amazonaws.com/api-gateway:sha-ef1ccc7
+ image: 516209541629.dkr.ecr.us-east-1.amazonaws.com/api-gateway:sha-demo01
```

### Step 3: Auto-sync fires (selfHeal: true)

Within ~3 minutes (or immediately after `--refresh`):

```bash
argocd app wait api-gateway-dev --health --sync
kubectl get pods -n dev -l app=api-gateway
kubectl describe pod -n dev -l app=api-gateway | grep "Image:"
```

The old pod terminates, new pod starts with `sha-demo01`.

### Step 4: Show rollback in the ArgoCD UI

1. Open ArgoCD UI → `api-gateway-dev` → **History & Rollback** tab
2. Two rows appear: previous commit (`sha-ef1ccc7`) and current commit (`sha-demo01`)
3. Click **Rollback** on the previous row → cluster reverts in ~30 seconds
4. Confirm:

```bash
kubectl describe pod -n dev -l app=api-gateway | grep "Image:"
```

**"This is the core GitOps loop. No `kubectl apply`. No `helm upgrade` in CI. Git is the source of truth; ArgoCD is the actuator."**

### What to highlight in the ArgoCD UI during the demo

| UI element | What it shows |
|---|---|
| App tile OutOfSync badge | Detected drift between Git and cluster |
| Diff view | Exactly which fields changed |
| Sync status bar | Rolling update progress |
| History tab | Every Git commit that triggered a sync |
| Self-heal toggle | `selfHeal: true` — manual kubectl changes auto-revert |

---

## Q&A Prep — Likely Questions

| Question | Answer |
|---|---|
| What if the Helm chart template itself changes? | All 9 apps re-render since they all share the same `helm-charts/` path. One template change propagates to every service in every environment. |
| How do we promote an image from dev to qa? | Update `envs/qa/values-<service>.yaml` `image.tag` in Git. CI can automate this after integration tests pass. |
| Can we use different replica counts per env? | Yes — `replicaCount: 1` in dev values, `replicaCount: 3` in prod values. The template reads `{{ .Values.replicaCount }}` from whichever file ArgoCD passes. |
| What happens if ArgoCD is down? | The cluster keeps running. Existing pods are unaffected. When ArgoCD restarts it reconciles from Git and catches up. |
| Why not use `helm upgrade` directly in CI? | Push-based vs pull-based. CI only touches Git, never the cluster. ArgoCD holds the cluster credentials. This means CI compromise does not equal cluster compromise. |
| Why does `helm template` show no namespace in the Deployment? | ArgoCD injects the namespace from `destination.namespace`. Templates should not hardcode namespace — ArgoCD overrides it at apply time. |

---

## Checkpoints

| Checkpoint | Command | Expected |
|---|---|---|
| ArgoCD apps all synced | `argocd app list` | All 9 apps `Synced`/`Healthy` |
| Image tag correct in dev | `kubectl describe pod -n dev -l app=api-gateway \| grep Image:` | `sha-ef1ccc7` |
| Ingress exists for api-gateway | `kubectl get ingress -n dev` | `api-gateway` row present |
| No ingress for auth-service | `kubectl get ingress -n dev` | No `auth-service` row |
| Secrets materialized | `kubectl get secret db-credentials -n dev` | Secret exists |
| ESO synced | `kubectl get externalsecret -n dev` | STATUS: `SecretSynced` |
