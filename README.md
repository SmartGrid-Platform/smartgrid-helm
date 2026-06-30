<div align="center">

# smartgrid-helm

**GitOps delivery repository — SmartGrid Utility Management Platform**

[![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-orange?logo=argo)](https://argo-cd.readthedocs.io/)
[![Helm](https://img.shields.io/badge/Helm-3-0F1689?logo=helm)](https://helm.sh/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS-326CE5?logo=kubernetes)](https://aws.amazon.com/eks/)
[![Prometheus](https://img.shields.io/badge/Monitoring-Prometheus-E6522C?logo=prometheus)](https://prometheus.io/)
[![Grafana](https://img.shields.io/badge/Dashboards-Grafana-F46800?logo=grafana)](https://grafana.com/)

</div>

---

## What Is This Repository?

This repository is the **single source of truth** for all Kubernetes deployments in the SmartGrid Platform. It contains:

- A **Helm chart** (`helm/smartgrid`) that renders all Kubernetes resources for the 6 microservices
- **Two environment value files** (`values.yaml` base + `values-prod.yaml` overrides)
- **ArgoCD Application manifests** that register the chart for continuous GitOps delivery
- **Monitoring stack configuration** (kube-prometheus-stack with Prometheus, Grafana, Alertmanager)

**How it fits in the system:** When a developer pushes code to [smartgrid-app](https://github.com/SmartGrid-Platform/smartgrid-app), the CI pipeline builds Docker images, pushes them to ECR, and commits the new `imageTag` (7-char SHA) to `values.yaml` in this repository. ArgoCD detects that commit, re-renders the Helm chart, and applies the diff to the EKS cluster — zero manual `kubectl apply` or `helm upgrade` commands.

> **Application code:** See [smartgrid-app](https://github.com/SmartGrid-Platform/smartgrid-app).  
> **Infrastructure (Terraform/AWS):** See [smartgrid-infra](https://github.com/SmartGrid-Platform/smartgrid-infra).

---

## Repository Structure

```
smartgrid-helm/
│
├── helm/
│   └── smartgrid/
│       ├── Chart.yaml                  # Chart metadata (version: 1.0.0)
│       ├── values.yaml                 # Base configuration (dev + prod shared defaults)
│       ├── values-prod.yaml            # Production overrides (higher HPA limits)
│       └── templates/
│           ├── deployment.yaml         # One Deployment per service (loop)
│           ├── service.yaml            # One ClusterIP Service per service (loop)
│           ├── ingress.yaml            # Single ALB Ingress, all 9 API path rules
│           ├── hpa.yaml                # HPA for meter-service and billing-service
│           ├── configmap.yaml          # Non-sensitive config (conditional)
│           ├── secrets.yaml            # Sensitive config (conditional, legacy mode)
│           ├── serviceaccount.yaml     # IRSA-annotated ServiceAccount
│           ├── networkpolicy.yaml      # Zero-trust: deny-all + explicit allow rules
│           ├── servicemonitor.yaml     # Prometheus ServiceMonitor per service
│           ├── prometheusrule.yaml     # 15+ PrometheusRule alert definitions
│           ├── grafana-dashboard.yaml  # 3 pre-built Grafana dashboards (ConfigMap)
│           └── db-migrate-job.yaml     # Pre-install/upgrade DB migration + bootstrap Job
│
└── argocd/
    ├── application.yaml                # Dev ArgoCD Application (auto-sync)
    ├── application-prod.yaml           # Production ArgoCD Application (auto-sync)
    └── application-monitoring.yaml     # kube-prometheus-stack ArgoCD Application
```

---

## GitOps Delivery Flow

```
Developer pushes code to smartgrid-app/dev
        │
        ▼
GitHub Actions CI (per-service, parallel)
  Build Docker → Trivy scan → Push to ECR
        │
        ▼
update-helm-values job
  Commits new imageTag (SHA) to helm/smartgrid/values.yaml in this repo
        │
        ▼
ArgoCD detects commit (webhook / polling)
  Fetches chart from smartgrid-helm:dev
  Renders Helm templates (values.yaml + values-prod.yaml)
  Computes diff against live cluster state
        │
        ▼
ArgoCD applies diff to EKS (production namespace)
  Helm hooks execute first: ConfigMap → Secret → DB migration Job
  Deployments rolled out with new image tags
  Liveness + readiness probes confirm health
        │
        ▼
selfHeal=true: ArgoCD continuously reconciles
  Any manual change to EKS is automatically reverted
```

---

## Helm Chart

### Chart Metadata

```yaml
apiVersion: v2
name: smartgrid
description: Helm chart for the SmartGrid Utility Management Platform on AWS EKS
type: application
version: 1.0.0
appVersion: "1.0.0"
```

### Base Values (`values.yaml`)

**Environment & image:**

| Key | Value | Description |
|-----|-------|-------------|
| `environment` | `default` | Used in ECR repo names and resource labels |
| `nodeEnv` | `production` | NODE_ENV for all services |
| `awsRegion` | `ap-south-1` | Injected as AWS_REGION |
| `awsSecretName` | `smartgrid-default/config` | Secrets Manager secret name |
| `imageTag` | `b38ede7` | **Auto-updated by CI** (7-char SHA) |
| `ecrRegistryUrl` | `846511227686.dkr.ecr.ap-south-1.amazonaws.com` | ECR registry |

**Secrets management mode:**

| Key | `true` (legacy) | `false` (ArgoCD / production) |
|-----|----------------|-------------------------------|
| `secrets.create` | Helm creates K8s Secret from values | Bootstrap pre-creates from Secrets Manager |
| `configmap.create` | Helm creates ConfigMap from values | Bootstrap pre-creates from Secrets Manager |

**ServiceAccount (IRSA):**
- Name: `smartgrid-sa`
- Annotation: `eks.amazonaws.com/role-arn: arn:aws:iam::846511227686:role/smartgrid-default-eks-pod-role`

**Services:**

| Service | Port | Replicas | HPA | CPU Req/Lim | Mem Req/Lim |
|---------|------|----------|-----|-------------|-------------|
| auth-service | 3001 | 2 | — | 100m / 500m | 128Mi / 512Mi |
| consumer-service | 3002 | 2 | — | 100m / 500m | 128Mi / 512Mi |
| meter-service | 3003 | 2 | 2–6 (CPU 70%, Mem 80%) | 100m / 500m | 128Mi / 512Mi |
| billing-service | 3004 | 2 | 2–6 (CPU 70%, Mem 80%) | 100m / 500m | 128Mi / 512Mi |
| alert-service | 3005 | 2 | — | 100m / 500m | 128Mi / 512Mi |
| ai-assistant-service | 4004 | 2 | — | 100m / 500m | 128Mi / 512Mi |

**AI Assistant environment variables:**

```yaml
ai-assistant-service:
  env:
    BEDROCK_REGION: "us-east-1"
    BEDROCK_MODEL_PRIMARY: "us.amazon.nova-pro-v1:0"
    BEDROCK_MODEL_FALLBACK: "us.amazon.nova-lite-v1:0"
    CONSUMER_SERVICE_URL: "http://consumer-service:3002"
    BILLING_SERVICE_URL: "http://billing-service:3004"
    METER_SERVICE_URL: "http://meter-service:3003"
```

**Monitoring:**

```yaml
monitoring:
  enabled: true
  scrapeInterval: "30s"   # Balances freshness with cardinality (Prometheus default is 1m)
  scrapeTimeout: "10s"    # < scrapeInterval; allows slow pods without premature timeout
```

### Production Overrides (`values-prod.yaml`)

```yaml
secrets:
  create: false     # Pre-created by bootstrap from Secrets Manager
configmap:
  create: false     # Pre-created by bootstrap from Secrets Manager

services:
  meter-service:
    hpa:
      maxReplicas: 8   # Increased from 6 for production peak load
  billing-service:
    hpa:
      maxReplicas: 8   # Increased from 6 for production peak load
```

---

## Helm Templates

### deployment.yaml

Generates one `Deployment` per entry in `values.services` using a range loop.

**Security context (all pods):**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
```

**Image format:**
```
<ecrRegistryUrl>/smartgrid-<environment>-<service.image>:<imageTag>
# Example:
846511227686.dkr.ecr.ap-south-1.amazonaws.com/smartgrid-default-auth-service:b38ede7
```

**Pull policy:** `Always` — forces ECR pull on every pod start.

**Environment injection:**
- `envFrom.configMapRef: smartgrid-config` — non-sensitive config
- `envFrom.secretRef: smartgrid-secrets` — sensitive config (DB password, JWT secret)
- `env.PORT` — set explicitly per service

**Health probes:**

| Probe | Path | Initial Delay | Period | Failure Threshold |
|-------|------|--------------|--------|-------------------|
| Liveness | `/healthz` | 20s | 15s | 3 |
| Readiness | `/ready` | 15s | 10s | 3 |

AI assistant uses `/api/assistant/healthz` and `/api/assistant/ready`.

**Base replica count:** 12 pods total (2 per service).

---

### service.yaml

One `ClusterIP` Service per microservice. Internal-only — external traffic enters via the ALB Ingress.

**Kubernetes DNS names (within cluster):**

| Service | DNS |
|---------|-----|
| auth-service | `auth-service.<namespace>.svc.cluster.local:3001` |
| consumer-service | `consumer-service.<namespace>.svc.cluster.local:3002` |
| meter-service | `meter-service.<namespace>.svc.cluster.local:3003` |
| billing-service | `billing-service.<namespace>.svc.cluster.local:3004` |
| alert-service | `alert-service.<namespace>.svc.cluster.local:3005` |
| ai-assistant-service | `ai-assistant-service.<namespace>.svc.cluster.local:4004` |

---

### ingress.yaml

Single `Ingress` resource — AWS Load Balancer Controller provisions one ALB.

**ALB annotations:**
```yaml
alb.ingress.kubernetes.io/scheme: internet-facing
alb.ingress.kubernetes.io/target-type: ip          # Direct pod IP routing
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
alb.ingress.kubernetes.io/load-balancer-name: smartgrid-default-eks-alb
alb.ingress.kubernetes.io/group.name: smartgrid    # Shares ALB with Grafana (cost)
alb.ingress.kubernetes.io/healthcheck-path: /health
alb.ingress.kubernetes.io/success-codes: "200"
```

**Routing rules:**

| Path | Backend Service | Port |
|------|----------------|------|
| `/api/auth` | auth-service | 3001 |
| `/api/consumers` | consumer-service | 3002 |
| `/api/meters` | meter-service | 3003 |
| `/api/bills` | billing-service | 3004 |
| `/api/tariffs` | billing-service | 3004 |
| `/api/recharges` | billing-service | 3004 |
| `/api/alerts` | alert-service | 3005 |
| `/api/inspections` | alert-service | 3005 |
| `/api/assistant` | ai-assistant-service | 4004 |

---

### hpa.yaml

Created only for services that have an `hpa:` block in values.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          averageUtilization: 70   # Scale up when CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          averageUtilization: 80   # Scale up when memory > 80%
```

**Scaling matrix:**

| Service | Env | Min | Max | Scale Trigger |
|---------|-----|-----|-----|---------------|
| meter-service | dev | 2 | 6 | CPU > 70% OR Mem > 80% |
| billing-service | dev | 2 | 6 | CPU > 70% OR Mem > 80% |
| meter-service | prod | 2 | **8** | CPU > 70% OR Mem > 80% |
| billing-service | prod | 2 | **8** | CPU > 70% OR Mem > 80% |

---

### configmap.yaml & secrets.yaml

Both use Helm hooks to run before Deployments:

```yaml
annotations:
  helm.sh/hook: pre-install,pre-upgrade
  helm.sh/hook-weight: "-10"          # Runs before DB migration Job (-5)
  helm.sh/hook-delete-policy: before-hook-creation
```

**ConfigMap (`smartgrid-config`) data keys:**
`NODE_ENV`, `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PORT`, `AWS_REGION`, `AWS_SECRET_NAME`

**Secret (`smartgrid-secrets`) data keys (base64-encoded):**
`DB_PASSWORD`, `JWT_SECRET`, `ADMIN_NAME`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`

In production (`secrets.create=false`), these resources are pre-created by the bootstrap script from AWS Secrets Manager and skipped by Helm.

---

### serviceaccount.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: smartgrid-sa
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::846511227686:role/smartgrid-default-eks-pod-role"
```

All 6 Deployments reference `serviceAccountName: smartgrid-sa`. The IRSA annotation allows pods to call AWS APIs (Secrets Manager, S3, SNS, Bedrock) without any static credentials.

---

### networkpolicy.yaml

Three `NetworkPolicy` resources implement **zero-trust networking**:

**1. Default deny-all:**
```yaml
podSelector: {}      # Applies to ALL pods in namespace
policyTypes: [Ingress, Egress]
# No rules = deny everything
```

**2. Allow inter-service + ALB ingress:**
- **Ingress:** From pods with matching `environment` label + from ALB on service ports (3001–3005, 4004)
- **Egress:** DNS (UDP/TCP 53) + pods with matching label + AWS services (HTTPS 443: Secrets Manager, S3, SQS, Bedrock) + RDS (TCP 3306)

**3. The effect:** Services can communicate with each other and with AWS managed services. They cannot reach arbitrary internet endpoints. Any attempt to exfiltrate data to an unknown host is blocked at the network level.

---

### servicemonitor.yaml

One `ServiceMonitor` per service. Labels match the Prometheus Operator selector (`release: kube-prometheus-stack`).

```yaml
endpoints:
  - port: http
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
    scheme: http
```

All 6 services must expose a `/metrics` endpoint (prom-client instrumentation) for Prometheus to scrape.

---

### prometheusrule.yaml

15+ alert rules across 5 groups:

**Group 1: Availability (interval: 30s)**

| Alert | Expression | Severity | For |
|-------|-----------|----------|-----|
| `SmartGridServiceDown` | `up == 0` | critical | 1m |
| `SmartGridPodCrashLooping` | restart count > 0 in 15m | critical | 0m |
| `SmartGridDeploymentReplicasMismatch` | spec replicas ≠ ready replicas | warning | 5m |

**Group 2: HTTP Error Rates (interval: 30s)**

| Alert | Threshold | Severity | For |
|-------|-----------|----------|-----|
| `SmartGridHighErrorRate` | 5xx rate > 1% | warning | 2m |
| `SmartGridAuthFailureSpike` | Login failures > 2/min | warning | 2m |
| `SmartGridBillingFailures` | Billing errors > 0 | warning | 2m |

**Group 3: Latency (interval: 60s)**

| Alert | Threshold | Severity | For |
|-------|-----------|----------|-----|
| `SmartGridHighP95Latency` | P95 HTTP duration > 500ms | warning | 3m |
| `SmartGridAIResponseSlow` | AI P95 latency > 10s | warning | 2m |

**Group 4: Resource Utilisation (interval: 30s)**

| Alert | Threshold | Severity | For |
|-------|-----------|----------|-----|
| `SmartGridHighCPU` | Node CPU > 30% | warning | 2m |
| `SmartGridHighMemory` | Node memory > 40% | warning | 2m |
| `SmartGridPodMemoryNearLimit` | Pod memory > 50% of 512Mi limit | warning | 3m |

> Thresholds are intentionally low for t3.small SPOT nodes. Raise for production-grade nodes.

**Group 5: Notifications (interval: 60s)**

| Alert | Threshold | Severity | For |
|-------|-----------|----------|-----|
| `SmartGridNotificationFailures` | Email errors > 0 | warning | 5m |

All alerts include `team: smartgrid` label for Alertmanager routing to Slack/email.

---

### grafana-dashboard.yaml

Three Grafana dashboards auto-imported via ConfigMap label `grafana_dashboard: "1"` (Grafana sidecar).

**Dashboard 1: Application Overview** (30s refresh)
- Total request rate across all services (req/s)
- Overall 5xx error rate
- P95 response time
- Running pod count
- Auth login failure rate (last 5m)
- Bills generated (last 1h)
- Request rate per service (timeseries)
- Error rate per service (timeseries)
- Node.js process memory per service (timeseries)
- AI Bedrock P95 latency (timeseries)

**Dashboard 2: Infrastructure Health** (60s refresh)
- Node CPU usage % (gauge)
- Node memory usage % (gauge)
- Pod restart count (last 15m)
- EKS node count
- CPU per node (timeseries)
- Memory per node (timeseries)
- Pod memory vs limit (timeseries)

**Dashboard 3: API Performance** (30s refresh)
- P50/P95/P99 latency per service
- 5xx errors per service (red)
- 4xx client errors per service (orange)
- Top routes by request count (table)
- Slowest routes by P95 (table)
- Meter readings submitted/s
- Bills generated/min
- AI fallback to Nova Lite count/hr

---

### db-migrate-job.yaml

Kubernetes `Job` that runs database migrations before every Helm install or upgrade.

```yaml
annotations:
  helm.sh/hook: pre-install,pre-upgrade
  helm.sh/hook-weight: "-5"                     # After ConfigMap/Secret (-10), before Deployments
  helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      serviceAccountName: smartgrid-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - image: "<ecr>/smartgrid-<env>-auth-service:<imageTag>"
          workingDir: /app/shared/database
          command: ["/bin/sh", "-c"]
          args: ["npm run migrate && npm run bootstrap"]
```

**What it does:**
1. `npm run migrate` — Sequelize sync: creates or alters all 9 database tables
2. `npm run bootstrap` — Seeds the initial admin user (idempotent; skips if already exists)

The Job uses the auth-service container image because it bundles the `shared/database` module. It's deleted after successful completion.

---

## ArgoCD Applications

### `application.yaml` — Dev

```yaml
source:
  repoURL: https://github.com/SmartGrid-Platform/smartgrid-helm.git
  targetRevision: dev
  path: helm/smartgrid
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
    - ApplyOutOfSyncOnly=true
```

### `application-prod.yaml` — Production

```yaml
source:
  targetRevision: dev         # Same branch; production uses values-prod.yaml overlay
  helm:
    valueFiles:
      - values.yaml
      - values-prod.yaml
destination:
  namespace: production
syncPolicy:
  automated:
    prune: false
    selfHeal: true
  syncOptions:
    - RespectIgnoreDifferences=true
ignoreDifferences:
  - kind: Secret
    name: smartgrid-secrets
    jsonPointers: [/data]     # Managed externally by bootstrap
  - kind: ConfigMap
    name: smartgrid-config
    jsonPointers: [/data]     # Managed externally by bootstrap
```

**`prune: false` rationale:** In production, resources are never automatically deleted when removed from Git — requires explicit human action to prevent accidental service removal.

---

### `application-monitoring.yaml` — kube-prometheus-stack

Deploys the full Prometheus + Grafana + Alertmanager stack from the Prometheus Community Helm chart.

```yaml
source:
  repoURL: https://prometheus-community.github.io/helm-charts
  chart: kube-prometheus-stack
  targetRevision: ">=61.0.0"
destination:
  namespace: monitoring
```

**Key configuration decisions:**

| Component | Configuration | Rationale |
|-----------|--------------|-----------|
| Prometheus retention | 15 days / 8 GB | Balances observability window with t3.small disk |
| Prometheus storage | `emptyDir` | No PVC (data lost on pod restart); switch to EBS CSI PVC for production |
| Grafana version | `10.4.7` (pinned) | 11.x breaks with EKS RBAC app-platform API |
| Grafana dashboard sidecar | `searchNamespace: ALL` | Discovers dashboard ConfigMaps from all namespaces |
| Alertmanager SMTP | Gmail, port 587, TLS | Configured via mounted secret (not values) |
| ServiceMonitor selector | `release: kube-prometheus-stack` | Must match label on ServiceMonitor resources |
| node-exporter | Enabled | Node-level CPU/memory/disk metrics |
| kube-state-metrics | Enabled | Kubernetes object state metrics |

**Alertmanager routing:**

| Match | Receiver | Repeat |
|-------|----------|--------|
| `team: smartgrid` + `severity: critical` | `smartgrid-slack` | 15 min |
| `team: smartgrid` + `severity: warning` | `smartgrid-slack` | 30 min |
| Other | `null` (discard) | — |

Slack messages include alert name, severity, summary, description, and runbook. Email (asadchamp109@gmail.com) receives HTML-formatted alerts. Both channels configured via mounted Kubernetes secrets.

---

## Deployment Resource Summary

| Resource Type | Count | Notes |
|--------------|-------|-------|
| Deployments | 6 | One per service, 2 replicas minimum |
| Services (ClusterIP) | 6 | Internal-only |
| Ingress | 1 | Single ALB, 9 path rules |
| HPA | 2 | meter-service, billing-service |
| ServiceAccount | 1 | IRSA-annotated (`smartgrid-sa`) |
| NetworkPolicy | 3 | Deny-all + inter-service allow + ALB allow |
| ServiceMonitor | 6 | Prometheus scraping at 30s |
| PrometheusRule | 1 | 15+ alert rules in 5 groups |
| ConfigMap | 4 | `smartgrid-config` + 3 Grafana dashboards |
| Secret | 1 | `smartgrid-secrets` (legacy / dev mode) |
| Job | 1 | DB migration (pre-install/upgrade) |

**Total pods at base:** 12 (2 per service)  
**Total pods at production max:** 28 (meter + billing scale to 8 each)

---

## Manual Operations

### Force an ArgoCD sync

```bash
argocd app sync smartgrid-production
argocd app sync kube-prometheus-stack
```

### Roll back to a previous image

```bash
# Edit values.yaml imageTag to the previous SHA
git commit -m "revert: roll back to previous image"
git push
# ArgoCD auto-syncs within 3 minutes
```

### Check application health

```bash
argocd app get smartgrid-production
kubectl get pods -n production
kubectl get hpa -n production
kubectl get ingress -n production
```

### View Prometheus alerts

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Open http://localhost:9090/alerts
```

### Access Grafana

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# Open http://localhost:3000
```

Or via the ALB Ingress: `http://<alb-dns>/grafana`

---

## Related Repositories

| Repository | Purpose |
|------------|---------|
| [smartgrid-app](https://github.com/SmartGrid-Platform/smartgrid-app) | Application source — React frontend, 6 microservices, Lambda functions |
| [smartgrid-infra](https://github.com/SmartGrid-Platform/smartgrid-infra) | Terraform — full AWS infrastructure, CI/CD for infra, cluster bootstrap |
