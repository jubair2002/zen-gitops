# Prometheus + Grafana — ArgoCD Deployment Guide

## Overview

This sets up `kube-prometheus-stack` (Prometheus + Grafana + Alertmanager) via ArgoCD.

| What | Detail |
|------|--------|
| Metrics collected | Pod-level (kube-state-metrics) + Node-level (node-exporter) |
| Metric retention | **14 days (2 weeks)** |
| Storage driver | **AWS EBS CSI driver** — `gp3` volumes via `gp3-ebs-csi` StorageClass |
| Dashboards | **None provisioned automatically** — you add them manually in Grafana |
| Alert rules | **None created automatically** — you add PrometheusRule objects manually |
| Namespace | `monitoring` |
| Grafana URL | `https://monitoring.pharma.internal` (via NGINX ingress) |

---

## Files Added

```
observability_project4/
├── monitoring-app.yaml          ← ArgoCD Application CR  (apply this once)
├── prometheus-values.yaml       ← Helm values for kube-prometheus-stack
└── k8s/
    └── ebs-storageclass.yaml    ← StorageClass backed by the EBS CSI driver
```

---

## Pre-requisite: EBS CSI Driver

The EBS CSI driver (`ebs.csi.aws.com`) must be installed on the EKS cluster.  
EKS cluster and ArgoCD are already running, so check whether the add-on is present:

```bash
aws eks describe-addon \
  --cluster-name <your-cluster-name> \
  --addon-name aws-ebs-csi-driver \
  --query 'addon.status'
```

