# 🔄 ArgoCD — Interview Q&A (12 Scenarios)

> GitOps, Sync, App of Apps, Rollback, RBAC, Multi-cluster

---

## ARGO-Q1. "ArgoCD architecture explain karo."

```
ArgoCD Components
├── API Server          — gRPC/REST API, UI backend, CLI
├── Repository Server   — Git repo clone + manifest generate
├── Application Controller — Cluster state monitor + sync
├── Redis               — Cache
├── Dex (optional)      — SSO/OIDC provider
└── ApplicationSet Controller — Multiple apps generate

GitOps Loop:
Git Repo → ArgoCD watches → Diff detect → Sync → Kubernetes
     ↑________________________________|
     (Manual change → auto revert if selfHeal=true)
```

```bash
# ArgoCD install
kubectl create namespace argocd
kubectl apply -n argocd -f \
    https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# UI access
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Initial password
kubectl get secret argocd-initial-admin-secret \
    -n argocd -o jsonpath='{.data.password}' | base64 -d

# CLI login
argocd login localhost:8080 --username admin --password <pwd> --insecure
```

---

## ARGO-Q2. "Application create karo — Helm chart deploy."

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-production
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io   # Delete pe resources bhi delete
spec:
  project: production
  
  source:
    repoURL: https://github.com/my-org/helm-charts
    targetRevision: v1.2.3    # Tag, branch, or commit SHA
    path: charts/my-app
    
    helm:
      valueFiles:
      - values.yaml
      - values-production.yaml
      set:
      - name: image.tag
        value: "abc1234"
      - name: replicas
        value: "3"
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false    # Empty manifest se sync mat karo
    syncOptions:
    - Validate=true
    - CreateNamespace=true
    - PruneLast=true       # Resources baad mein prune karo
    - ApplyOutOfSyncOnly=true  # Sirf changed resources apply
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 1m
```

```bash
# CLI se create
argocd app create my-app \
    --repo https://github.com/my-org/k8s-manifests \
    --path apps/production \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace production \
    --sync-policy automated \
    --auto-prune \
    --self-heal

# Status check
argocd app get my-app-production
argocd app sync my-app-production
argocd app wait my-app-production --health
```

---

## ARGO-Q3. "App of Apps Pattern — Multiple applications manage karo."

```yaml
# Root App — baaki sab apps manage karta hai
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/my-org/k8s-manifests
    path: argocd/apps    # Yahan aur ArgoCD Application yamls hain
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

# argocd/apps/ folder structure:
# ├── frontend-app.yaml
# ├── backend-app.yaml
# ├── postgres-app.yaml
# └── monitoring-app.yaml
```

```
Git Repo Structure (recommended):
├── apps/
│   ├── base/              (shared manifests)
│   ├── overlays/
│   │   ├── staging/
│   │   └── production/
├── argocd/
│   ├── apps/              (Application CRDs — App of Apps)
│   └── projects/          (AppProject CRDs)
└── helm-charts/
    └── my-app/
```

---

## ARGO-Q4. "ApplicationSet — Multiple clusters pe same app deploy karo."

```yaml
# ApplicationSet — Dynamically applications generate karo
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-all-clusters
  namespace: argocd
