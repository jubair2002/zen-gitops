# Fluent Bit Deployment Guide (ArgoCD — `pharma` Project)

## Overview

Fluent Bit runs as a **DaemonSet** on every EKS node. It tails `/var/log/containers/*.log`, enriches
logs with Kubernetes metadata via the `kubernetes` filter, derives a `_service_name` field via a Lua
script, and ships logs to **Elastic Cloud** over TLS on port 443.

ArgoCD (project: `pharma`) manages the Helm chart at `helm-charts-fluent-bit/` using the env-specific
values file at `envs/dev/values-fluent-bit.yaml`.

---

## Architecture

```
EKS Node
└── /var/log/containers/*.log
        │
        ▼
  Fluent Bit DaemonSet (namespace: dev)
        │  tail → kubernetes filter → grep(dev) → lua(service_name)
        ▼
  Elastic Cloud (GCP us-central1)
  https://my-elasticsearch-project-be5821.es.us-central1.gcp.elastic.cloud:443
        │
        ▼
  Index pattern:  <service-name>-YYYY.MM.DD
```

---

## What Was Missing Before Applying via ArgoCD

The following items must exist in the cluster **before** ArgoCD syncs the Application, otherwise the
DaemonSet pods will fail to start with `CreateContainerConfigError`.

### 1. Kubernetes Secret — Elastic API Key

The DaemonSet reads `ELASTIC_API_KEY` from a Secret. ArgoCD does **not** create this Secret (it is
not in the Helm chart by design, to avoid storing credentials in Git).

**Create the secret manually once per cluster:**

```bash
kubectl create secret generic fluent-bit-elastic-credentials \
  --namespace dev \
  --from-literal=api_key='Y2xZRzc1NEJhZjZzUnhfam0wdHo6bExaQjdqQUw4aGJtbzFCdEp0SEh6UQ==' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Verify:

```bash
kubectl get secret fluent-bit-elastic-credentials -n dev
kubectl describe secret fluent-bit-elastic-credentials -n dev
```

> **Note:** The raw manifest at `k8s/fluent-bit/secret.yaml` contains an **old** API key tied to the
> previous Elastic deployment. Use the `kubectl create` command above with the current key.

---

### 2. Elasticsearch Host — Update Values File

`envs/dev/values-fluent-bit.yaml` and `k8s/fluent-bit/configmap.yaml` still reference the old
Elastic endpoint. Update both before pushing:

**`envs/dev/values-fluent-bit.yaml`**

```yaml
elasticsearch:
  host: my-elasticsearch-project-be5821.es.us-central1.gcp.elastic.cloud
  port: 443
  tls: true
  credentialsSecret: fluent-bit-elastic-credentials
```

**`k8s/fluent-bit/configmap.yaml`** — OUTPUT section:

```ini
[OUTPUT]
    Name                 es
    Match                kube.*
    Host                 my-elasticsearch-project-be5821.es.us-central1.gcp.elastic.cloud
    Port                 443
    TLS                  On
    TLS.Verify           On
    http_api_key         ${ELASTIC_API_KEY}
    Logstash_Format      On
    Logstash_Prefix_Key  _service_name
    Logstash_Prefix      dev-service
    Logstash_DateFormat  %Y.%m.%d
    Retry_Limit          5
    Suppress_Type_Name   On
    Replace_Dots         On
```

---

### 3. ArgoCD Project — Add Source Repo

`argocd/projects/pharma-project.yaml` currently allows only `ravdy/zen-gitops.git`. The fluent-bit
Application (`observability_project4/fluent-bit-app.yaml`) references `DPP-2026/zen-gitops.git`.
Add the forked repo to `sourceRepos`:

```yaml
sourceRepos:
  - "https://github.com/DPP-2026/zen-gitops.git"   # ← add this
  - "https://github.com/ravdy/zen-gitops.git"
  - "https://charts.helm.sh/stable"
  - "https://prometheus-community.github.io/helm-charts"
  - "https://kubernetes.github.io/ingress-nginx"
```

Apply the updated project:

```bash
kubectl apply -f argocd/projects/pharma-project.yaml
```

---

## Deployment Steps

### Step 1 — Create Elasticsearch on Elastic Cloud

1. Log in at <https://cloud.elastic.co/>
2. Create a new **Elasticsearch** deployment in region `GCP us-central1`
3. Copy the **Cloud endpoint**:
   ```
   https://my-elasticsearch-project-be5821.es.us-central1.gcp.elastic.cloud:443
   ```
4. Under **Security → API Keys**, create a key with `write` + `auto_configure` privileges on index
   pattern `*`.  
   The encoded key value is used in Step 2.

---

### Step 2 — Create the Kubernetes Secret

```bash
kubectl create secret generic fluent-bit-elastic-credentials \
  --namespace dev \
  --from-literal=api_key='Y2xZRzc1NEJhZjZzUnhfam0wdHo6bExaQjdqQUw4aGJtbzFCdEp0SEh6UQ==' \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

