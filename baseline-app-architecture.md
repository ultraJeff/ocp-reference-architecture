# Baseline Application Architecture on OpenShift

> A foundational reference architecture for deploying a "well-architected" application on OpenShift Container Platform. This baseline covers the common components most applications need — UI, APIs, data, observability, security, and CI/CD — before any application-specific customization.

---

## Architecture Overview

```
                        ┌──────────────────────────────────────────────────┐
                        │              OpenShift Cluster                   │
                        │                                                  │
   End Users ──────►    │  ┌─────────────────────────────────────────┐     │
                        │  │           Ingress (Router/HAProxy)      │     │
                        │  │         TLS termination + routing       │     │
                        │  └────────┬──────────────────┬─────────────┘     │
                        │           │                  │                   │
                        │           ▼                  ▼                   │
                        │  ┌────────────────┐  ┌────────────────┐         │
                        │  │   UI Tier       │  │   API Tier     │         │
                        │  │   (Route)       │  │   (Route)      │         │
                        │  │                 │  │                 │         │
                        │  │  Deployment     │  │  Deployment    │         │
                        │  │  + HPA          │  │  + HPA         │         │
                        │  │  + Service      │  │  + Service     │         │
                        │  └────────┬────────┘  └───┬──────┬─────┘        │
                        │           │               │      │               │
                        │           │               ▼      ▼               │
                        │           │       ┌──────────────────────┐       │
                        │           │       │   Data Tier           │       │
                        │           │       │                       │       │
                        │           │       │  StatefulSet / CRD    │       │
                        │           │       │  + PVC (RWO/RWX)      │       │
                        │           │       │  + Service (headless) │       │
                        │           │       └──────────────────────┘       │
                        │                                                  │
                        │  ┌──────────────────────────────────────────┐    │
                        │  │          Platform Services                │    │
                        │  │  Logging | Monitoring | Secrets | GitOps  │    │
                        │  └──────────────────────────────────────────┘    │
                        └──────────────────────────────────────────────────┘
```

---

## 1. Namespace Strategy

| Namespace | Purpose |
|-----------|---------|
| `myapp-dev` | Development environment |
| `myapp-stage` | Staging / QA environment |
| `myapp-prod` | Production environment |

Each namespace is a security and resource boundary with its own:
- ResourceQuotas and LimitRanges
- NetworkPolicies
- RoleBindings
- Secrets / ConfigMaps

---

## 2. Presentation Tier (UI)

The frontend application (SPA, SSR, or static site) served to end users.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-ui
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: ui
          image: registry.example.com/myapp/ui:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-ui
spec:
  selector:
    app: myapp-ui
  ports:
    - port: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp-ui
spec:
  tls:
    termination: edge
  to:
    kind: Service
    name: myapp-ui
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-ui
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-ui
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## 3. API / Business Logic Tier

Backend services that implement business logic and expose APIs.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: api
          image: registry.example.com/myapp/api:latest
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: myapp-db-credentials
                  key: host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: myapp-db-credentials
                  key: password
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-api
spec:
  selector:
    app: myapp-api
  ports:
    - port: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp-api
spec:
  tls:
    termination: edge
  to:
    kind: Service
    name: myapp-api
  path: /api
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## 4. Data Tier

Databases and persistent storage. Two common patterns:

### Option A: Operator-Managed Database (Recommended)

Use a certified Operator from OperatorHub (e.g., Crunchy PostgreSQL, MongoDB Enterprise, Redis Enterprise).

```yaml
# Example: Crunchy PostgreSQL via PGO
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: myapp-db
spec:
  postgresVersion: 16
  instances:
    - replicas: 2
      dataVolumeClaimSpec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 50Gi
        storageClassName: gp3-csi
  backups:
    pgbackrest:
      repos:
        - name: repo1
          volume:
            volumeClaimSpec:
              accessModes: [ReadWriteOnce]
              resources:
                requests:
                  storage: 100Gi
```

### Option B: StatefulSet (Self-Managed)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myapp-db
spec:
  serviceName: myapp-db
  replicas: 1
  template:
    spec:
      containers:
        - name: postgres
          image: registry.redhat.io/rhel9/postgresql-16
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/pgsql/data
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: "2"
              memory: 4Gi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: gp3-csi
        resources:
          requests:
            storage: 50Gi
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-db
spec:
  clusterIP: None  # headless
  selector:
    app: myapp-db
  ports:
    - port: 5432
```

---

## 5. Observability

### 5a. Application Logging

Applications write structured logs to `stdout`/`stderr`. The platform handles collection and forwarding.

```
Application (stdout/stderr)
    └──► CRI-O captures container logs
          └──► ClusterLogForwarder (OpenShift Logging)
                ├──► Internal: LokiStack / Elasticsearch
                └──► External: Splunk, CloudWatch, etc.
```

**Application-level best practice:**
- Use structured logging (JSON)
- Include correlation IDs for request tracing
- Use appropriate log levels (INFO, WARN, ERROR)

```yaml
# ClusterLogForwarder example
apiVersion: observability.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
spec:
  outputs:
    - name: external-splunk
      type: splunk
      splunk:
        indexName: myapp
      url: https://splunk.example.com:8088
  pipelines:
    - name: app-logs
      inputRefs: [application]
      outputRefs: [external-splunk]
      filterRefs: [myapp-filter]
  filters:
    - name: myapp-filter
      type: kubeAPIAudit
