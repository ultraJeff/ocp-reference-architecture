---
layout: default
title: Application Readiness & Platform Capabilities
nav_order: 2
---

# Application Readiness & Platform Capabilities

## Application Readiness Requirements

Before an application is deployed to OpenShift, it meets these baseline requirements. These are not suggestions — they are prerequisites for platform onboarding.

### Container-Native Design

| Requirement | Standard | Why |
|-------------|----------|-----|
| Single process per container | Each container runs one process | Enables independent scaling, restart isolation, and clear resource attribution |
| Non-root execution | Containers run as non-root (UID ≥ 1000) | OpenShift enforces `restricted-v2` SCC by default; root containers will be rejected |
| No host dependencies | No hostPath volumes, host networking, or privileged access | Ensures workload portability and cluster security |
| Immutable images | Application code is baked into the image at build time, not mounted or downloaded at runtime | Guarantees reproducibility and auditability across environments |
| No hardcoded configuration | All environment-specific values (URLs, credentials, feature flags) are injected via ConfigMaps, Secrets, or environment variables | Enables promotion across dev → stage → prod without image rebuilds |

### Health & Lifecycle

Every application component implements the following endpoints:

| Endpoint | Purpose | Implementation |
|----------|---------|----------------|
| **Readiness probe** (`/ready`) | Signals the pod is ready to accept traffic | Returns 200 when the app can serve requests. Return 503 during startup or when downstream dependencies are unavailable |
| **Liveness probe** (`/healthz`) | Signals the process is alive and not deadlocked | Returns 200 if the process is running. Do NOT check downstream dependencies here — that causes cascading restarts |
| **Startup probe** (if needed) | Protects slow-starting apps from premature liveness kills | Use `failureThreshold × periodSeconds` to define a startup budget |
| **Graceful shutdown** | Handles `SIGTERM` cleanly | Drain in-flight requests, close DB connections, flush buffers. Default grace period: 30s |

### Observability Contract

Applications are not production-ready until they are observable. Every component fulfills this contract:

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

### Configuration & Secrets

| Policy | Standard |
|--------|----------|
| Configuration injection | All config is injected via environment variables or mounted ConfigMaps. No config files baked into images |
| Secrets management | Production secrets are sourced from an external secrets manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) via the External Secrets Operator. Kubernetes Secrets objects are acceptable only in dev/stage |
| No secrets in images | Container images do not contain credentials, API keys, certificates, or connection strings |
| No secrets in Git | No secrets in source repositories. Use SealedSecrets or ExternalSecrets for GitOps workflows |

---

## Platform-Provided Capabilities

The platform provides these capabilities out of the box. Application teams do not reimplement them — doing so creates operational burden, security gaps, and supportability issues.

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

### Platform Citizenship Contract

Deploying on a shared platform is a social contract. Every application team agrees to these responsibilities in exchange for the capabilities the platform provides:

| Commitment | What This Means |
|------------|-----------------|
| **Own your resource footprint** | Set accurate CPU/memory requests and limits. Over-requesting starves other tenants; under-requesting causes OOM kills and noisy-neighbor effects |
| **Be observable** | If the platform team can't see your app's health, they can't help you during incidents. Logging, metrics, and health probes are mandatory — not optional extras |
| **Respect shared infrastructure** | Don't circumvent NetworkPolicies, don't request privileged SCCs without justification, don't run as root. These policies protect everyone on the cluster |
| **Keep your images current** | Rebuild regularly against updated base images. Stale images accumulate CVEs that affect the platform's security posture |
| **Automate everything** | No manual deployments, no snowflake configurations. If it can't be reproduced from Git, it doesn't belong in production |
| **Participate in incident response** | When your app causes cluster impact, respond promptly. Maintain runbooks and ensure on-call coverage |
| **Communicate changes** | Coordinate with the platform team for large-scale changes (migration of workloads, major scaling events, new external integrations) |