### Step 3 — Push Config Changes to Git

Update the two files listed in section 2 above, commit, and push to the
`feature/fluent-bit-logging` branch:

```bash
git add envs/dev/values-fluent-bit.yaml k8s/fluent-bit/configmap.yaml
git commit -m "fix: update Elastic Cloud endpoint to new project"
git push origin feature/fluent-bit-logging
```

---

### Step 4 — Apply the ArgoCD Project (if not already applied)

```bash
kubectl apply -f argocd/projects/pharma-project.yaml
```

---

### Step 5 — Apply the ArgoCD Application

```bash
kubectl apply -f observability_project4/fluent-bit-app.yaml
```

ArgoCD will now sync the `helm-charts-fluent-bit/` chart against the cluster. Monitor in the UI or:

```bash
argocd app get fluent-bit-dev
argocd app sync fluent-bit-dev   # force sync if needed
```

---

## Verification

### Check Pods Are Running

```bash
kubectl get pods -n dev -l app=fluent-bit
kubectl logs -n dev -l app=fluent-bit --tail=50
```

### Confirm Logs Are Reaching Elastic

```bash
# Health endpoint exposed by Fluent Bit HTTP server (port 2020)
kubectl port-forward -n dev ds/fluent-bit 2020:2020
curl http://localhost:2020/api/v1/health
curl http://localhost:2020/api/v1/metrics
```

In **Elastic Cloud → Discover**, search for index pattern `dev-service-*` or `<service-name>-*`.

### Prometheus Metrics (if kube-prometheus-stack is installed)

The DaemonSet pods expose `/api/v1/metrics/prometheus` on port `2020`.  
A `PodMonitor` at `k8s/monitoring/fluent-bit-podmonitor.yaml` scrapes this endpoint.

```bash
kubectl apply -f k8s/monitoring/fluent-bit-podmonitor.yaml
```

---

## File Reference

| File | Purpose |
|------|---------|
| `observability_project4/fluent-bit-app.yaml` | ArgoCD Application manifest |
| `helm-charts-fluent-bit/Chart.yaml` | Helm chart definition |
| `helm-charts-fluent-bit/values.yaml` | Default Helm values (host left blank) |
| `envs/dev/values-fluent-bit.yaml` | Dev-env overrides including ES host |
| `helm-charts-fluent-bit/templates/configmap.yaml` | Helm-templated Fluent Bit config |
| `helm-charts-fluent-bit/templates/daemonset.yaml` | DaemonSet template |
| `helm-charts-fluent-bit/templates/rbac.yaml` | ServiceAccount + ClusterRole |
| `k8s/fluent-bit/configmap.yaml` | Raw (non-Helm) ConfigMap with Lua script |
| `k8s/fluent-bit/daemonset.yaml` | Raw DaemonSet manifest |
| `k8s/fluent-bit/rbac.yaml` | Raw RBAC manifests |
| `k8s/fluent-bit/secret.yaml` | **Old key — do not apply directly** |
| `k8s/monitoring/fluent-bit-podmonitor.yaml` | Prometheus PodMonitor |
| `argocd/projects/pharma-project.yaml` | ArgoCD AppProject (add DPP-2026 repo) |

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Pods in `CreateContainerConfigError` | Secret `fluent-bit-elastic-credentials` missing | Run Step 2 |
| Logs not appearing in Elastic | Wrong ES host in configmap/values | Update host to `my-elasticsearch-project-be5821.es.us-central1.gcp.elastic.cloud` |
| ArgoCD sync error: `application repo not permitted` | Source repo not in AppProject | Add `DPP-2026/zen-gitops.git` to `pharma-project.yaml` |
| ArgoCD sync error: `revision not found` | Branch `feature/fluent-bit-logging` doesn't exist | Push the branch or update `targetRevision` |
| `401 Unauthorized` from Elastic | API key rotated or wrong key in secret | Re-create secret with current API key |
| No metrics in Prometheus | PodMonitor not applied | `kubectl apply -f k8s/monitoring/fluent-bit-podmonitor.yaml` |