spec:
  generators:
  # List Generator — Fixed list
  - list:
      elements:
      - cluster: staging
        url: https://staging.k8s.example.com
        env: staging
        replicas: "2"
      - cluster: production
        url: https://production.k8s.example.com
        env: production
        replicas: "5"
  
  # Git Generator — Git se automatically detect
  - git:
      repoURL: https://github.com/my-org/k8s-manifests
      revision: HEAD
      directories:
      - path: clusters/*   # clusters/staging, clusters/production etc.
  
  template:
    metadata:
      name: '{{cluster}}-my-app'
    spec:
      source:
        repoURL: https://github.com/my-org/helm-charts
        path: charts/my-app
        helm:
          values: |
            replicas: {{replicas}}
            env: {{env}}
      destination:
        server: '{{url}}'
        namespace: my-app
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## ARGO-Q5. "Sync Waves aur Sync Hooks — Deployment order control."

```yaml
# Sync Wave — Resources ka order control karo
# Wave 0 → Wave 1 → Wave 2 (aagey badhta hai)

# Database pehle (wave -1)
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
  annotations:
    argocd.argoproj.io/sync-wave: "-1"

---
# Migration job (wave 0)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/sync-wave: "0"

---
# App deployment baad mein (wave 1)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    argocd.argoproj.io/sync-wave: "1"

---
# Sync Hook — Sync lifecycle pe kuch karo
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  annotations:
    argocd.argoproj.io/hook: PostSync        # Sync ke baad chalao
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
    # PreSync, Sync, PostSync, SyncFail, Skip
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: smoke-test
        image: curlimages/curl
        command: ["curl", "-f", "http://my-app/health"]
```

---

## ARGO-Q6. "Rollback karo aur History manage karo."

```bash
# Application history dekho
argocd app history my-app-production

# Specific revision pe rollback
argocd app rollback my-app-production 5   # Revision 5 pe

# Ya CLI se
argocd app set my-app-production \
    --revision abc1234git  # Specific commit

# History limit set karo
argocd app set my-app-production \
    --revision-history-limit 10

# Force sync (kuch stuck ho to)
argocd app sync my-app-production --force

# Hard refresh (Git se fresh fetch)
argocd app get my-app-production --hard-refresh
```

```yaml
# Application mein history limit
spec:
  revisionHistoryLimit: 10   # Default 10
```

---

## ARGO-Q7. "Projects — Multi-team RBAC aur isolation."

```yaml
# AppProject — Team isolation
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-alpha
  namespace: argocd
spec:
  description: "Team Alpha's applications"
  
  # Allowed source repos
  sourceRepos:
  - https://github.com/my-org/team-alpha-*
  
  # Allowed destinations
  destinations:
  - namespace: team-alpha-*
    server: https://kubernetes.default.svc
  
  # Cluster resource access (CRDs, Namespaces etc.)
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  
  # Namespace resource blacklist
  namespaceResourceBlacklist:
  - group: ''
    kind: ResourceQuota   # Team ResourceQuota change nahi kar sakti
  
  # Roles
  roles:
  - name: developer
    description: "Deploy to staging only"
    policies:
    - p, proj:team-alpha:developer, applications, sync, team-alpha/*, allow
    - p, proj:team-alpha:developer, applications, get, team-alpha/*, allow
    - p, proj:team-alpha:developer, applications, delete, team-alpha/production-*, deny
    groups:
    - team-alpha-developers    # GitHub/LDAP group
  
  - name: admin
    policies:
    - p, proj:team-alpha:admin, applications, *, team-alpha/*, allow
    groups:
    - team-alpha-leads
```

---

## ARGO-Q8. "Secrets kaise manage karo ArgoCD ke saath — Sealed Secrets / ESO."

```bash
# Option 1: Sealed Secrets
# Secret encrypt karke Git mein safe rakhna

# Install kubeseal
kubeseal --fetch-cert \
    --controller-name=sealed-secrets \
    --controller-namespace=kube-system > pub-cert.pem

# Secret seal karo
kubectl create secret generic db-secret \
    --from-literal=password=mysecret123 \
    --dry-run=client -o yaml \
    | kubeseal --cert pub-cert.pem --format yaml > db-sealed-secret.yaml

# Yeh sealed-secret.yaml safely Git mein commit karo!
# ArgoCD apply karega → Sealed Secret controller decrypt karega → Real Secret bane
```

```yaml
# Option 2: External Secrets + ArgoCD
# ArgoCD sirf ExternalSecret CRD deploy karta hai
# ESO automatically AWS Secrets Manager se sync karta hai

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-store
    kind: ClusterSecretStore
  target:
    name: db-secret
  data:
  - secretKey: password
    remoteRef:
      key: production/db
      property: password
```

---

## ARGO-Q9. "Multi-Cluster setup — ArgoCD se multiple clusters manage karo."

```bash
# Cluster add karo
argocd cluster add production-eks \
    --kubeconfig ~/.kube/production-config \
    --name production

argocd cluster add staging-eks \
    --kubeconfig ~/.kube/staging-config \
    --name staging

# Clusters list
argocd cluster list

# Application specific cluster pe
argocd app create backend-production \
    --dest-server https://production-eks.example.com \
    --dest-namespace production

# EKS ke liye — IRSA use karo
# ArgoCD ServiceAccount ko IAM Role do
# Role ko production cluster ka kubeconfig access do
```

---

## ARGO-Q10. "ArgoCD notifications — Slack, Email alerts."

```yaml
# Notifications Controller
# argocd-notifications-cm ConfigMap

apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # Slack integration
  service.slack: |
    token: $slack-token
  
  # Templates
  template.app-sync-succeeded: |
    slack:
      attachments: |
        [{
          "title": "✅ {{.app.metadata.name}} Deployed",
          "color": "good",
          "fields": [{
            "title": "Revision",
            "value": "{{.app.status.sync.revision | truncate 7 \"\"}}",
            "short": true
          }]
        }]
  
  template.app-sync-failed: |
    slack:
      attachments: |
        [{
          "title": "❌ {{.app.metadata.name}} Sync Failed",
          "color": "danger"
        }]
  
  # Triggers
  trigger.on-sync-succeeded: |
    - when: app.status.sync.status == 'Synced' && app.status.health.status == 'Healthy'
      send: [app-sync-succeeded]
  
  trigger.on-sync-failed: |
    - when: app.status.sync.status == 'OutOfSync'
      send: [app-sync-failed]

---
# Application annotation mein enable karo
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: "deployments"
    notifications.argoproj.io/subscribe.on-sync-failed.slack: "alerts"
```

---

## ARGO-Q11. "Image Updater — Automatic image tag update."

```yaml
# ArgoCD Image Updater — New image push hote hi update karo
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  annotations:
    # Image update policy
    argocd-image-updater.argoproj.io/image-list: my-app=my-registry/my-app
    argocd-image-updater.argoproj.io/my-app.update-strategy: semver
    # Strategies: semver, latest, digest, name
    argocd-image-updater.argoproj.io/my-app.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    
    # Git write back — Git mein commit karo (GitOps!)
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
```

---

## ARGO-Q12. "ArgoCD Troubleshooting — Common issues debug karo."

```bash
# App OutOfSync — kyon?
argocd app diff my-app    # Git vs Cluster diff dekho
argocd app get my-app --show-operation

# Sync Failed
argocd app get my-app | grep "Message\|Phase"

# Controller logs
kubectl logs -n argocd \
    -l app.kubernetes.io/name=argocd-application-controller \
    --tail 100

# Repo server logs (Git clone issues)
kubectl logs -n argocd \
    -l app.kubernetes.io/name=argocd-repo-server \
    --tail 100

# Common issues:
# 1. Sync loop — selfHeal + manual change conflict
argocd app set my-app --sync-policy none   # Temporarily disable

# 2. "Unable to connect to repository"
argocd repo list    # Repo credentials check
argocd repo get https://github.com/my-org/repo

# 3. "Namespace not found"
argocd app set my-app --sync-option CreateNamespace=true

# 4. CRD missing
# CRD pehle apply karo (sync wave -1 pe rakhna)

# Health check fail — custom health check
kubectl get application my-app -n argocd -o yaml | grep -A 10 "health"
```

---
---

# 📊 Observability — Interview Q&A (12 Scenarios)

> Prometheus, Grafana, Loki, Alerting, Tracing, OpenTelemetry

---

## OBS-Q1. "Observability ke teen pillars kya hain?"

```
THREE PILLARS OF OBSERVABILITY

1. METRICS (Numbers over time)
   ├── What: CPU %, error rate, request count
   ├── Tool: Prometheus + Grafana
   └── Format: Time series data

2. LOGS (Events records)
   ├── What: Error messages, request details, audit
   ├── Tool: Loki + Grafana (ya ELK Stack)
   └── Format: Structured/Unstructured text

3. TRACES (Request journey)
   ├── What: Request ka path across services
   ├── Tool: Jaeger, Tempo, Zipkin
   └── Format: Spans with timing

The Four Golden Signals (SRE):
├── Latency    — Request kitne time mein serve hua
├── Traffic    — Kitne requests aa rahe hain
├── Errors     — Error rate (4xx/5xx)
└── Saturation — Resources kitne full hain (CPU/Memory)
```

---

## OBS-Q2. "Prometheus setup karo Kubernetes mein — kube-prometheus-stack."

```bash
# kube-prometheus-stack — Prometheus + Grafana + AlertManager sab ek saath
helm repo add prometheus-community \
    https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack \
    prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --create-namespace \
    --values prometheus-values.yaml
```

```yaml
# prometheus-values.yaml
prometheus:
  prometheusSpec:
    retention: 30d              # 30 din data rakho
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 100Gi
    
    # Scrape interval
    scrapeInterval: 30s
    evaluationInterval: 30s
    
    # Extra scrape configs
    additionalScrapeConfigs:
    - job_name: 'my-app'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: [production]
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

grafana:
  adminPassword: "changeme"
  persistence:
    enabled: true
    size: 10Gi

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 10Gi
```

---

## OBS-Q3. "Application metrics expose karo — Custom metrics."

```python
# Python app mein Prometheus metrics (prometheus-client library)
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

# Metrics define karo
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status_code']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

ACTIVE_CONNECTIONS = Gauge(
    'active_connections',
    'Number of active connections'
)

DB_POOL_SIZE = Gauge('db_connection_pool_size', 'DB pool size')

# Use karo
@app.route('/api/users')
def get_users():
    start = time.time()
    ACTIVE_CONNECTIONS.inc()
    
    try:
        # Logic
        result = db.query("SELECT * FROM users")
        REQUEST_COUNT.labels(method='GET', endpoint='/api/users', status_code=200).inc()
        return result
    except Exception as e:
        REQUEST_COUNT.labels(method='GET', endpoint='/api/users', status_code=500).inc()
        raise
    finally:
        REQUEST_LATENCY.labels(method='GET', endpoint='/api/users').observe(time.time() - start)
        ACTIVE_CONNECTIONS.dec()

# /metrics endpoint start karo
start_http_server(9090)
```

```yaml
# Kubernetes annotation — Prometheus automatically scrape karega
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
```

---

## OBS-Q4. "PromQL — Important queries jo interview mein aate hain."

```promql
# === REQUEST METRICS ===

# Request rate (per second, last 5 min)
rate(http_requests_total[5m])

# Error rate percentage
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# P99 Latency
histogram_quantile(0.99,
    sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# === KUBERNETES METRICS ===

# Pod CPU usage percentage
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)
/
sum(container_spec_cpu_quota{container!=""}) by (pod)
* 100

# Memory usage
sum(container_memory_working_set_bytes{container!=""}) by (pod)

# CPU Throttling
sum(rate(container_cpu_cfs_throttled_seconds_total[5m])) by (pod)
> 0

# Node disk usage
(node_filesystem_size_bytes - node_filesystem_free_bytes)
/
node_filesystem_size_bytes * 100

# === ALERTING RULES ===

groups:
- name: critical-alerts
  rules:
  - alert: HighErrorRate
    expr: |
      sum(rate(http_requests_total{status_code=~"5.."}[5m]))
      /
      sum(rate(http_requests_total[5m]))
      > 0.05
    for: 2m           # 2 min lagaataar hona chahiye
    labels:
      severity: critical
    annotations:
      summary: "High error rate: {{ $value | humanizePercentage }}"
      runbook: "https://wiki.company.com/runbooks/high-error-rate"
  
  - alert: PodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} is crash looping"
  
  - alert: NodeDiskPressure
    expr: |
      (node_filesystem_size_bytes - node_filesystem_free_bytes)
      / node_filesystem_size_bytes * 100 > 85
    for: 5m
    labels:
      severity: warning
```

---

## OBS-Q5. "AlertManager configure karo — Routing, Grouping, Silencing."

```yaml
# alertmanager.yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/...'

# Alert routing tree
route:
  group_by: ['alertname', 'cluster', 'namespace']
  group_wait: 30s       # Group gather karne ka time
  group_interval: 5m    # Grouped alerts ka interval
  repeat_interval: 4h   # Repeat kab bheje (resolved nahi hua to)
  receiver: 'slack-default'
  
  routes:
  # Critical → PagerDuty + Slack
  - match:
      severity: critical
    receiver: pagerduty-critical
    routes:
    - match:
        namespace: production
      receiver: pagerduty-prod
  
  # Warning → Slack only
  - match:
      severity: warning
    receiver: slack-warnings
  
  # Specific team
  - match_re:
      alertname: '^(Database|Postgres).*'
    receiver: dba-team

receivers:
- name: 'slack-default'
  slack_configs:
  - channel: '#alerts'
    send_resolved: true
    title: '{{ template "slack.title" . }}'
    text: '{{ template "slack.text" . }}'
    color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'

- name: 'pagerduty-critical'
  pagerduty_configs:
  - routing_key: '$pagerduty-key'
    description: '{{ .CommonAnnotations.summary }}'

- name: 'dba-team'
  email_configs:
  - to: 'dba-team@company.com'
    subject: '{{ .GroupLabels.alertname }}'

inhibit_rules:
# Critical alert agar aa gaya to warning suppress karo
- source_match:
    severity: critical
  target_match:
    severity: warning
  equal: ['alertname', 'cluster', 'namespace']
```

---

## OBS-Q6. "Grafana Dashboard — Important dashboards aur panels."

```json
// Grafana dashboard tips

// Dashboard variables (dropdowns)
{
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "query": "label_values(kube_pod_info, namespace)",
        "refresh": 2
      },
      {
        "name": "pod",
        "type": "query",
        "query": "label_values(kube_pod_info{namespace=\"$namespace\"}, pod)"
      }
    ]
  }
}
```

```
Must-Have Dashboards:
├── Kubernetes Overview (Node/Pod health)
├── Application Performance (latency, errors, traffic)
├── Infrastructure (CPU, Memory, Disk, Network)
├── Database Performance
└── Business Metrics (custom)

Pre-built Dashboards (Grafana.com):
├── ID 3119 — Kubernetes cluster monitoring
├── ID 6417 — Kubernetes pods
├── ID 1860 — Node Exporter Full
└── ID 315  — Kubernetes cluster monitoring

Panel Types:
├── Time series — Metrics over time
├── Stat        — Single number
├── Table       — Tabular data
├── Gauge       — Percentage/capacity
└── Heatmap     — Distribution (latency buckets)
```

---

## OBS-Q7. "Loki — Log aggregation setup aur queries."

```yaml
# Loki stack install
helm install loki grafana/loki-stack \
    --namespace monitoring \
    --set grafana.enabled=false \
    --set prometheus.enabled=false \
    --set loki.persistence.enabled=true \
    --set loki.persistence.size=50Gi

# Promtail — Log collector (node pe DaemonSet)
# ya
# Fluent Bit — More powerful log collector
```

```logql
# === LOGQL QUERIES ===

# Basic log stream select
{namespace="production", app="my-app"}

# Filter — error wale logs
{namespace="production"} |= "ERROR"

# Filter out — health check logs exclude
{namespace="production", app="backend"} != "/health"

# Regex filter
{app="nginx"} |~ "status=5[0-9][0-9]"

# JSON parsing
{app="my-app"} | json | level="error"

# Label filter
{app="my-app"} | json | status_code >= 500

# Log rate (per minute)
rate({namespace="production"} |= "ERROR" [1m])

# Error count last 5 min
count_over_time({namespace="production"} |= "ERROR" [5m])

# Top 10 error messages
topk(10,
    sum by (message) (
        count_over_time(
            {namespace="production"} |= "ERROR" | json | __error__="" [1h]
        )
    )
)
```

---

## OBS-Q8. "Distributed Tracing — OpenTelemetry aur Jaeger."

```python
# OpenTelemetry setup — Python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

# Provider setup
provider = TracerProvider()
exporter = OTLPSpanExporter(endpoint="http://jaeger:4317", insecure=True)
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)

# Auto-instrumentation
FlaskInstrumentor().instrument_app(app)    # HTTP requests auto-trace
RequestsInstrumentor().instrument()         # Outgoing HTTP
SQLAlchemyInstrumentor().instrument(engine=db_engine)  # DB queries

# Manual spans
tracer = trace.get_tracer(__name__)

def process_order(order_id):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("user.id", current_user.id)
        
        try:
            result = db.process(order_id)
            span.set_attribute("order.status", "success")
            return result
        except Exception as e:
            span.record_exception(e)
            span.set_status(trace.Status(trace.StatusCode.ERROR))
            raise
```

```yaml
# Kubernetes deployment mein OTEL config
env:
- name: OTEL_EXPORTER_OTLP_ENDPOINT
  value: "http://otel-collector:4317"
- name: OTEL_SERVICE_NAME
  value: "my-app"
- name: OTEL_RESOURCE_ATTRIBUTES
  value: "deployment.environment=production,k8s.namespace.name=production"
```

---

## OBS-Q9. "SLI, SLO, SLA — Error Budget kya hai?"

```
SLI (Service Level Indicator) — Actual measurement
└── Example: "Last 30 days request success rate = 99.5%"

SLO (Service Level Objective) — Target goal
└── Example: "99.9% requests successful in 30 days"

SLA (Service Level Agreement) — Business contract (with penalty)
└── Example: "99.9% uptime guarantee, else refund"

Error Budget — SLO se kitna failure allowed hai
├── SLO = 99.9%
├── Error budget = 100% - 99.9% = 0.1%
├── 30 days = 43200 minutes
├── Error budget = 43200 × 0.001 = 43.2 minutes
└── Itna downtime allow hai mahine mein
```

```yaml
# Prometheus mein SLO tracking

# SLI calculation
- record: sli:request_success_rate:5m
  expr: |
    sum(rate(http_requests_total{status_code!~"5.."}[5m]))
    /
    sum(rate(http_requests_total[5m]))

# Error budget burn rate alert
- alert: ErrorBudgetBurnRateHigh
  expr: |
    (
      1 - sli:request_success_rate:5m
    ) > (1 - 0.999) * 14.4  # 1 hour mein 1% budget burn
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Error budget burning fast — investigate immediately"
```

---

## OBS-Q10. "Kubernetes Events aur Audit Logging."

```bash
# Events — Real-time cluster events
kubectl get events -n production
kubectl get events -n production --sort-by='.lastTimestamp'
kubectl get events -n production --field-selector reason=OOMKilling
kubectl get events -A --field-selector type=Warning

# Event watch
kubectl get events -n production -w

# Specific resource ke events
kubectl describe pod my-pod | grep -A 20 "Events"
```

```yaml
# Audit Policy — API requests log karo
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Sensitive operations — full log
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
  verbs: ["create", "update", "patch", "delete"]

# Delete operations — log karo
- level: Request
  verbs: ["delete", "deletecollection"]

# Read operations — metadata only
- level: Metadata
  resources:
  - group: ""
    resources: ["pods"]
  verbs: ["get", "list", "watch"]

# Health checks — ignore
- level: None
  users: ["system:serviceaccount:kube-system:*"]
  nonResourceURLs: ["/healthz", "/readyz", "/livez"]
```

---

## OBS-Q11. "Runbooks aur Incident Response."

```markdown
# Runbook: High Error Rate Alert

## Alert: HighErrorRate (>5% for 2+ minutes)

### Immediate Steps (0-5 min)
1. Check which service: `kubectl top pods -n production`
2. Check recent deployments: `kubectl rollout history deployment -n production`
3. Error logs: `{namespace="production"} |= "ERROR" | json`

### Investigation (5-15 min)
1. Specific errors:
   kubectl logs -l app=my-app -n production --tail=100 | grep ERROR

2. Database connectivity:
   kubectl exec -it <pod> -- nc -zv postgres 5432

3. Downstream service:
   kubectl exec -it <pod> -- curl -v http://payment-svc/health

### Remediation
- Recent deploy ki wajah se: `kubectl rollout undo deployment/my-app`
- Database issue: Check DB metrics in Grafana
- Memory leak: `kubectl rollout restart deployment/my-app`

### Escalation
- 15+ min unresolved → Page on-call lead
- Production data loss risk → Page CTO
```

---

## OBS-Q12. "OpenTelemetry Collector — Centralized telemetry pipeline."

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  
  # Prometheus scrape
  prometheus:
    config:
      scrape_configs:
      - job_name: 'kubernetes'
        kubernetes_sd_configs:
        - role: pod

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024
  
  # Sensitive data remove karo
  attributes:
    actions:
    - key: db.user
      action: delete
    - key: http.request.header.authorization
      action: delete
  
  # Resource attributes add karo
  resource:
    attributes:
    - key: cluster.name
      value: production
      action: upsert

exporters:
  # Traces → Jaeger
  jaeger:
    endpoint: jaeger-collector:14250
    tls:
      insecure: true
  
  # Metrics → Prometheus
  prometheus:
    endpoint: 0.0.0.0:8889
  
  # Logs → Loki
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, attributes, resource]
      exporters: [jaeger]
    
    metrics:
      receivers: [otlp, prometheus]
      processors: [batch]
      exporters: [prometheus]
    
    logs:
      receivers: [otlp]
      processors: [batch, attributes]
      exporters: [loki]
```

---
---

# ☁️ AWS — DevOps Interview Q&A (25 Scenarios)

> EC2, VPC, EKS, ECS, S3, RDS, IAM, Lambda, CloudFormation, CloudWatch + more

---

## AWS-Q1. "VPC design karo — Production-grade 3-tier architecture."

```
VPC CIDR: 10.0.0.0/16

PUBLIC SUBNETS (Internet access via IGW)
├── 10.0.1.0/24 — ap-south-1a (NAT Gateway, Bastion, ALB)
└── 10.0.2.0/24 — ap-south-1b (ALB, redundancy)

PRIVATE SUBNETS (Internet via NAT Gateway)
├── 10.0.10.0/24 — ap-south-1a (App servers, EKS nodes)
└── 10.0.11.0/24 — ap-south-1b (App servers, EKS nodes)

DATABASE SUBNETS (No internet, only from private)
├── 10.0.20.0/24 — ap-south-1a (RDS Primary)
└── 10.0.21.0/24 — ap-south-1b (RDS Standby)

COMPONENTS
├── Internet Gateway — Public internet
├── NAT Gateway (AZ per) — Private outbound
├── Route Tables — Per subnet
├── Security Groups — Instance level firewall
└── NACLs — Subnet level stateless firewall
```

```bash
# Route Tables
# Public: 0.0.0.0/0 → Internet Gateway
# Private: 0.0.0.0/0 → NAT Gateway
# Database: No internet route (local only)

# Security Group design
# ALB-SG:    Inbound 443/80 from 0.0.0.0/0
# App-SG:    Inbound 8080 from ALB-SG only
# DB-SG:     Inbound 5432 from App-SG only
# Bastion-SG: Inbound 22 from office IP only
```

---

## AWS-Q2. "IAM — Least privilege principle aur roles setup."

```json
// IAM Policy — EC2 specific S3 access
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-app-bucket/${aws:username}/*",
      "Condition": {
        "StringEquals": {
          "s3:prefix": ["uploads/"]
        },
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    },
    {
      "Effect": "Deny",
      "Action": "s3:DeleteBucket",
      "Resource": "*"
    }
  ]
}
```

```bash
# IAM Role for EC2 (Instance Profile)
aws iam create-role \
    --role-name ec2-app-role \
    --assume-role-policy-document '{
      "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "ec2.amazonaws.com"},
        "Action": "sts:AssumeRole"
      }]
    }'

# Policy attach karo
aws iam attach-role-policy \
    --role-name ec2-app-role \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# IRSA — Kubernetes pods ke liye AWS access
eksctl create iamserviceaccount \
    --cluster my-cluster \
    --namespace production \
    --name my-app-sa \
    --attach-policy-arn arn:aws:iam::123456789:policy/my-app-policy \
    --approve

# Test permissions
aws sts get-caller-identity
aws iam simulate-principal-policy \
    --policy-source-arn arn:aws:iam::123456789:role/my-role \
    --action-names s3:GetObject \
    --resource-arns arn:aws:s3:::my-bucket/*
```

---

## AWS-Q3. "EKS Cluster setup karo — Production ready."

```bash
# eksctl se EKS create (simplest)
eksctl create cluster \
    --name production \
    --region ap-south-1 \
    --version 1.29 \
    --nodegroup-name workers \
    --node-type m5.large \
    --nodes 3 \
    --nodes-min 2 \
    --nodes-max 10 \
    --managed \
    --asg-access \
    --external-dns-access \
    --full-ecr-access \
    --alb-ingress-access \
    --with-oidc   # IRSA ke liye

# Ya Terraform se (production preferred)

# Add-ons
eksctl create addon --name aws-ebs-csi-driver --cluster production
eksctl create addon --name coredns --cluster production
eksctl create addon --name kube-proxy --cluster production
eksctl create addon --name vpc-cni --cluster production

# Cluster Autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
    --set autoDiscovery.clusterName=production \
    --set awsRegion=ap-south-1 \
    --namespace kube-system

# AWS Load Balancer Controller
helm install aws-load-balancer-controller \
    eks/aws-load-balancer-controller \
    --set clusterName=production \
    --namespace kube-system
```

---

## AWS-Q4. "ALB + Target Groups + Health Checks configure karo."

```bash
# ALB create karo (Terraform ya CLI)
aws elbv2 create-load-balancer \
    --name production-alb \
    --type application \
    --scheme internet-facing \
    --subnets subnet-1a subnet-1b \
    --security-groups alb-sg

# Target Group
aws elbv2 create-target-group \
    --name my-app-tg \
    --protocol HTTP \
    --port 8080 \
    --vpc-id vpc-abc123 \
    --health-check-path /health \
    --health-check-interval-seconds 30 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 3

# Listener
aws elbv2 create-listener \
    --load-balancer-arn <alb-arn> \
    --protocol HTTPS \
    --port 443 \
    --certificates CertificateArn=<acm-cert-arn> \
    --default-actions Type=forward,TargetGroupArn=<tg-arn>

# HTTP → HTTPS redirect
aws elbv2 create-listener \
    --load-balancer-arn <alb-arn> \
    --protocol HTTP \
    --port 80 \
    --default-actions \
        Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'

# Target health check
aws elbv2 describe-target-health --target-group-arn <tg-arn>
```

---

## AWS-Q5. "Auto Scaling Group setup karo — Scale on CPU aur custom metrics."

```bash
# Launch Template
aws ec2 create-launch-template \
    --launch-template-name my-app-lt \
    --launch-template-data '{
        "ImageId": "ami-abc123",
        "InstanceType": "m5.large",
        "SecurityGroupIds": ["sg-app"],
        "IamInstanceProfile": {"Name": "ec2-app-profile"},
        "UserData": "<base64-encoded-script>"
    }'

# ASG create
aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name my-app-asg \
    --launch-template LaunchTemplateId=lt-abc123,Version='$Latest' \
    --min-size 2 \
    --max-size 20 \
    --desired-capacity 3 \
    --vpc-zone-identifier "subnet-1a,subnet-1b" \
    --target-group-arns <tg-arn> \
    --health-check-type ELB \
    --health-check-grace-period 300

# Scaling Policy — Target Tracking (recommended)
aws autoscaling put-scaling-policy \
    --auto-scaling-group-name my-app-asg \
    --policy-name cpu-target-tracking \
    --policy-type TargetTrackingScaling \
    --target-tracking-configuration '{
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "ASGAverageCPUUtilization"
        },
        "TargetValue": 70.0,
        "ScaleInCooldown": 300,
        "ScaleOutCooldown": 60
    }'
```

---

## AWS-Q6. "RDS — High Availability, Backup, Performance."

```bash
# RDS Multi-AZ (High Availability)
aws rds create-db-instance \
    --db-instance-identifier production-postgres \
    --db-instance-class db.r6g.xlarge \
    --engine postgres \
    --engine-version 15.4 \
    --master-username admin \
    --master-user-password 'MySecurePass123!' \
    --db-name myapp \
    --multi-az \                    # Standby in another AZ
    --storage-type gp3 \
    --allocated-storage 100 \
    --max-allocated-storage 1000 \  # Auto-scale storage
    --storage-encrypted \
    --db-subnet-group-name db-subnet-group \
    --vpc-security-group-ids sg-db \
    --backup-retention-period 7 \
    --preferred-backup-window "02:00-03:00" \
    --deletion-protection \
    --enable-performance-insights \
    --performance-insights-retention-period 7

# Read Replica (read scaling ke liye)
aws rds create-db-instance-read-replica \
    --db-instance-identifier prod-postgres-replica \
    --source-db-instance-identifier production-postgres \
    --db-instance-class db.r6g.large

# Point-in-time restore
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier production-postgres \
    --target-db-instance-identifier restored-postgres \
    --restore-time 2024-01-15T10:00:00Z

# RDS Proxy (connection pooling)
aws rds create-db-proxy \
    --db-proxy-name my-rds-proxy \
    --engine-family POSTGRESQL \
    --auth '[{"AuthScheme":"SECRETS","SecretArn":"arn:aws:secretsmanager:..."}]' \
    --role-arn <proxy-role-arn> \
    --vpc-subnet-ids subnet-1a subnet-1b \
    --vpc-security-group-ids sg-db
```

---

## AWS-Q7. "S3 — Versioning, Lifecycle, Encryption, Access Control."

```bash
# Bucket create with encryption
aws s3api create-bucket \
    --bucket my-production-bucket \
    --region ap-south-1 \
    --create-bucket-configuration LocationConstraint=ap-south-1

# Encryption enable
aws s3api put-bucket-encryption \
    --bucket my-production-bucket \
    --server-side-encryption-configuration '{
        "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "aws:kms"}}]
    }'

# Versioning enable
aws s3api put-bucket-versioning \
    --bucket my-production-bucket \
    --versioning-configuration Status=Enabled

# Public access block (IMPORTANT!)
aws s3api put-public-access-block \
    --bucket my-production-bucket \
    --public-access-block-configuration \
        BlockPublicAcls=true,IgnorePublicAcls=true,\
        BlockPublicPolicy=true,RestrictPublicBuckets=true

# Lifecycle rules — cost optimization
aws s3api put-bucket-lifecycle-configuration \
    --bucket my-production-bucket \
    --lifecycle-configuration '{
        "Rules": [{
            "ID": "archive-old",
            "Status": "Enabled",
            "Filter": {"Prefix": "logs/"},
            "Transitions": [
                {"Days": 30, "StorageClass": "STANDARD_IA"},
                {"Days": 90, "StorageClass": "GLACIER"},
                {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
            ],
            "NoncurrentVersionExpiration": {"NoncurrentDays": 30}
        }]
    }'
```

---

## AWS-Q8. "CloudWatch — Metrics, Logs, Alarms setup."

```bash
# Custom metric publish
aws cloudwatch put-metric-data \
    --namespace "MyApp/Production" \
    --metric-name "ActiveUsers" \
    --value 1234 \
    --unit Count \
    --dimensions Environment=production,Service=backend

# Alarm create
aws cloudwatch put-metric-alarm \
    --alarm-name "High-CPU-Production" \
    --metric-name CPUUtilization \
    --namespace AWS/EC2 \
    --dimensions Name=AutoScalingGroupName,Value=my-app-asg \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions arn:aws:sns:ap-south-1:123456789:alerts \
    --ok-actions arn:aws:sns:ap-south-1:123456789:alerts

# Log Group + Metric Filter
aws logs create-log-group --log-group-name /app/production

aws logs put-metric-filter \
    --log-group-name /app/production \
    --filter-name ErrorCount \
    --filter-pattern "ERROR" \
    --metric-transformations \
        metricName=ErrorCount,metricNamespace=MyApp,metricValue=1

# Container Insights for EKS
aws eks update-cluster-config \
    --name production \
    --logging '{"clusterLogging":[
        {"types":["api","audit","authenticator","controllerManager","scheduler"],
         "enabled":true}
    ]}'
```

---

## AWS-Q9. "AWS Secrets Manager — Secrets lifecycle manage karo."

```bash
# Secret create
aws secretsmanager create-secret \
    --name production/myapp/db \
    --description "Production DB credentials" \
    --secret-string '{"username":"admin","password":"MyPass123!","host":"db.prod.internal","port":5432}'

# Auto-rotation enable karo (RDS ke liye built-in)
aws secretsmanager rotate-secret \
    --secret-id production/myapp/db \
    --rotation-lambda-arn arn:aws:lambda:ap-south-1:123456789:function:SecretsManagerRDSPostgreSQLRotationSingleUser \
    --rotation-rules AutomaticallyAfterDays=30

# Application mein use karo (boto3)
import boto3, json
client = boto3.client('secretsmanager', region_name='ap-south-1')
secret = client.get_secret_value(SecretId='production/myapp/db')
creds = json.loads(secret['SecretString'])
DB_URL = f"postgresql://{creds['username']}:{creds['password']}@{creds['host']}:{creds['port']}/mydb"

# Version management
aws secretsmanager list-secret-version-ids \
    --secret-id production/myapp/db
```

---

## AWS-Q10. "Lambda — Serverless functions aur event triggers."

```python
# Lambda function — API Gateway trigger
import json
import boto3
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('users')

def lambda_handler(event, context):
    """API Gateway Lambda proxy handler"""

    logger.info(f"Event: {json.dumps(event)}")

    http_method = event['httpMethod']
    path = event['path']
    path_params = event.get('pathParameters', {})

    try:
        if http_method == 'GET' and path == '/users/{id}':
            user_id = path_params['id']
            response = table.get_item(Key={'id': user_id})
            item = response.get('Item')

            if not item:
                return {
                    'statusCode': 404,
                    'body': json.dumps({'error': 'User not found'})
                }

            return {
                'statusCode': 200,
                'headers': {'Content-Type': 'application/json'},
                'body': json.dumps(item)
            }

    except Exception as e:
        logger.error(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Internal server error'})
        }
```

```bash
# Lambda deploy
zip function.zip lambda_function.py
aws lambda create-function \
    --function-name my-api \
    --runtime python3.11 \
    --role arn:aws:iam::123456789:role/lambda-role \
    --handler lambda_function.lambda_handler \
    --zip-file fileb://function.zip \
    --memory-size 256 \
    --timeout 30 \
    --environment Variables='{TABLE_NAME=users}'

# Environment variables update
aws lambda update-function-configuration \
    --function-name my-api \
    --environment Variables='{TABLE_NAME=users,LOG_LEVEL=INFO}'
```

---

## AWS-Q11. "CloudFront — CDN setup aur cache optimization."

```bash
# CloudFront distribution create
aws cloudfront create-distribution \
    --distribution-config '{
        "Origins": {
            "Items": [{
                "Id": "ALB-Origin",
                "DomainName": "my-alb.ap-south-1.elb.amazonaws.com",
                "CustomOriginConfig": {
                    "HTTPPort": 80,
                    "HTTPSPort": 443,
                    "OriginProtocolPolicy": "https-only"
                }
            }, {
                "Id": "S3-Static",
                "DomainName": "my-bucket.s3.amazonaws.com",
                "S3OriginConfig": {"OriginAccessIdentity": "origin-access-identity/cloudfront/..."}
            }]
        },
        "DefaultCacheBehavior": {
            "TargetOriginId": "ALB-Origin",
            "ViewerProtocolPolicy": "redirect-to-https",
            "CachePolicyId": "...",
            "Compress": true
        },
        "CacheBehaviors": {
            "Items": [{
                "PathPattern": "/static/*",
                "TargetOriginId": "S3-Static",
                "CachePolicyId": "Managed-CachingOptimized",
                "ViewerProtocolPolicy": "redirect-to-https"
            }]
        }
    }'

# Cache invalidation (deploy ke baad)
aws cloudfront create-invalidation \
    --distribution-id E1234567 \
    --paths "/*"

# Cache-Control headers (S3 objects pe)
aws s3 cp index.html s3://my-bucket/ \
    --cache-control "public, max-age=31536000, immutable"  # JS/CSS
aws s3 cp index.html s3://my-bucket/ \
    --cache-control "no-cache, no-store"  # HTML
```

---

## AWS-Q12. "ECR — Container registry management."

```bash
# ECR Repository create
aws ecr create-repository \
    --repository-name my-app \
    --region ap-south-1 \
    --image-scanning-configuration scanOnPush=true \
    --encryption-configuration encryptionType=AES256

# Login
aws ecr get-login-password --region ap-south-1 \
    | docker login --username AWS \
    --password-stdin 123456789.dkr.ecr.ap-south-1.amazonaws.com

# Build + push
docker build -t my-app:latest .
docker tag my-app:latest 123456789.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest
docker push 123456789.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest

# Lifecycle policy — purane images delete karo
aws ecr put-lifecycle-policy \
    --repository-name my-app \
    --lifecycle-policy-text '{
        "rules": [{
            "rulePriority": 1,
            "description": "Keep last 10 images",
            "selection": {
                "tagStatus": "any",
                "countType": "imageCountMoreThan",
                "countNumber": 10
            },
            "action": {"type": "expire"}
        }]
    }'

# Image scan results
aws ecr describe-image-scan-findings \
    --repository-name my-app \
    --image-id imageTag=latest \
    --query 'imageScanFindings.findings[?severity==`CRITICAL`]'
```

---

## AWS-Q13. "Route53 — DNS management aur Health Checks."

```bash
# Hosted zone create
aws route53 create-hosted-zone \
    --name example.com \
    --caller-reference $(date +%s)

# Record create (A record)
aws route53 change-resource-record-sets \
    --hosted-zone-id Z123456 \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "api.example.com",
                "Type": "A",
                "AliasTarget": {
                    "DNSName": "my-alb.ap-south-1.elb.amazonaws.com",
                    "EvaluateTargetHealth": true,
                    "HostedZoneId": "ZP97RAFLXTNZK"  # ALB hosted zone ID
                }
            }
        }]
    }'

# Health Check + Failover
# Primary record (main region)
aws route53 create-health-check \
    --caller-reference $(date +%s) \
    --health-check-config '{
        "Type": "HTTPS",
        "FullyQualifiedDomainName": "api.example.com",
        "ResourcePath": "/health",
        "RequestInterval": 30,
        "FailureThreshold": 3
    }'

# Weighted routing (A/B testing)
# 90% primary, 10% canary
```

---

## AWS-Q14. "Systems Manager (SSM) — EC2 management without SSH."

```bash
# SSM Session (SSH ki zaroorat nahi)
aws ssm start-session --target i-1234567890

# Parameter Store — Secrets ke liye
aws ssm put-parameter \
    --name /production/myapp/db-password \
    --value "MySecurePassword" \
    --type SecureString \
    --key-id alias/myapp-key

# Get parameter
aws ssm get-parameter \
    --name /production/myapp/db-password \
    --with-decryption \
    --query 'Parameter.Value' \
    --output text

# Run Command — Multiple instances pe
aws ssm send-command \
    --targets "Key=tag:Environment,Values=production" \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["sudo systemctl restart nginx"]' \
    --output-s3-bucket-name my-ssm-logs

# Patch Management
aws ssm start-automation-execution \
    --document-name AWS-PatchInstanceWithRollback \
    --parameters InstanceId=i-1234567890
```

---

## AWS-Q15. "Cost Optimization — EC2, S3, RDS cost reduce karo."

```bash
# Reserved Instances vs Spot vs On-demand
# On-demand: Full price, no commitment
# Reserved: 40-75% discount, 1-3 year commitment
# Spot: 70-90% discount, can be interrupted
# Savings Plans: Flexible (compute/EC2)

# Spot Instances for non-critical workloads
aws ec2 request-spot-instances \
    --instance-count 2 \
    --spot-price "0.05" \
    --launch-specification '{
        "ImageId": "ami-abc123",
        "InstanceType": "m5.large"
    }'

# Cost analysis
aws ce get-cost-and-usage \
    --time-period Start=2024-01-01,End=2024-01-31 \
    --granularity MONTHLY \
    --metrics BlendedCost \
    --group-by Type=DIMENSION,Key=SERVICE

# Unused resources find karo
# Unattached EBS volumes
aws ec2 describe-volumes \
    --filters Name=status,Values=available \
    --query 'Volumes[*].{ID:VolumeId,Size:Size}'

# Idle load balancers
aws elbv2 describe-load-balancers \
    --query 'LoadBalancers[?State.Code==`active`]'

# S3 Cost optimization
# Intelligent Tiering enable karo
aws s3api put-bucket-intelligent-tiering-configuration \
    --bucket my-bucket \
    --id whole-bucket \
    --intelligent-tiering-configuration '{
        "Id": "whole-bucket",
        "Status": "Enabled",
        "Tierings": [
            {"AccessTier": "ARCHIVE_ACCESS", "Days": 90},
            {"AccessTier": "DEEP_ARCHIVE_ACCESS", "Days": 180}
        ]
    }'
```

---

## AWS-Q16. "CloudTrail + Config — Audit aur Compliance."

```bash
# CloudTrail — API calls log karo
aws cloudtrail create-trail \
    --name production-trail \
    --s3-bucket-name my-cloudtrail-bucket \
    --is-multi-region-trail \
    --enable-log-file-validation \
    --include-global-service-events

aws cloudtrail start-logging --name production-trail

# Event query (Athena ya CloudTrail Insights)
# Kisi ne Security Group delete kiya?
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteSecurityGroup \
    --start-time 2024-01-01 \
    --end-time 2024-01-31

# AWS Config — Resource configuration track
aws configservice put-config-rule \
    --config-rule '{
        "ConfigRuleName": "restricted-ssh",
        "Source": {
            "Owner": "AWS",
            "SourceIdentifier": "INCOMING_SSH_DISABLED"
        }
    }'

# Non-compliant resources
aws configservice get-compliance-details-by-config-rule \
    --config-rule-name restricted-ssh \
    --compliance-types NON_COMPLIANT
```

---

## AWS-Q17. "SQS + SNS — Message queuing aur notifications."

```bash
# SQS Queue create
aws sqs create-queue \
    --queue-name my-app-queue \
    --attributes '{
        "VisibilityTimeout": "300",
        "MessageRetentionPeriod": "86400",
        "ReceiveMessageWaitTimeSeconds": "20",  # Long polling
        "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:...dlq\",\"maxReceiveCount\":\"3\"}"
    }'

# Message bhejo
aws sqs send-message \
    --queue-url https://sqs.ap-south-1.amazonaws.com/123456789/my-app-queue \
    --message-body '{"orderId": "12345", "action": "process"}'

# Message receive
aws sqs receive-message \
    --queue-url <queue-url> \
    --max-number-of-messages 10 \
    --wait-time-seconds 20   # Long polling

# SNS Topic + Subscribers
aws sns create-topic --name my-alerts
aws sns subscribe \
    --topic-arn arn:aws:sns:ap-south-1:123456789:my-alerts \
    --protocol email \
    --notification-endpoint team@company.com

# SQS ko SNS se subscribe karo (fan-out pattern)
aws sns subscribe \
    --topic-arn <topic-arn> \
    --protocol sqs \
    --notification-endpoint <queue-arn>
```

---

## AWS-Q18. "ECS vs EKS — Kab kya use karo?"

```
ECS (Elastic Container Service)
├── AWS managed (simpler)
├── EC2 launch type ya Fargate (serverless)
├── Deep AWS integration
├── Less operational overhead
└── Use karo: Simple containerized apps, small teams

EKS (Elastic Kubernetes Service)
├── Kubernetes standard
├── More control, more complexity
├── Multi-cloud portability
├── Large ecosystem
└── Use karo: Complex microservices, K8s expertise hai, multi-cloud

Fargate vs EC2 Launch Type:
├── Fargate: Serverless, no node management, per-task billing
└── EC2: More control, cheaper at scale, GPU support

ECS Task Definition:
```

```json
{
  "family": "my-app",
  "cpu": "256",
  "memory": "512",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "containerDefinitions": [{
    "name": "my-app",
    "image": "123456789.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest",
    "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/my-app",
        "awslogs-region": "ap-south-1",
        "awslogs-stream-prefix": "ecs"
      }
    },
    "secrets": [{
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:..."
    }]
  }]
}
```

---

## AWS-Q19. "KMS — Encryption key management."

```bash
# KMS Key create
aws kms create-key \
    --description "Production app encryption key" \
    --key-usage ENCRYPT_DECRYPT \
    --policy '{
        "Statement": [{
            "Effect": "Allow",
            "Principal": {"AWS": "arn:aws:iam::123456789:root"},
            "Action": "kms:*",
            "Resource": "*"
        }, {
            "Effect": "Allow",
            "Principal": {"Service": "s3.amazonaws.com"},
            "Action": ["kms:GenerateDataKey", "kms:Decrypt"],
            "Resource": "*"
        }]
    }'

# Alias
aws kms create-alias \
    --alias-name alias/production-key \
    --target-key-id <key-id>

# Encrypt/Decrypt
aws kms encrypt \
    --key-id alias/production-key \
    --plaintext "mysecretvalue" \
    --query CiphertextBlob \
    --output text | base64 -d > encrypted.bin

aws kms decrypt \
    --ciphertext-blob fileb://encrypted.bin \
    --query Plaintext \
    --output text | base64 -d
```

---

## AWS-Q20. "Well-Architected Framework — 6 Pillars."

```
1. OPERATIONAL EXCELLENCE
   ├── IaC (Terraform/CloudFormation)
   ├── Monitoring aur observability
   ├── Runbooks aur automation
   └── Continuous improvement

2. SECURITY
   ├── IAM least privilege
   ├── Encryption at rest + transit
   ├── VPC security groups + NACLs
   └── CloudTrail + Config

3. RELIABILITY
   ├── Multi-AZ deployment
   ├── Auto Scaling
   ├── Backup + DR
   └── Chaos engineering

4. PERFORMANCE EFFICIENCY
   ├── Right-sizing instances
   ├── CloudFront CDN
   ├── Caching (ElastiCache)
   └── Database optimization

5. COST OPTIMIZATION
   ├── Reserved + Spot instances
   ├── S3 Intelligent Tiering
   ├── Right-sizing
   └── Cost allocation tags

6. SUSTAINABILITY
   ├── Efficient resource usage
   ├── Serverless where possible
   └── Region selection (renewable energy)
```

---

## AWS-Q21. "Disaster Recovery — RTO/RPO aur strategies."

```
DR Strategies (cost badh raha hai):

1. BACKUP & RESTORE (RTO: hours, RPO: hours)
   └── S3 backups, cheapest, slowest recovery

2. PILOT LIGHT (RTO: 10-30 min, RPO: minutes)
   └── Minimal always-on infra, scale on disaster

3. WARM STANDBY (RTO: minutes, RPO: seconds)
   └── Scaled-down copy running in DR region

4. ACTIVE-ACTIVE (RTO: seconds, RPO: zero)
   └── Both regions serving traffic, most expensive

RTO: Recovery Time Objective  — Kitne time mein recover hoga
RPO: Recovery Point Objective — Kitna data loss acceptable hai

Implementation:
├── Route53 Health Checks + Failover routing
├── RDS Cross-Region Read Replica → Promote
├── S3 Cross-Region Replication
├── DynamoDB Global Tables
└── CloudFormation for quick infra recreation
```

---

## AWS-Q22. "EventBridge + Lambda — Event-driven architecture."

```python
# EventBridge Rule — EC2 state change pe Lambda trigger
```

```bash
aws events put-rule \
    --name "ec2-state-change" \
    --event-pattern '{
        "source": ["aws.ec2"],
        "detail-type": ["EC2 Instance State-change Notification"],
        "detail": {
            "state": ["terminated", "stopped"]
        }
    }' \
    --state ENABLED

aws events put-targets \
    --rule ec2-state-change \
    --targets '[{
        "Id": "notify-lambda",
        "Arn": "arn:aws:lambda:ap-south-1:123456789:function:ec2-notifier"
    }]'

# Custom events
aws events put-events \
    --entries '[{
        "Source": "my.app",
        "DetailType": "OrderPlaced",
        "Detail": "{\"orderId\": \"12345\", \"amount\": 99.99}"
    }]'
```

---

## AWS-Q23. "AWS GuardDuty + Security Hub — Security monitoring."

```bash
# GuardDuty enable
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES

# Findings check
aws guardduty list-findings \
    --detector-id <detector-id> \
    --finding-criteria '{
        "Criterion": {
            "severity": {"Gte": 7}  # High severity only
        }
    }'

# Security Hub enable
aws securityhub enable-security-hub \
    --enable-default-standards

# Findings aggregation
aws securityhub get-findings \
    --filters '{
        "SeverityLabel": [{"Value": "CRITICAL", "Comparison": "EQUALS"}],
        "RecordState": [{"Value": "ACTIVE", "Comparison": "EQUALS"}]
    }'

# Inspector — EC2/Container vulnerability scanning
aws inspector2 enable \
    --resource-types EC2 ECR
```

---

## AWS-Q24. "ElastiCache — Redis setup aur use cases."

```bash
# ElastiCache Redis create (cluster mode)
aws elasticache create-replication-group \
    --replication-group-id my-redis \
    --description "Production Redis" \
    --cache-node-type cache.r6g.large \
    --engine redis \
    --engine-version 7.0 \
    --num-cache-clusters 3 \        # 1 Primary + 2 Replicas
    --automatic-failover-enabled \
    --multi-az-enabled \
    --at-rest-encryption-enabled \
    --transit-encryption-enabled \
    --auth-token "MySecureAuthToken" \
    --cache-subnet-group-name my-subnet-group \
    --security-group-ids sg-redis

# Use cases:
# Session storage, Caching, Rate limiting, Leaderboards, Pub/Sub

# Python connection
import redis
r = redis.Redis(
    host='my-redis.abc123.cache.amazonaws.com',
    port=6379,
    password='MySecureAuthToken',
    ssl=True,
    decode_responses=True
)

# Cache pattern
def get_user(user_id):
    cache_key = f"user:{user_id}"
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)
    
    user = db.query_user(user_id)
    r.setex(cache_key, 3600, json.dumps(user))  # 1 hour TTL
    return user
```

---

## AWS-Q25. "Production Deployment Checklist — AWS Best Practices."

```
BEFORE DEPLOYMENT
✅ Multi-AZ configured (EC2, RDS, ElastiCache)
✅ Auto Scaling configured
✅ Health checks configured (ALB, ASG)
✅ SSL/TLS certificate (ACM)
✅ CloudFront CDN configured
✅ WAF rules configured
✅ VPC properly segmented
✅ Security groups least privilege
✅ Encryption enabled (EBS, RDS, S3)
✅ CloudTrail enabled
✅ GuardDuty enabled
✅ Backup strategy (RDS automated, S3 versioning)
✅ CloudWatch alarms configured
✅ SNS notifications set up
✅ IAM roles (not users) for services
✅ Secrets in Secrets Manager (not hardcoded)
✅ Tags on all resources (Environment, Owner, Project)
✅ Cost alerts configured
✅ DR tested

MONITORING
✅ CloudWatch dashboards
✅ Custom metrics published
✅ Log groups configured
✅ Alarms: CPU, Memory, Error rate, Latency

COST
✅ Reserved instances purchased
✅ Spot instances for non-critical
✅ S3 Intelligent Tiering
✅ Unattached volumes cleaned up
```

---
---

# 🏗️ Terraform — Interview Q&A (15 Scenarios)

> State management, Modules, Workspaces, Remote backend, CI/CD integration

---

## TF-Q1. "Terraform basics — Core concepts explain karo."

```hcl
# Terraform configuration structure

terraform {
  required_version = ">= 1.6.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"    # 5.x but not 6.x
    }
  }
  
  # Remote backend (important!)
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"   # State locking
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Environment = var.environment
      Project     = var.project_name
    }
  }
}
```

```
Terraform Core Commands:
├── terraform init      — Provider download, backend setup
├── terraform plan      — Preview changes (dry run)
├── terraform apply     — Apply changes
├── terraform destroy   — Destroy all
├── terraform validate  — Syntax check
├── terraform fmt       — Format code
├── terraform output    — Show outputs
├── terraform state     — State management
└── terraform import    — Existing resource import
```

---

## TF-Q2. "Variables, Outputs aur Data Sources."

```hcl
# variables.tf
variable "environment" {
  description = "Deployment environment"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Must be: dev, staging, or production"
  }
}

variable "instance_count" {
  type    = number
  default = 2
}

variable "allowed_ips" {
  type    = list(string)
  default = []
}

variable "tags" {
  type    = map(string)
  default = {}
}

# Sensitive variable
variable "db_password" {
  type      = string
  sensitive = true   # logs mein mask hoga
}

# outputs.tf
output "alb_dns_name" {
  description = "ALB DNS name"
  value       = aws_lb.main.dns_name
}

output "db_endpoint" {
  description = "RDS endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}

# data sources — existing resources use karo
data "aws_vpc" "existing" {
  tags = {
    Name = "production-vpc"
  }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/myapp/db"
}

# Use karo
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  vpc_id        = data.aws_vpc.existing.id
  
  password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
}
```

---

## TF-Q3. "Modules — Reusable infrastructure components."

```hcl
# modules/vpc/main.tf — VPC module
variable "vpc_cidr"     { type = string }
variable "environment"  { type = string }
variable "az_count"     { type = number; default = 2 }

locals {
  azs             = slice(data.aws_availability_zones.available.names, 0, var.az_count)
  public_cidrs    = [for i, az in local.azs : cidrsubnet(var.vpc_cidr, 8, i)]
  private_cidrs   = [for i, az in local.azs : cidrsubnet(var.vpc_cidr, 8, i + 10)]
  database_cidrs  = [for i, az in local.azs : cidrsubnet(var.vpc_cidr, 8, i + 20)]
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = { Name = "${var.environment}-vpc" }
}

resource "aws_subnet" "public" {
  count             = var.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_cidrs[count.index]
  availability_zone = local.azs[count.index]
  
  map_public_ip_on_launch = true
  tags = { Name = "${var.environment}-public-${local.azs[count.index]}" }
}

output "vpc_id"             { value = aws_vpc.main.id }
output "public_subnet_ids"  { value = aws_subnet.public[*].id }
output "private_subnet_ids" { value = aws_subnet.private[*].id }
```

```hcl
# Root main.tf — Module use karo
module "vpc" {
  source = "./modules/vpc"
  # ya
  source = "git::https://github.com/my-org/terraform-modules.git//vpc?ref=v1.2.0"
  
  vpc_cidr    = "10.0.0.0/16"
  environment = var.environment
  az_count    = 3
}

module "eks" {
  source = "./modules/eks"
  
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids
  cluster_name    = "${var.environment}-cluster"
}

# Output use karo
output "eks_endpoint" {
  value = module.eks.cluster_endpoint
}
```

---

## TF-Q4. "State Management — Remote backend aur State locking."

```hcl
# S3 backend + DynamoDB locking setup

# Backend resources pehle banao (manually ya separate terraform)
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-company-terraform-state"
  
  lifecycle {
    prevent_destroy = true   # Accidental delete se bachao
  }
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"   # State history rakho
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

```bash
# State operations
terraform state list                        # Resources list
terraform state show aws_instance.app       # Resource details
terraform state mv aws_instance.app aws_instance.web  # Rename
terraform state rm aws_instance.old         # State se remove (destroy nahi)

# Import existing resource
terraform import aws_instance.app i-1234567890

# State pull (remote se download)
terraform state pull > state.json

# Multiple state files — Workspaces
terraform workspace new staging
terraform workspace select production
terraform workspace list

# Variables per workspace
variable "instance_count" {
  default = {
    default    = 1
    staging    = 2
    production = 5
  }
}

resource "aws_instance" "app" {
  count = var.instance_count[terraform.workspace]
}
```

---

## TF-Q5. "count aur for_each — Multiple resources create karo."

```hcl
# count — Simple numeric repetition
resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}

# Access: aws_subnet.public[0].id

# for_each — Map ya Set (recommended over count)
resource "aws_iam_user" "team" {
  for_each = toset(["alice", "bob", "charlie"])
  name     = each.value
}

# Map se
variable "environments" {
  default = {
    staging    = { instance_type = "t3.small", replicas = 2 }
    production = { instance_type = "m5.large", replicas = 5 }
  }
}

resource "aws_autoscaling_group" "env" {
  for_each      = var.environments
  name          = "${each.key}-asg"
  min_size      = each.value.replicas
  
  # each.key = "staging" ya "production"
  # each.value = {instance_type = ..., replicas = ...}
}

# Access: aws_autoscaling_group.env["production"]

# Dynamic blocks
resource "aws_security_group" "app" {
  name = "app-sg"
  
  dynamic "ingress" {
    for_each = var.ingress_rules   # List of rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

---

## TF-Q6. "Lifecycle — Resource replacement control."

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.app.id
  instance_type = var.instance_type
  
  lifecycle {
    # Naya banao pehle, phir purana delete karo (zero downtime)
    create_before_destroy = true
    
    # Accidental destroy se bachao
    prevent_destroy = true
    
    # AMI change hone pe ignore karo (drift allow)
    ignore_changes = [ami, tags["LastUpdated"]]
    
    # Replace condition (Terraform 1.2+)
    replace_triggered_by = [
      aws_launch_template.app.latest_version
    ]
  }
}

# Null resource — Provisioners ke liye
resource "null_resource" "db_migration" {
  triggers = {
    # Code change hone pe re-run karo
    migration_sha = sha256(file("migrations/v2.sql"))
  }
  
  provisioner "local-exec" {
    command = "psql ${var.db_url} -f migrations/v2.sql"
  }
  
  depends_on = [aws_db_instance.main]
}
```

---

## TF-Q7. "Workspaces aur Multiple Environments."

```hcl
# Approach 1: Workspaces
# terraform workspace new production

locals {
  env_config = {
    development = {
      instance_type = "t3.micro"
      min_capacity  = 1
      max_capacity  = 3
    }
    staging = {
      instance_type = "t3.small"
      min_capacity  = 2
      max_capacity  = 5
    }
    production = {
      instance_type = "m5.large"
      min_capacity  = 3
      max_capacity  = 20
    }
  }
  
  config = local.env_config[terraform.workspace]
}

resource "aws_instance" "app" {
  instance_type = local.config.instance_type
}

# Approach 2: Directory structure (better for large teams)
# ├── modules/
# │   ├── vpc/
# │   ├── eks/
# │   └── rds/
# ├── environments/
# │   ├── development/
# │   │   ├── main.tf
# │   │   └── terraform.tfvars
# │   ├── staging/
# │   └── production/

# terraform.tfvars (environment specific)
environment    = "production"
instance_count = 5
instance_type  = "m5.large"
db_instance    = "db.r6g.large"
```

---

## TF-Q8. "Terraform mein Conditionals aur Expressions."

```hcl
# Conditional expression
resource "aws_instance" "bastion" {
  count         = var.enable_bastion ? 1 : 0
  instance_type = "t3.micro"
}

# Conditional value
variable "environment" {}

locals {
  is_prod      = var.environment == "production"
  instance_type = local.is_prod ? "m5.large" : "t3.small"
  
  # Ternary chaining
  db_class = (
    var.environment == "production" ? "db.r6g.xlarge" :
    var.environment == "staging"    ? "db.t3.medium"  :
                                      "db.t3.micro"
  )
}

# String interpolation
resource "aws_s3_bucket" "logs" {
  bucket = "${var.project}-${var.environment}-logs-${data.aws_caller_identity.current.account_id}"
}

# Functions
locals {
  # List operations
  private_subnet_ids = tolist(aws_subnet.private[*].id)
  subnet_count       = length(aws_subnet.private)
  
  # Map operations
  common_tags = merge(var.tags, {
    Environment = var.environment
    ManagedBy   = "Terraform"
  })
  
  # String functions
  cluster_name = lower(replace("${var.project}-${var.environment}", " ", "-"))
  
  # JSON
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject"]
      Resource = "${aws_s3_bucket.main.arn}/*"
    }]
  })
}
```

---

## TF-Q9. "Terraform CI/CD — GitHub Actions mein automate karo."

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [main]
    paths: ['infrastructure/**']
  pull_request:
    paths: ['infrastructure/**']

env:
  TF_VAR_environment: production
  AWS_REGION: ap-south-1

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infrastructure/

    steps:
    - uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: 1.7.0

    - name: Configure AWS
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Terraform Init
      run: terraform init

    - name: Terraform Validate
      run: terraform validate

    - name: Terraform Format Check
      run: terraform fmt -check -recursive

    - name: Terraform Plan
      id: plan
      run: terraform plan -out=tfplan -no-color
      continue-on-error: true   # Plan fail hone pe bhi PR comment karo

    - name: PR Comment — Plan Output
      if: github.event_name == 'pull_request'
      uses: actions/github-script@v7
      with:
        script: |
          const output = `#### Plan: ${{ steps.plan.outcome }}
          \`\`\`
          ${{ steps.plan.outputs.stdout }}
          \`\`\``;
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: output
          })

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && steps.plan.outcome == 'success'
      run: terraform apply -auto-approve tfplan
```

---

## TF-Q10. "Terraform Security — tfsec aur Checkov."

```bash
# tfsec — Security scanner
docker run --rm -v "$(pwd):/src" aquasec/tfsec /src

# Checkov — Comprehensive policy checks
pip install checkov
checkov -d . --framework terraform

# Common issues tfsec detect karta hai:
# - S3 bucket public access enabled
# - Security group 0.0.0.0/0 ssh open
# - RDS publicly accessible
# - EBS not encrypted
# - CloudTrail not enabled

# Ignore specific checks
resource "aws_s3_bucket" "public_assets" {
  #tfsec:ignore:aws-s3-block-public-acls
  bucket = "my-public-website-assets"
}

# OPA / Sentinel policies (Terraform Cloud)
# Custom policy as code
```

---

## TF-Q11. "Terraform Import — Existing resources import karo."

```bash
# Existing resource ko Terraform state mein lao

# Step 1 — Resource config likho
# main.tf
resource "aws_instance" "existing_server" {
  # Attributes pehle se pata nahi — terraform import ke baad fill karo
}

# Step 2 — Import karo
terraform import aws_instance.existing_server i-0123456789abcdef0

# Step 3 — State se actual values dekho
terraform state show aws_instance.existing_server

# Step 4 — Config update karo exact values se

# Terraform 1.5+ — Import block (better approach)
import {
  id = "i-0123456789abcdef0"
  to = aws_instance.existing_server
}

# Terraform 1.5+ — Generate config automatically
terraform plan -generate-config-out=generated.tf

# Bulk import — multiple resources
# S3 buckets
for bucket in $(aws s3 ls | awk '{print $3}'); do
    terraform import "aws_s3_bucket.${bucket//-/_}" "$bucket"
done
```

---

## TF-Q12. "Terragrunt — DRY configurations."

```hcl
# terragrunt.hcl — Root (shared config)
remote_state {
  backend = "s3"
  config = {
    bucket         = "my-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# Directory structure:
# ├── terragrunt.hcl (root)
# ├── production/
# │   ├── vpc/
# │   │   └── terragrunt.hcl
# │   ├── eks/
# │   │   └── terragrunt.hcl
# │   └── rds/
# │       └── terragrunt.hcl

# production/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../modules//vpc"
}

inputs = {
  vpc_cidr    = "10.0.0.0/16"
  environment = "production"
}

# production/eks/terragrunt.hcl
dependency "vpc" {
  config_path = "../vpc"
}

inputs = {
  vpc_id     = dependency.vpc.outputs.vpc_id
  subnet_ids = dependency.vpc.outputs.private_subnet_ids
}
```

```bash
# Terragrunt commands
terragrunt init
terragrunt plan
terragrunt apply

# Sab ek saath (dependency order)
terragrunt run-all plan
terragrunt run-all apply
```

---

## TF-Q13. "Terraform Troubleshooting — Common errors fix karo."

```bash
# Error 1: State lock
# Error: Error acquiring the state lock
terraform force-unlock <lock-id>

# Error 2: Resource already exists
# Error: resource already exists in AWS but not in state
terraform import <resource_type>.<name> <id>

# Error 3: Provider version conflict
terraform init -upgrade

# Error 4: Module not found
terraform get          # Modules download karo
terraform init         # Ya init

# Error 5: Sensitive variable in plan
# Solution: sensitive = true variable mein

# Debug mode
TF_LOG=DEBUG terraform plan 2>&1 | tee debug.log
TF_LOG=TRACE terraform apply

# Refresh state (drift detect)
terraform refresh

# Plan only specific resource
terraform plan -target=aws_instance.app

# Apply specific resource
terraform apply -target=module.vpc

# State manipulation
terraform state pull > backup.tfstate    # Backup
terraform state push backup.tfstate      # Restore

# Graph — dependency visualization
terraform graph | dot -Tpng > graph.png
```

---

## TF-Q14. "Terraform Cloud / Enterprise features."

```hcl
# Terraform Cloud backend
terraform {
  cloud {
    organization = "my-company"
    
    workspaces {
      name = "production"
      # ya
      tags = ["production", "aws"]
    }
  }
}
```

```
Terraform Cloud Features:
├── Remote State Management (free)
├── Remote Plan/Apply (runs in cloud)
├── Sentinel Policies (Enterprise)
│   ├── Policy as Code
│   ├── Enforce tagging standards
│   └── Block unsafe changes
├── Audit Logging
├── SSO Integration
├── Cost Estimation
└── VCS Integration (GitHub, GitLab)

Benefits over local:
├── State locking automatic
├── Sensitive variables encrypted
├── Team collaboration
├── Plan history
└── Approval workflows
```

---

## TF-Q15. "Complete Production Terraform Structure."

```
terraform-infrastructure/
├── modules/              # Reusable components
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── eks/
│   ├── rds/
│   ├── s3/
│   └── iam/
│
├── environments/
│   ├── development/
│   │   ├── main.tf           # Module calls
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── backend.tf        # Remote state config
│   │   └── terraform.tfvars  # Env specific values
│   ├── staging/
│   └── production/
│
├── .github/
│   └── workflows/
│       ├── terraform-plan.yml   # PR pe plan
│       └── terraform-apply.yml  # Main pe apply
│
├── .tflint.hcl                  # Linting config
├── .pre-commit-config.yaml      # Pre-commit hooks
└── Makefile                     # Common commands
```

```makefile
# Makefile
ENV ?= development

init:
	cd environments/$(ENV) && terraform init

plan:
	cd environments/$(ENV) && terraform plan -var-file=terraform.tfvars

apply:
	cd environments/$(ENV) && terraform apply -var-file=terraform.tfvars

destroy:
	cd environments/$(ENV) && terraform destroy -var-file=terraform.tfvars

fmt:
	terraform fmt -recursive

validate:
	terraform validate

scan:
	tfsec .
	checkov -d .

docs:
	terraform-docs markdown . > README.md
```

---

## 📌 Terraform Quick Reference

```
Resource Block:
resource "<provider>_<type>" "<name>" {
  argument = value
}

Data Block:
data "<provider>_<type>" "<name>" {
  filter { ... }
}

Variable Types:
string, number, bool, list(type), map(type), set(type), object({...})

Important Functions:
toset()    tolist()    tomap()
length()   lookup()    merge()
join()     split()     replace()
format()   jsonencode() jsondecode()
cidrsubnet() file()    sha256()
flatten()  distinct()  concat()
```

---

*Python+Shell: 20Q | Kubernetes: 25Q | ArgoCD: 12Q | Observability: 12Q | AWS: 25Q | Terraform: 15Q*
*Total: 109 Production Interview Questions — Complete DevOps Preparation*