Expected output: `"ACTIVE"` — if so, skip to [Step 1](#step-1--push-this-branch-to-github).

If not installed, follow the three steps below.

---

### A — Fetch cluster details (needed for the trust policy)

```bash
# Set these once — used in every command below
CLUSTER_NAME=<your-cluster-name>
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)

# Get the OIDC issuer URL for the cluster (no https:// prefix needed for IAM)
OIDC_ISSUER=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query "cluster.identity.oidc.issuer" \
  --output text | sed 's|https://||')

echo "Account : $AWS_ACCOUNT_ID"
echo "Region  : $AWS_REGION"
echo "OIDC    : $OIDC_ISSUER"
```

---

### B — Create the IAM role with OIDC trust policy

The EBS CSI driver runs as the `ebs-csi-controller-sa` ServiceAccount in `kube-system`.
The IAM role must trust that specific ServiceAccount via the cluster's OIDC provider.

```bash
# 1. Write the trust policy document
cat > /tmp/ebs-csi-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_ISSUER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_ISSUER}:aud": "sts.amazonaws.com",
          "${OIDC_ISSUER}:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }
  ]
}
EOF

# 2. Create the IAM role
aws iam create-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --assume-role-policy-document file:///tmp/ebs-csi-trust-policy.json \
  --description "IAM role for the AWS EBS CSI driver on EKS cluster ${CLUSTER_NAME}"

# 3. Attach the AWS-managed EBS CSI policy to the role
aws iam attach-role-policy \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy

# 4. Confirm the policy is attached
aws iam list-attached-role-policies \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --query 'AttachedPolicies[*].PolicyName'
```

Expected output of step 4: `[ "AmazonEBSCSIDriverPolicy" ]`

---

### C — Install the EBS CSI add-on

```bash
aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole \
  --resolve-conflicts OVERWRITE

# Wait for the add-on to become ACTIVE (takes ~60–90 seconds)
aws eks wait addon-active \
  --cluster-name $CLUSTER_NAME \
  --addon-name aws-ebs-csi-driver

# Confirm
aws eks describe-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name aws-ebs-csi-driver \
  --query 'addon.status'
```

Expected output: `"ACTIVE"`

Verify the driver pods are running:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

You should see `ebs-csi-controller-*` (2 replicas) and `ebs-csi-node-*` (one per node) all in `Running` state.

---

## How Storage Works

```
Prometheus pod
    │
    │  PersistentVolumeClaim (20 Gi)
    │  storageClassName: gp3-ebs-csi
    ▼
StorageClass: gp3-ebs-csi          ← observability_project4/k8s/ebs-storageclass.yaml
    │  provisioner: ebs.csi.aws.com
    │  type: gp3 | iops: 3000 | throughput: 125 MB/s | encrypted: true
    │  volumeBindingMode: WaitForFirstConsumer  (volume created in same AZ as pod)
    │  reclaimPolicy: Retain  (EBS volume survives PVC deletion — protects your data)
    ▼
AWS EBS gp3 volume (20 Gi)
    │
    └── Prometheus stores metrics here for 14 days
        retentionSize guard: 18 GB (prevents the 20 Gi PVC from filling completely)
```

Grafana and Alertmanager each get their own `gp3-ebs-csi` volumes (5 Gi each).

---

## Step 1 — Push This Branch to GitHub

ArgoCD pulls from the git repo, so the new files must be on a remote branch first.

```bash
git add observability_project4/monitoring-app.yaml \
        observability_project4/prometheus-values.yaml \
        observability_project4/k8s/ebs-storageclass.yaml \
        observability_project4/implementation.md
git commit -m "feat: add prometheus+grafana monitoring via ArgoCD with EBS CSI storage"
git push
```

---

## Step 2 — Update targetRevision in monitoring-app.yaml

Open `observability_project4/monitoring-app.yaml` and set `targetRevision` on both git
sources to the branch/tag you just pushed to:

```yaml
sources:
  ...
  - repoURL: https://github.com/DPP-2026/zen-gitops.git
    targetRevision: <your-branch-name>    # ← update here
    ref: values

  - repoURL: https://github.com/DPP-2026/zen-gitops.git
    targetRevision: <your-branch-name>    # ← update here
    path: observability_project4/k8s
```

---

## Step 3 — Remove the Old monitoring-app (if it exists)

If the previous `argocd/apps/dev/monitoring-app.yaml` was already applied to the cluster,
delete it first to avoid a name conflict:

```bash
kubectl delete application monitoring -n argocd
```

> ArgoCD apps with `finalizers` will clean up their managed resources on deletion.
> If you want to keep the existing Prometheus data, remove the finalizer first:
> ```bash
> kubectl patch application monitoring -n argocd \
>   -p '{"metadata":{"finalizers":[]}}' --type=merge
> kubectl delete application monitoring -n argocd
> ```

---

## Step 4 — Apply the ArgoCD Application

```bash
kubectl apply -f observability_project4/monitoring-app.yaml
```

ArgoCD will then:
1. Apply `observability_project4/k8s/ebs-storageclass.yaml` → creates the `gp3-ebs-csi` StorageClass
2. Deploy `kube-prometheus-stack` via Helm using `observability_project4/prometheus-values.yaml`
3. Dynamically provision three EBS gp3 volumes (Prometheus 20 Gi, Grafana 5 Gi, Alertmanager 5 Gi)

---

## Step 5 — Monitor Sync

```bash
# Watch ArgoCD sync status
argocd app get monitoring

# Force sync if needed
argocd app sync monitoring

# Watch pods come up in the monitoring namespace
kubectl get pods -n monitoring -w
```

Expected pods (all should reach `Running`):

| Pod | Purpose |
|-----|---------|
| `monitoring-kube-prometheus-prometheus-*` | Prometheus server |
| `monitoring-grafana-*` | Grafana |
| `monitoring-kube-prometheus-alertmanager-*` | Alertmanager |
| `monitoring-kube-state-metrics-*` | Pod-level metrics |
| `monitoring-prometheus-node-exporter-*` | Node-level metrics (one per node) |
| `monitoring-kube-prometheus-operator-*` | Prometheus Operator |

---

## Step 6 — Verify Storage

Confirm the EBS volumes were provisioned and bound:

```bash
kubectl get pvc -n monitoring
```

Expected output:

```
NAME                                              STATUS   VOLUME   CAPACITY   STORAGECLASS
alertmanager-monitoring-kube-prometheus-alert-0   Bound    pvc-xxx  5Gi        gp3-ebs-csi
grafana                                           Bound    pvc-xxx  5Gi        gp3-ebs-csi
prometheus-monitoring-kube-prometheus-prometh-0   Bound    pvc-xxx  20Gi       gp3-ebs-csi
```

Verify retention is configured correctly inside the Prometheus pod:

```bash
kubectl exec -n monitoring \
  $(kubectl get pod -n monitoring -l app.kubernetes.io/name=prometheus -o name) \
  -- prometheus --version

kubectl get prometheus -n monitoring -o yaml | grep -E 'retention|storage'
```

---

## Step 7 — Access Grafana

**Option A — Port-forward (local access):**

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Open `http://localhost:3000`

**Option B — Ingress (requires NGINX ingress controller + DNS/hosts entry):**

Add to `/etc/hosts` (or your DNS):
```
<ingress-controller-LB-IP>   monitoring.pharma.internal
```

Then open `https://monitoring.pharma.internal`

**Credentials:**
- Username: `admin`
- Password: `pharma-grafana-2026`

---

## Step 8 — Add Your Dashboards (Manual)

No dashboards are pre-loaded. Log in to Grafana and add them yourself:

1. **Dashboards → New → Import**
2. Paste a dashboard JSON or enter a Grafana.com dashboard ID
3. Select **Prometheus** as the data source

Useful dashboard IDs to start with (optional):

| Dashboard | Grafana ID |
|-----------|-----------|
| Node Exporter Full | 1860 |
| Kubernetes Cluster (kube-state-metrics) | 13332 |
| Kubernetes Pod Resources | 6336 |

---

## Step 9 — Add Your Alerts (Manual)

No PrometheusRules are pre-created. To add alerts, create a `PrometheusRule` manifest and
apply it (or let ArgoCD pick it up if you place it in a watched path):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-custom-alerts
  namespace: monitoring
  labels:
    release: monitoring    # must match the Helm release label for Prometheus to pick it up
spec:
  groups:
    - name: example
      rules:
        - alert: PodNotReady
          expr: kube_pod_status_ready{condition="true"} == 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} is not ready"
