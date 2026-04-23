# Application Modernization Standards & Cloud-Native Best Practices for OpenShift

> This document defines the standards, policies, and architectural patterns for deploying well-architected applications on Red Hat OpenShift Container Platform. It is prescriptive by design: **MUST** requirements are non-negotiable for production workloads, **SHOULD** items are strongly recommended, and **MAY** items are available when justified by application needs.

---

## 1. Application Readiness Requirements

Before an application is deployed to OpenShift, it **MUST** meet these baseline requirements. These are not suggestions — they are prerequisites for platform onboarding.

### 1.1 Container-Native Design (MUST)

| Requirement | Standard | Why |
|-------------|----------|-----|
| Single process per container | Each container runs one process | Enables independent scaling, restart isolation, and clear resource attribution |
| Non-root execution | Containers MUST run as non-root (UID ≥ 1000) | OpenShift enforces `restricted-v2` SCC by default; root containers will be rejected |
| No host dependencies | No hostPath volumes, host networking, or privileged access | Ensures workload portability and cluster security |
| Immutable images | Application code is baked into the image at build time, not mounted or downloaded at runtime | Guarantees reproducibility and auditability across environments |
| No hardcoded configuration | All environment-specific values (URLs, credentials, feature flags) are injected via ConfigMaps, Secrets, or environment variables | Enables promotion across dev → stage → prod without image rebuilds |

### 1.2 Health & Lifecycle (MUST)

Every application component **MUST** implement the following endpoints:

| Endpoint | Purpose | Implementation |
|----------|---------|----------------|
| **Readiness probe** (`/ready`) | Signals the pod is ready to accept traffic | Returns 200 when the app can serve requests. Return 503 during startup or when downstream dependencies are unavailable |
| **Liveness probe** (`/healthz`) | Signals the process is alive and not deadlocked | Returns 200 if the process is running. Do NOT check downstream dependencies here — that causes cascading restarts |
| **Startup probe** (if needed) | Protects slow-starting apps from premature liveness kills | Use `failureThreshold × periodSeconds` to define a startup budget |
| **Graceful shutdown** | Handles `SIGTERM` cleanly | Drain in-flight requests, close DB connections, flush buffers. Default grace period: 30s |

### 1.3 Observability Contract (MUST)

Applications are not production-ready until they are observable. Every component **MUST** fulfill this contract:

**Logging:**
- Write all logs to `stdout`/`stderr` — never to files inside the container
- Use structured format (JSON) with fields: `timestamp`, `level`, `message`, `correlationId`
- Use standard log levels: `DEBUG`, `INFO`, `WARN`, `ERROR`
- Include a correlation/request ID that propagates across service calls

**Metrics:**
- Expose a Prometheus-compatible `/metrics` endpoint
- At minimum, expose: request count, request latency (histogram), error rate, and saturation (active connections/threads)
- Follow the [RED method](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/): Rate, Errors, Duration

**Tracing (SHOULD):**
- Instrument with OpenTelemetry SDK
- Propagate trace context headers (`traceparent`, `tracestate`) across service boundaries

### 1.4 Configuration & Secrets (MUST)

| Policy | Standard |
|--------|----------|
| Configuration injection | All config MUST be injected via environment variables or mounted ConfigMaps. No config files baked into images |
| Secrets management | Production secrets MUST be sourced from an external secrets manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) via the External Secrets Operator. Kubernetes Secrets objects are acceptable only in dev/stage |
| No secrets in images | Container images MUST NOT contain credentials, API keys, certificates, or connection strings |
| No secrets in Git | No secrets in source repositories. Use SealedSecrets or ExternalSecrets for GitOps workflows |

---

## 2. Platform-Provided Capabilities

The platform provides these capabilities out of the box. Application teams **MUST NOT** reimplement them — doing so creates operational burden, security gaps, and supportability issues.

### What the Platform Gives You — Use It

