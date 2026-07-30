# YouKnowIt — GitOps

ArgoCD-based GitOps manifests for the YouKnowIt portfolio project. This repository is the single source of truth for everything running inside the EKS cluster — from the application itself to the full observability stack.

**Repositories:** [YouKnowIt-app](https://github.com/Lielik/YouKnowIt-app) · [YouKnowIt-infra](https://github.com/Lielik/YouKnowIt-infra) · [YouKnowIt-gitops](https://github.com/Lielik/YouKnowIt-gitops)

---

## How it works

Terraform (in `YouKnowIt-infra`) installs ArgoCD via Helm and applies the `app-of-apps` manifest from this repo. From that point on, ArgoCD watches this repository and continuously reconciles the cluster toward the state declared here. No `kubectl apply` commands are run manually after the initial bootstrap.

Every push to `main` is picked up by ArgoCD within its next poll cycle (60 seconds). Image tag updates are automated — the CI/CD pipeline in `YouKnowIt-app` directly commits a new `tag:` value to `charts/youknowit/values.yaml` after a successful build.

---

## Repository structure

```
argocd/
├── bootstrap/
│   └── argocd-values.yaml      # ArgoCD Helm values (fetched by Terraform at apply time)
└── apps/
    ├── app-of-apps.yaml        # Root Application — watches this directory
    ├── youknowit.yaml          # The flashcard app
    ├── ingress-nginx.yaml      # NLB ingress controller
    ├── cert-manager.yaml       # TLS certificate automation
    ├── external-secrets.yaml   # ESO — pulls secrets from AWS Secrets Manager
    ├── prometheus-stack.yaml   # Prometheus, Grafana, Alertmanager
    ├── loki-app.yaml           # Loki log aggregation (S3-backed)
    ├── promtail.yaml           # Log shipper (DaemonSet, one pod per node)
    ├── monitoring-config.yaml  # Raw manifests for monitoring namespace
    ├── logging-config.yaml     # Raw manifests for logging namespace
    └── storage-config.yaml     # StorageClass for EBS CSI driver

charts/
└── youknowit/
    ├── Chart.yaml
    ├── values.yaml             # Image tag updated automatically by CI/CD
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        ├── ingress.yaml
        ├── configmap.yaml
        ├── serviceaccount.yaml
        ├── externalsecret.yaml
        ├── clustersecretstore.yaml
        ├── clusterissuer.yaml
        └── servicemonitor.yaml

platform/
├── monitoring/
│   ├── grafana-external-secret.yaml      # ESO → Kubernetes Secret for Grafana admin
│   ├── alertmanager-external-secret.yaml # ESO → Kubernetes Secret for Slack webhook
│   └── loki-datasource.yaml             # Grafana datasource ConfigMap for Loki
├── logging/
│   └── loki-serviceaccount.yaml         # Loki ServiceAccount with IRSA annotation
└── storage/
    └── storageclass.yaml                 # ebs-csi-gp3 StorageClass (cluster default)
```

---

## App-of-Apps pattern

`argocd/apps/app-of-apps.yaml` is the entry point. It points ArgoCD at the `argocd/apps/` directory and watches it for `Application` manifests. Any new `.yaml` file dropped into that directory is automatically discovered and deployed. Removing a file prunes the Application and its resources from the cluster.

```
app-of-apps (watches argocd/apps/)
├── youknowit          → charts/youknowit/
├── ingress-nginx      → Helm chart (kubernetes.github.io/ingress-nginx)
├── cert-manager       → Helm chart (charts.jetstack.io)
├── external-secrets   → Helm chart (charts.external-secrets.io)
├── prometheus-stack   → Helm chart (prometheus-community.github.io/helm-charts)
├── loki               → Helm chart (grafana.github.io/helm-charts)
├── promtail           → Helm chart (grafana.github.io/helm-charts)
├── monitoring-config  → platform/monitoring/
├── logging-config     → platform/logging/
└── storage-config     → platform/storage/
```

---

## Deployed Applications

### youknowit

The flashcard application. Deployed from the local `charts/youknowit/` Helm chart in this repo.

The image tag in `values.yaml` is updated automatically after each successful push to the `main` branch of `YouKnowIt-app` — the CI/CD pipeline commits the new tag directly to this file.

Key resources created by the chart:

| Resource | Description |
|---|---|
| `Deployment` | 2 replicas, liveness + readiness probes on `/health` |
| `Service` | ClusterIP on port 80, targeting port 8000 |
| `Ingress` | Routes `youknowit.ddns.net` to the Service via ingress-nginx |
| `ConfigMap` | Non-secret environment variables |
| `ServiceAccount` | Used by the app pod |
| `ExternalSecret` | Pulls `SECRET_KEY` and `DATABASE_URL` from Secrets Manager |
| `ClusterSecretStore` | ESO store — authenticates to AWS via ESO's IRSA role |
| `ClusterIssuer` | Let's Encrypt ACME issuer (staging by default) |
| `ServiceMonitor` | Tells Prometheus to scrape `/metrics` every 30 seconds |

### ingress-nginx

NGINX ingress controller provisioned with an AWS NLB (Network Load Balancer) via the `service.beta.kubernetes.io/aws-load-balancer-type: nlb` annotation. NLB rather than ALB is the correct pairing for the NGINX ingress controller.

Chart version: `4.15.1`

### cert-manager

Automates TLS certificate issuance and renewal via Let's Encrypt. The `ClusterIssuer` resource (created by the `youknowit` chart) uses an HTTP-01 challenge over ingress-nginx.

The `ClusterIssuer` currently points at the **Let's Encrypt staging** endpoint. To switch to production, change two values in `charts/youknowit/values.yaml`:

```yaml
certManager:
  clusterIssuer: letsencrypt-prod
  acmeServer: https://acme-v02.api.letsencrypt.org/directory
```

Chart version: `v1.20.2`

### external-secrets

External Secrets Operator — bridges AWS Secrets Manager and Kubernetes Secrets. ESO's controller ServiceAccount is annotated with an IAM role ARN (IRSA) scoped to read secrets under the `youknowit/` prefix.

The `ClusterSecretStore` resource (in the `youknowit` chart) references ESO's own ServiceAccount in the `external-secrets` namespace. This single store serves `ExternalSecret` resources across all namespaces — the app, monitoring, and logging namespaces all consume from it.

`ServerSideApply=true` is required in the sync options because ESO's CRD schemas exceed Kubernetes' 256KiB annotation limit for client-side apply.

Chart version: `2.6.0` (requires `external-secrets.io/v1` API — `v1beta1` was dropped in ESO v0.17.0)

### prometheus-stack

`kube-prometheus-stack` — Prometheus, Grafana, and Alertmanager as a single chart.

- **Grafana** admin credentials are pulled from Secrets Manager via ESO (`youknowit/grafana/admin`). Persistence uses the `ebs-csi-gp3` StorageClass with a 5Gi volume.
- **Prometheus** retains 7 days of metrics. The `youknowit` app's `/metrics` endpoint is scraped via the `ServiceMonitor` resource in the app chart.
- **Alertmanager** routes alerts to Slack. The webhook URL is mounted as a file from a Kubernetes Secret (`alertmanager-slack-secret`, populated by ESO from `youknowit/alertmanager/slack`). Watchdog alerts are routed to a `null` receiver to suppress them.

`ServerSideApply=true` is required for the same CRD annotation-size reason as ESO.

Chart version: `86.3.2`

### loki

Grafana Loki in `SingleBinary` deployment mode — a single pod suitable for dev/portfolio workloads. Logs are stored in S3 (`youknowit-loki-logs`), authenticated via IRSA.

The `loki` ServiceAccount is not created by the chart (`serviceAccount.create: false`) — it is created separately in `platform/logging/loki-serviceaccount.yaml` with the IRSA annotation baked in.

Schema: `tsdb` / `v13` / `s3` — the current recommended configuration per Grafana's own deployment guides.

Chart version: `7.0.0`

### promtail

DaemonSet — one pod per node — that tails container log files from `/var/log/pods/` and ships them to Loki at `http://loki.logging.svc.cluster.local:3100`.

Chart version: `6.17.1`

### monitoring-config / logging-config / storage-config

Three Applications that sync raw Kubernetes manifests from the `platform/` directory:

- `monitoring-config` — Grafana and Alertmanager `ExternalSecret` resources, plus the Loki datasource `ConfigMap` that Grafana's sidecar picks up automatically.
- `logging-config` — Loki's `ServiceAccount` with its IRSA role annotation.
- `storage-config` — the `ebs-csi-gp3` StorageClass, set as the cluster default.

---

## Secrets architecture

No secrets are stored in this repository. The flow for every secret is:

```
AWS Secrets Manager → ESO (via IRSA) → Kubernetes Secret → Pod env var or mounted file
```

| Secrets Manager path | Consumed by | Delivered as |
|---|---|---|
| `youknowit/dev/app` | `youknowit` app | `SECRET_KEY` env var |
| `youknowit/dev/database-url` | `youknowit` app | `DATABASE_URL` env var |
| `youknowit/grafana/admin` | Grafana | `grafana-admin-secret` Kubernetes Secret |
| `youknowit/alertmanager/slack` | Alertmanager | File at `/etc/alertmanager/secrets/alertmanager-slack-secret/webhook-url` |

---

## Accessing cluster tools

ArgoCD has no public ingress — it is accessed only via port-forward:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080
# Username: admin
# Password: kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d
```

Same pattern for Prometheus and Grafana:

```bash
kubectl port-forward svc/prometheus-stack-kube-prom-prometheus -n monitoring 9090:9090
kubectl port-forward svc/prometheus-stack-grafana -n monitoring 3000:80
kubectl port-forward svc/prometheus-stack-kube-prom-alertmanager -n monitoring 9093:9093
```

---

## Design decisions

**App-of-Apps over ApplicationSets.** A flat directory of `Application` manifests is easier to reason about and debug than a generator-based approach for a project of this size.

**Platform manifests separate from Helm charts.** Resources that configure other charts (ESO `ExternalSecret` for Grafana credentials, the Loki `ServiceAccount` with its IRSA annotation) live in `platform/` as raw YAML rather than being folded into the corresponding chart's values. This avoids cross-chart coupling — the `prometheus-stack` chart doesn't need to know about ESO, and the Loki chart doesn't need to know about IAM.

**ESO's controller ServiceAccount holds the IRSA role, not per-app ServiceAccounts.** A single IAM role scoped to `system:serviceaccount:external-secrets:external-secrets` serves all `ExternalSecret` resources across all namespaces. This is correct — the ESO controller does the actual AWS API calls on behalf of every secret it manages, regardless of which namespace the `ExternalSecret` lives in.

**IAM role ARNs are hardcoded.** Since IAM role names are chosen in Terraform (e.g. `youknowit-dev-eso`) and the AWS account ID is static, the full ARN is deterministic before apply. Hardcoding it avoids the complexity of Terraform-writes-to-Git automation while preserving a single source of truth for the name.

**Let's Encrypt staging by default.** The `ClusterIssuer` uses the staging ACME endpoint to avoid rate limit issues during development. Switching to production requires changing two values in `charts/youknowit/values.yaml`.

**`ServerSideApply=true` for ESO and Prometheus.** Both charts include CRDs whose OpenAPI schemas exceed Kubernetes' 256KiB annotation limit for client-side apply. Server-side apply bypasses this by tracking field ownership in etcd rather than in annotations.