```

```bash
kubectl apply -f your-alert.yaml
```

Prometheus discovers it automatically because `ruleSelectorNilUsesHelmValues: false` is set
in `prometheus-values.yaml`.

---

## Verify Metrics Are Being Collected

```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
```

Open `http://localhost:9090/targets` — confirm these targets show `UP`:

- `serviceMonitor/monitoring/monitoring-kube-state-metrics` → pod-level metrics
- `serviceMonitor/monitoring/monitoring-prometheus-node-exporter` → node-level metrics
- `serviceMonitor/monitoring/monitoring-kube-prometheus-prometheus` → Prometheus itself

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| PVCs stuck in `Pending` | EBS CSI driver not installed or IAM role missing |
| ArgoCD sync error on StorageClass | Run `kubectl describe storageclass gp3-ebs-csi` |
| Grafana pod not starting | Check PVC is bound: `kubectl get pvc -n monitoring` |
| Prometheus not scraping targets | Verify `serviceMonitorSelectorNilUsesHelmValues: false` is applied |
| `monitoring` app name conflict | Delete old ArgoCD Application first (see Step 3) |

---

## Useful Commands

```bash
# Check Prometheus retention config
kubectl get prometheus -n monitoring monitoring-kube-prometheus-prometheus \
  -o jsonpath='{.spec.retention}'

# Check EBS volumes in AWS
aws ec2 describe-volumes \
  --filters "Name=tag:kubernetes.io/created-for/pvc/namespace,Values=monitoring" \
  --query 'Volumes[*].{ID:VolumeId,Size:Size,Type:VolumeType,State:State}'

# Resize a PVC later (EBS CSI supports online resize — no pod restart needed)
kubectl patch pvc prometheus-monitoring-kube-prometheus-prometh-0 -n monitoring \
  -p '{"spec":{"resources":{"requests":{"storage":"40Gi"}}}}'
```