| Capability | Platform Component | App Team Responsibility |
|------------|--------------------|------------------------|
| **TLS termination** | OpenShift Router (HAProxy) | Create a Route with `tls.termination: edge` or `reencrypt`. Do not manage your own certs at the app level |
| **Log aggregation** | OpenShift Logging (ClusterLogForwarder → Loki/Splunk) | Write structured logs to stdout. The platform collects and forwards them |
| **Metrics collection** | OpenShift Monitoring (Prometheus) | Expose `/metrics` endpoint, deploy a ServiceMonitor. The platform scrapes and stores |
| **Alerting** | Prometheus AlertManager | Define PrometheusRules for application-specific alerts. Platform routes them |
| **Container registry** | Quay / OpenShift Internal Registry | Push images to the approved registry. The platform handles scanning and vulnerability detection |
| **Image scanning** | Red Hat Advanced Cluster Security (ACS) | Images are scanned automatically in the CI pipeline and at admission. Fix flagged CVEs |
| **Identity & auth** | OpenShift OAuth / OIDC provider integration | Use the platform's identity provider. Do not build custom auth stacks |
| **Secrets sync** | External Secrets Operator | Declare ExternalSecret CRs. The platform syncs from Vault/cloud provider |
| **GitOps deployment** | OpenShift GitOps (ArgoCD) | Maintain a GitOps repo with Kustomize overlays. ArgoCD handles sync |
| **DNS & service discovery** | CoreDNS / Kubernetes Services | Use internal Service DNS names (`myapp-api.myapp-prod.svc.cluster.local`). Do not hardcode IPs |
| **Resource governance** | ResourceQuotas, LimitRanges | Set resource requests and limits on every container. Quotas are enforced at the namespace level |

### What the Platform Does NOT Provide (App Team Owns)

| Concern | App Team Must Handle |
|---------|---------------------|
| Application business logic | Obviously |
| Data model & schema management | Migrations, schema versioning, backward compatibility |
| API contracts | OpenAPI specs, versioning strategy, deprecation policy |
| Application-level caching | Redis, in-memory — chosen per app needs |
| Async/event processing | Kafka consumers, queue workers — declared as additional Deployments |
| Performance testing | Load test before production promotion |

---

## 3. Architecture Standards by Tier

### 3.1 Presentation Tier (UI)

| Standard | Requirement Level |
|----------|-------------------|
| Serve static assets via a lightweight HTTP server (nginx, Caddy) or SSR runtime (Node.js) | MUST |
| Externalize API base URLs via environment variables injected at build or runtime | MUST |
| Set minimum 2 replicas in production | MUST |
| Configure HPA with CPU target ≤ 70% | SHOULD |
| Use a Route with TLS edge termination | MUST |
| Return a meaningful `/ready` and `/healthz` | MUST |

### 3.2 API / Business Logic Tier

| Standard | Requirement Level |
|----------|-------------------|
| Stateless design — no local session state, no local file storage for runtime data | MUST |
| Expose a RESTful or gRPC API with versioning (URI path or header) | MUST |
| Connect to databases and external services via injected configuration (never hardcoded) | MUST |
| Set minimum 2 replicas in production | MUST |
| Configure HPA with CPU target ≤ 70% | SHOULD |
| Implement circuit breakers for downstream calls | SHOULD |
| Expose `/metrics` with RED metrics (rate, errors, duration) | MUST |
| Implement request timeouts and retry budgets | SHOULD |
| Use an internal Service (ClusterIP) for intra-cluster traffic; Route only for external-facing APIs | MUST |

### 3.3 Data Tier

| Standard | Requirement Level |
|----------|-------------------|
| Use an Operator-managed database (Crunchy PGO, MongoDB Enterprise Operator, Redis Enterprise Operator) | MUST for production |
| Self-managed StatefulSets are acceptable in dev/stage only | MAY |
| Configure automated backups with a tested restore procedure | MUST |
| Use a dedicated StorageClass with appropriate IOPS and reclaim policy (`Retain` for prod) | MUST |
| Database credentials sourced from External Secrets Operator | MUST |
| Run schema migrations as Kubernetes Jobs, not as part of application startup | SHOULD |
| Use headless Services for StatefulSet DNS | MUST |