```

### 5b. Metrics & Monitoring

```yaml
# ServiceMonitor — tells Prometheus to scrape the app
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-api
spec:
  selector:
    matchLabels:
      app: myapp-api
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
---
# PrometheusRule — define alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
spec:
  groups:
    - name: myapp
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{job="myapp-api",status=~"5.."}[5m]))
            / sum(rate(http_requests_total{job="myapp-api"}[5m])) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High 5xx error rate on myapp-api"
```

### 5c. Distributed Tracing (Optional but Recommended)

Instrument applications with OpenTelemetry SDK. Deploy the OpenTelemetry Collector Operator to collect and export traces to Tempo or Jaeger.

---

## 6. Security Baseline

### 6a. Pod Security

```yaml
# All workloads should target restricted-v2 SCC
# (enforced by default in OCP 4.x)
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: [ALL]
```

### 6b. Network Policies

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
# Allow UI → API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ui-to-api
spec:
  podSelector:
    matchLabels:
      app: myapp-api
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: myapp-ui
      ports:
        - port: 8080
---
# Allow API → Database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
spec:
  podSelector:
    matchLabels:
      app: myapp-db
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: myapp-api
      ports:
        - port: 5432
---
# Allow ingress from Router
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-router
spec:
  podSelector:
    matchLabels:
      app: myapp-ui
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              network.openshift.io/policy-group: ingress
```

### 6c. Secrets Management

```yaml
# Option A: ExternalSecret (recommended for production)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: myapp-db-credentials
  data:
    - secretKey: host
      remoteRef:
        key: myapp/database
        property: host
    - secretKey: password
      remoteRef:
        key: myapp/database
        property: password
```

---

## 7. CI/CD Pipeline

```
 Developer          Git Repo           CI (Tekton/GH Actions)        OCP
 ────────           ────────           ──────────────────────        ───
    │                  │                        │                      │
    ├── push ─────────►│                        │                      │
    │                  ├── webhook ─────────────►│                      │
    │                  │                        ├── lint + test         │
    │                  │                        ├── build image         │
    │                  │                        ├── scan image (ACS)    │
    │                  │                        ├── push to registry    │
    │                  │                        ├── update GitOps repo  │
    │                  │                        │                      │
    │                  │            GitOps Repo (ArgoCD watches)       │
    │                  │                        │                      │
    │                  │                        └── ArgoCD sync ──────►│
    │                  │                                               ├── deploy
    │                  │                                               ├── smoke test
    │                  │                                               └── done
```

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/example/myapp-gitops.git
    targetRevision: main
    path: overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 8. Kustomize Overlay Structure

```
myapp-gitops/
├── base/
│   ├── kustomization.yaml
│   ├── deployment-ui.yaml
│   ├── deployment-api.yaml
│   ├── service-ui.yaml
│   ├── service-api.yaml
│   ├── route-ui.yaml
│   ├── route-api.yaml
│   ├── hpa-ui.yaml
│   ├── hpa-api.yaml
│   ├── networkpolicy-default-deny.yaml
│   ├── networkpolicy-allow-ui-api.yaml
│   ├── networkpolicy-allow-api-db.yaml
│   └── servicemonitor-api.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml    # lower resource limits, 1 replica
│   │   └── patches/
│   ├── stage/
│   │   ├── kustomization.yaml    # production-like, fewer replicas
│   │   └── patches/
│   └── prod/
│       ├── kustomization.yaml    # full resources, HA replicas
│       └── patches/
```

---

## 9. Extension Points

This baseline is designed to be extended. Common additions based on application needs:

| Need | Add |
|------|-----|
| Event-driven processing | AMQ Streams (Kafka) + KafkaTopic CRs |
| Async job processing | Kubernetes Jobs / CronJobs |
| Inter-service auth + observability | OpenShift Service Mesh (Istio) |
| Auto-scaling on custom metrics | KEDA + ScaledObject |
| Serverless / scale-to-zero | OpenShift Serverless (Knative) |
| API management | 3scale or API gateway |
| Caching layer | Redis (Operator-managed) |
| File/object storage | ODF (OpenShift Data Foundation) or S3 |
| ML/AI inference | RHOAI / KServe |
| Multi-cluster deployment | ACM + ArgoCD ApplicationSets |

---

## 10. Summary: What Every App Gets

Every application deployed on this platform inherits:

- **TLS everywhere** — Routes terminate TLS; internal traffic can be encrypted via Service Mesh
- **Identity & access** — OAuth proxy or OIDC integration via OpenShift's built-in OAuth server
- **Autoscaling** — HPA on CPU/memory; KEDA for custom metrics
- **Self-healing** — Kubernetes restarts failed pods; liveness/readiness probes detect issues
- **Centralized logging** — stdout/stderr captured and forwarded automatically
- **Metrics & alerting** — Prometheus scrapes via ServiceMonitor; alerts via PrometheusRule
- **GitOps deployment** — ArgoCD ensures desired state matches actual state
- **Network segmentation** — NetworkPolicies enforce least-privilege communication
- **Image security** — ACS/Quay scan images before deployment
- **Secrets rotation** — ExternalSecrets sync from Vault/AWS Secrets Manager
- **Resource governance** — Quotas and limits prevent noisy-neighbor issues