---

## 4. Security Standards

These are non-negotiable. Exceptions require documented approval from the security team.

### 4.1 Pod Security (MUST)

All workloads MUST comply with the `restricted-v2` Security Context Constraint:

- `runAsNonRoot: true`
- `allowPrivilegeEscalation: false`
- `capabilities.drop: [ALL]`
- `seccompProfile.type: RuntimeDefault`

Any request for elevated SCCs (`anyuid`, `privileged`) requires a written exception with justification, time-bound approval, and a remediation plan.

### 4.2 Network Segmentation (MUST)

Every application namespace MUST deploy NetworkPolicies:

1. **Default deny all ingress** — baseline policy, no exceptions
2. **Explicit allow rules** — whitelist traffic between tiers:
   - Router → UI (from `network.openshift.io/policy-group: ingress` namespace)
   - UI → API (pod-to-pod within namespace)
   - API → Database (pod-to-pod within namespace)
   - API → external services (egress rules as needed)
3. **No open namespaces** — a namespace without NetworkPolicies will not pass security review

### 4.3 Image Provenance (MUST)

- Images MUST be pulled from an approved registry (Quay, OpenShift internal registry, Red Hat registry)
- Images MUST be built in a CI pipeline — no `docker build` on developer laptops pushed to prod
- Images MUST pass ACS vulnerability scanning before deployment
- Base images MUST use Red Hat UBI (Universal Base Image) or an approved alternative
- Image tags MUST use immutable references (SHA digests) in production GitOps manifests, not `:latest`

### 4.4 RBAC (MUST)

- Application ServiceAccounts MUST follow least-privilege — only the permissions the workload actually needs
- Do not use `cluster-admin` or `admin` ClusterRoles for application workloads
- Namespace-scoped Roles are preferred over ClusterRoles

---

## 5. CI/CD & GitOps Standards

### 5.1 Pipeline Requirements (MUST)

Every application MUST have an automated CI/CD pipeline. Manual deployments to staging or production are not permitted.

**CI pipeline stages (in order):**

```
1. Code checkout
2. Lint & static analysis
3. Unit tests
4. Build container image
5. Image vulnerability scan (ACS/Quay)
6. Push image to approved registry
7. Update GitOps repo with new image digest
```

**CD pipeline (GitOps):**

```
8.  ArgoCD detects change in GitOps repo
9.  ArgoCD syncs manifests to target namespace
10. Post-deploy smoke test / health check validation
```

### 5.2 GitOps Repository Structure (MUST)

All application deployments MUST be managed via a GitOps repository. ArgoCD is the standard deployment tool.

```
<app>-gitops/
├── base/                          # Shared manifests (Deployments, Services, etc.)
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
│   ├── servicemonitor-api.yaml
│   └── prometheusrule-alerts.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml     # Lower resources, 1 replica, relaxed probes
│   │   └── patches/
│   ├── stage/
│   │   ├── kustomization.yaml     # Production-like config, fewer replicas
│   │   └── patches/
│   └── prod/
│       ├── kustomization.yaml     # Full HA, strict resources, SHA image refs
│       └── patches/
```

### 5.3 Environment Promotion (MUST)

- **Dev → Stage**: Automated on merge to `main`. Image digest propagated via CI pipeline
- **Stage → Prod**: Requires manual approval gate (PR to `overlays/prod` in GitOps repo)
- Environment-specific configuration MUST live in Kustomize patches, not in the application code or image

---

## 6. Modernization Maturity Model

Not every application starts cloud-native. Use this maturity model to assess where an application is today and define a concrete path forward.

### Level 1: Containerized (Minimum Viable)

The application runs in a container on OpenShift but has not been redesigned.

| Characteristic | Status |
|----------------|--------|
| Runs in a container | Yes |
| Non-root execution | Yes (required by platform) |
| Health probes | Basic (TCP or HTTP) |
| Logging | May still write to files — needs stdout migration |
| Config | May still use embedded config files |
| State | May be stateful (session affinity, local files) |
| Scaling | Single replica, manual |
| Deployment | May be manual or semi-automated |

**What to focus on next:** Externalize configuration, migrate logging to stdout, implement proper health endpoints.

### Level 2: Cloud-Ready

The application meets all MUST requirements in this document and can be deployed reliably via GitOps.

| Characteristic | Status |
|----------------|--------|
| Runs in a container | Yes |
| Non-root, restricted SCC | Yes |
| Health probes | Readiness + liveness, meaningful checks |
| Logging | Structured JSON to stdout |
| Config | Externalized via ConfigMap/Secret |
| State | Stateless application tier, external data stores |
| Scaling | HPA-enabled, minimum 2 replicas |
| Deployment | Fully automated via GitOps |
| Metrics | `/metrics` exposed, ServiceMonitor deployed |
| Security | NetworkPolicies, ExternalSecrets, image scanning |

**What to focus on next:** Decompose monolithic components, add distributed tracing, implement circuit breakers.

### Level 3: Cloud-Native

The application is fully decomposed, independently deployable, and takes full advantage of the platform.

| Characteristic | Status |
|----------------|--------|
| All Level 2 requirements | Yes |
| Architecture | Microservices or well-bounded modular services |
| Communication | Async where appropriate (Kafka, AMQP), sync via well-defined APIs |
| Scaling | Custom metric scaling (KEDA), scale-to-zero where applicable |
| Resilience | Circuit breakers, retry budgets, bulkheads, graceful degradation |
| Tracing | Full distributed tracing via OpenTelemetry |
| Deployment | Canary or blue-green release strategies |
| API management | Versioned APIs, OpenAPI specs, deprecation lifecycle |
| Multi-environment | Consistent deployment across clusters via ApplicationSets |

---

## 7. Extension Catalog

These components are available when application requirements justify them. They are not part of the baseline — each adds operational complexity and should be adopted deliberately.

| Need | Solution | When to Adopt | Requirement Level |
|------|----------|---------------|-------------------|
| Event-driven processing | AMQ Streams (Kafka) | App needs async decoupling, event sourcing, or CDC | SHOULD for event-driven apps |
| Async job processing | Kubernetes Jobs / CronJobs | Batch processing, scheduled tasks, data pipelines | SHOULD for batch workloads |
| Custom metric autoscaling | KEDA | HPA on CPU/memory is insufficient; need to scale on queue depth, custom metrics | MAY |
| Scale-to-zero | OpenShift Serverless (Knative) | Bursty or infrequent workloads where idle cost matters | MAY |
| Inter-service traffic management | OpenShift Service Mesh (Istio) | Need mTLS, traffic shifting, canary rollouts, or cross-cluster service discovery | MAY |
| API gateway | 3scale / Kong | External API consumers need rate limiting, key management, analytics | SHOULD for public APIs |
| Caching | Redis (Operator-managed) | High-read, low-write data patterns; session caching | MAY |
| Object storage | ODF or S3 | Files, media, backups, ML artifacts | Use when app has unstructured data |
| ML/AI inference | RHOAI / KServe | Model serving at scale | Use for AI/ML workloads |
| Multi-cluster | ACM + ArgoCD ApplicationSets | DR, geo-distribution, blast radius isolation | SHOULD for critical workloads |

---

## 8. Baseline Architecture Diagram

Every well-architected application on this platform follows this foundational pattern. Extensions from the catalog above are layered on top.

```
                        ┌────────────────────────────────────────────────────────┐
                        │                  OpenShift Cluster                      │
                        │                                                        │
                        │  ┌──────────────────────────────────────────────────┐  │
   End Users ──────►    │  │          Ingress (Router / HAProxy)              │  │
                        │  │      TLS termination · Route-based routing       │  │
                        │  └──────────┬───────────────────────┬───────────────┘  │
                        │             │                       │                  │
                        │             ▼                       ▼                  │
                        │  ┌──────────────────┐    ┌──────────────────┐         │
                        │  │ Presentation Tier │    │   API Tier        │         │
                        │  │                   │    │                   │         │
                        │  │ Deployment (≥2)   │    │ Deployment (≥2)   │         │
                        │  │ Service + Route   │    │ Service + Route   │         │
                        │  │ HPA (CPU ≤70%)    │    │ HPA (CPU ≤70%)    │         │
                        │  │ /healthz + /ready │    │ /healthz + /ready │         │
                        │  │ /metrics          │    │ /metrics          │         │
                        │  └──────────────────┘    └────┬──────┬───────┘         │
                        │                               │      │                 │
                        │                   ┌───────────┘      └──────────┐      │
                        │                   ▼                             ▼      │
                        │  ┌──────────────────────┐    ┌────────────────────┐    │
                        │  │   Data Tier            │    │  External Services │    │
                        │  │                        │    │  (via NetworkPolicy │    │
                        │  │  Operator-managed DB   │    │   egress rules)    │    │
                        │  │  Automated backups     │    └────────────────────┘    │
                        │  │  PVC (Retain policy)   │                              │
                        │  └──────────────────────┘                              │
                        │                                                        │
                        │  ┌──────────────────────────────────────────────────┐  │
                        │  │               Platform Services                   │  │
                        │  │                                                   │  │
                        │  │  Logging (stdout → ClusterLogForwarder → Loki)    │  │
                        │  │  Monitoring (ServiceMonitor → Prometheus)          │  │
                        │  │  Alerting (PrometheusRule → AlertManager)          │  │
                        │  │  Secrets (ExternalSecret → Vault)                 │  │
                        │  │  Deployment (ArgoCD → GitOps repo)                │  │
                        │  │  Scanning (ACS → Quay)                            │  │
                        │  │  Network (default-deny + explicit allow)           │  │
                        │  └──────────────────────────────────────────────────┘  │
                        └────────────────────────────────────────────────────────┘

                        ┌────────────────────────────────────────────────────────┐
                        │                   CI/CD Pipeline                        │
                        │                                                        │
                        │  Git push → Lint → Test → Build → Scan → Push image   │
                        │  → Update GitOps repo → ArgoCD sync → Smoke test      │
                        └────────────────────────────────────────────────────────┘
```

---

## 9. Governance & Compliance Checklist

Use this checklist during application onboarding and periodic reviews.

### Pre-Deployment Gate (MUST pass all)

- [ ] Container runs as non-root with `restricted-v2` SCC
- [ ] Readiness and liveness probes implemented and tested
- [ ] Structured logging to stdout (JSON with correlation IDs)
- [ ] `/metrics` endpoint exposed with RED metrics
- [ ] ServiceMonitor and PrometheusRule deployed
- [ ] All config externalized (no hardcoded URLs, credentials, or feature flags)
- [ ] Secrets sourced from External Secrets Operator (prod) or Kubernetes Secrets (dev/stage)
- [ ] No secrets in container images or Git repositories
- [ ] NetworkPolicies deployed (default-deny + explicit allow)
- [ ] Images built in CI pipeline from approved base images (UBI)
- [ ] Images scanned by ACS with no critical/high unpatched CVEs
- [ ] Production image references use SHA digests, not mutable tags
- [ ] GitOps repo with Kustomize overlays for all target environments
- [ ] ArgoCD Application configured with `selfHeal` and `prune` enabled
- [ ] Minimum 2 replicas in production with HPA configured
- [ ] Resource requests and limits set on all containers
- [ ] Graceful shutdown handler implemented (handles SIGTERM)
- [ ] Backup and restore procedure documented and tested (data tier)

### Post-Deployment Verification (SHOULD)

- [ ] Load test completed against staging environment
- [ ] Alerts fire correctly under simulated failure conditions
- [ ] Runbook exists for common failure scenarios
- [ ] On-call team trained on application architecture and failure modes
