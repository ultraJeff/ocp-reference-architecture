# Application Modernization Standards & Cloud-Native Best Practices for OpenShift

> This document defines the standards, policies, and architectural patterns for deploying well-architected custom applications on Red Hat OpenShift Container Platform. It focuses on what happens **inside the application** — the code patterns, design decisions, and operational behaviors that make an app a good citizen on a shared platform — not just the infrastructure around it.
>
> **Audience:** Application developers, security engineers, and operations/SRE teams. Successful modernization is a cross-functional effort — developers write the code, security validates the posture, and operations ensures the app is observable and sustainable in production.
>
> It is prescriptive by design: **Required** items are non-negotiable for production workloads, **SHOULD** items are strongly recommended, and **MAY** items are available when justified by application needs.

---

## 1. Application Readiness Requirements

Before an application is deployed to OpenShift, it meets these baseline requirements. These are not suggestions — they are prerequisites for platform onboarding.

### 1.1 Container-Native Design

| Requirement | Standard | Why |
|-------------|----------|-----|
| Single process per container | Each container runs one process | Enables independent scaling, restart isolation, and clear resource attribution |
| Non-root execution | Containers run as non-root (UID ≥ 1000) | OpenShift enforces `restricted-v2` SCC by default; root containers will be rejected |
| No host dependencies | No hostPath volumes, host networking, or privileged access | Ensures workload portability and cluster security |
| Immutable images | Application code is baked into the image at build time, not mounted or downloaded at runtime | Guarantees reproducibility and auditability across environments |
| No hardcoded configuration | All environment-specific values (URLs, credentials, feature flags) are injected via ConfigMaps, Secrets, or environment variables | Enables promotion across dev → stage → prod without image rebuilds |

### 1.2 Health & Lifecycle

Every application component implements the following endpoints:

| Endpoint | Purpose | Implementation |
|----------|---------|----------------|
| **Readiness probe** (`/ready`) | Signals the pod is ready to accept traffic | Returns 200 when the app can serve requests. Return 503 during startup or when downstream dependencies are unavailable |
| **Liveness probe** (`/healthz`) | Signals the process is alive and not deadlocked | Returns 200 if the process is running. Do NOT check downstream dependencies here — that causes cascading restarts |
| **Startup probe** (if needed) | Protects slow-starting apps from premature liveness kills | Use `failureThreshold × periodSeconds` to define a startup budget |
| **Graceful shutdown** | Handles `SIGTERM` cleanly | Drain in-flight requests, close DB connections, flush buffers. Default grace period: 30s |

### 1.3 Observability Contract

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

### 1.4 Configuration & Secrets

| Policy | Standard |
|--------|----------|
| Configuration injection | All config is injected via environment variables or mounted ConfigMaps. No config files baked into images |
| Secrets management | Production secrets are sourced from an external secrets manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) via the External Secrets Operator. Kubernetes Secrets objects are acceptable only in dev/stage |
| No secrets in images | Container images do not contain credentials, API keys, certificates, or connection strings |
| No secrets in Git | No secrets in source repositories. Use SealedSecrets or ExternalSecrets for GitOps workflows |

---

## 2. Platform-Provided Capabilities

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

---

## 3. Architecture Standards by Tier

### 3.1 Presentation Tier (UI)

| Standard | Requirement Level |
|----------|-------------------|
| Serve static assets via a lightweight HTTP server (nginx, Caddy) or SSR runtime (Node.js, Kestrel) | Required |
| Externalize API base URLs via environment variables injected at build or runtime | Required |
| Set minimum 2 replicas in production  Required |
| Configure HPA with CPU target ≤ 70% | SHOULD |
| Use a Route with TLS edge termination  Required |
| Return a meaningful `/ready` and `/healthz`  Required |

**.NET-specific guidance:**
- Use ASP.NET Core with Kestrel as the web server — do not use IIS or HTTP.sys in-container
- For Blazor Server or MVC apps, ensure session state is externalized (Redis, SQL) rather than held in-process
- For Blazor WebAssembly or Angular/React SPAs served from a .NET backend, consider splitting the static assets into a separate lightweight container

### 3.2 API / Business Logic Tier

| Standard | Requirement Level |
|----------|-------------------|
| Stateless design — no local session state, no local file storage for runtime data  Required |
| Expose a RESTful or gRPC API with versioning (URI path or header)  Required |
| Connect to databases and external services via injected configuration (never hardcoded)  Required |
| Set minimum 2 replicas in production  Required |
| Configure HPA with CPU target ≤ 70% | SHOULD |
| Implement circuit breakers for downstream calls | SHOULD |
| Expose `/metrics` with RED metrics (rate, errors, duration)  Required |
| Implement request timeouts and retry budgets | SHOULD |
| Use an internal Service (ClusterIP) for intra-cluster traffic; Route only for external-facing APIs  Required |

**.NET-specific guidance:**
- Target .NET 8+ (LTS) for all new development and modernization efforts. .NET Framework 4.x applications are migrated to .NET 8+ to run on Linux containers
- Use the `Microsoft.Extensions.Configuration` abstraction to bind environment variables and ConfigMaps to strongly-typed options classes
- Use `IHttpClientFactory` with Polly for resilient HTTP calls (retry, circuit breaker, timeout) — do not instantiate `HttpClient` directly
- Use ASP.NET Core's built-in health check framework (`Microsoft.Extensions.Diagnostics.HealthChecks`) for `/healthz` and `/ready` endpoints
- Expose Prometheus metrics via `prometheus-net.AspNetCore` or OpenTelemetry .NET SDK
- Connection strings for external databases (e.g., MSSQL) are injected via environment variables or mounted Secrets — never in `appsettings.json` baked into the image

### 3.3 Data Tier

Databases frequently live outside the OpenShift cluster — on VMs, managed cloud services, or existing enterprise infrastructure. This is normal and expected. The standards below apply regardless of where the database runs.

#### External Database (Common Pattern)

Most modernization efforts keep the existing database in place and modernize the application tier. The application connects to the database over the network.

| Standard | Requirement Level |
|----------|-------------------|
| Database connection strings and credentials are injected via Secrets (never hardcoded, never baked into images) | Required |
| Production credentials are sourced from an external secrets manager via External Secrets Operator | Required |
| NetworkPolicy egress rules explicitly allow traffic from the API tier to the database host/port | Required |
| Connection pooling is configured appropriately for containerized workloads (containers restart more frequently than VMs — set reasonable pool sizes and connection lifetimes) | Required |
| Applications handle transient database connectivity failures gracefully (retry with backoff, not crash) | Required |
| Schema migrations SHOULD be run as Kubernetes Jobs or via a controlled CI/CD step, not as part of application startup | SHOULD |
| Database backup and restore procedures are documented and tested, even if managed by a separate team | Required |

**.NET + MSSQL-specific guidance:**
- Use `Microsoft.Data.SqlClient` (not the legacy `System.Data.SqlClient`) for SQL Server connectivity
- Configure `SqlConnection` pool size via connection string parameters (`Max Pool Size`, `Min Pool Size`) — defaults designed for long-lived VM processes may cause pool exhaustion in containers
- Set `Connection Lifetime` or `Load Balance Timeout` to recycle connections periodically, accommodating pod restarts and rolling deployments
- Use Entity Framework Core migrations via a dedicated Job or pipeline step — do not run `Database.Migrate()` in `Program.cs` for production
- For Always On Availability Groups or failover clusters, use `MultiSubnetFailover=True` in the connection string

#### In-Cluster Database (When Justified)

For dev/test environments, or when the architecture requires a database co-located with the application, in-cluster databases are acceptable.

| Standard | Requirement Level |
|----------|-------------------|
| Use an Operator-managed database (Crunchy PGO, MongoDB Enterprise, Redis Enterprise) when running databases on-cluster in production | SHOULD |
| Self-managed StatefulSets are acceptable in dev/stage | MAY |
| Configure automated backups with a tested restore procedure | Required |
| Use a dedicated StorageClass with appropriate IOPS and reclaim policy (`Retain` for prod) | Required |
| Use headless Services for StatefulSet DNS | Required |

### 3.4 Storage Tier

Many legacy applications rely on file shares (SMB/CIFS, NFS, Windows shared drives) for inter-process communication, document storage, or configuration sharing. This pattern does not translate well to containers — pods are ephemeral, shared file systems create tight coupling, and horizontal scaling breaks assumptions about file-based coordination.

#### Legacy File Share Pattern → Cloud-Native Alternative

| Legacy Pattern | Problem in Containers | Cloud-Native Alternative |
|---------------|----------------------|--------------------------|
| Shared network drive for document storage | Tight coupling between pods, no lifecycle management, poor performance at scale | **Object storage** (S3-compatible via ODF/NooBaa or external S3) with application-level access via SDK |
| File-based inter-process communication (drop files, pickup directories) | Race conditions across replicas, no ordering guarantees, breaks with horizontal scaling | **Message queue** (AMQ Streams/Kafka, AMQ Broker) or **event-driven architecture** |
| Local file storage for user uploads or generated reports | Data lost on pod restart, cannot scale horizontally | **Object storage** with pre-signed URLs for upload/download, or **PVC** (RWO) for single-writer workloads |
| Config files on shared drives | Environment drift, no auditability, manual updates | **ConfigMaps** and **Secrets** managed via GitOps |
| Log files written to shared volumes | No structured indexing, hard to search, fills disk | **Structured logging to stdout** → platform log aggregation (Loki/Splunk) |
| Temp files for processing | Breaks with multiple replicas, fills local storage | **EmptyDir volumes** (ephemeral, per-pod) for scratch space, or offload to object storage for durable processing |

#### When Persistent Storage Is Justified

Some workloads legitimately need persistent, file-system-like storage. Use PersistentVolumeClaims (PVCs) with the appropriate access mode:

| Access Mode | Use Case | Example |
|-------------|----------|---------|
| **RWO** (ReadWriteOnce) | Single pod writes, others read from API/service | Database data directory, single-instance processing |
| **RWX** (ReadWriteMany) | Multiple pods need concurrent file access | Shared content repository (use sparingly — prefer object storage) |
| **ROX** (ReadOnlyMany) | Multiple pods read static content | Shared reference data, ML models |

> **Decision rule:** If the data is unstructured (documents, images, reports), default to **object storage**. If the data is structured and queried, use a **database**. Use PVCs with RWX only when object storage or a database genuinely cannot serve the access pattern — and document the justification.

---

## 4. Security Standards

These are non-negotiable. Exceptions require documented approval from the security team.

### 4.1 Pod Security

All workloads comply with the `restricted-v2` Security Context Constraint:

- `runAsNonRoot: true`
- `allowPrivilegeEscalation: false`
- `capabilities.drop: [ALL]`
- `seccompProfile.type: RuntimeDefault`

Any request for elevated SCCs (`anyuid`, `privileged`) requires a written exception with justification, time-bound approval, and a remediation plan.

### 4.2 Network Segmentation

Every application namespace deploys NetworkPolicies:

1. **Default deny all ingress** — baseline policy, no exceptions
2. **Explicit allow rules** — whitelist traffic between tiers:
   - Router → UI (from `network.openshift.io/policy-group: ingress` namespace)
   - UI → API (pod-to-pod within namespace)
   - API → external database (egress to specific host IP and port, e.g., MSSQL on port 1433)
   - API → other external services (egress rules scoped to specific hosts/CIDRs as needed)
3. **No open namespaces** — a namespace without NetworkPolicies will not pass security review

### 4.3 Image Provenance

- Images are pulled from an approved registry (Quay, OpenShift internal registry, Red Hat registry)
- Images are built in a CI pipeline — no `docker build` on developer laptops pushed to prod
- Images pass ACS vulnerability scanning before deployment
- Base images use Red Hat UBI (Universal Base Image) or an approved alternative. For .NET workloads, use `registry.access.redhat.com/ubi8/dotnet-80-runtime` (runtime) or `ubi8/dotnet-80` (SDK) images
- Image tags use immutable references (SHA digests) in production GitOps manifests, not `:latest`

### 4.4 RBAC

- Application ServiceAccounts follow least-privilege — only the permissions the workload actually needs
- Do not use `cluster-admin` or `admin` ClusterRoles for application workloads
- Namespace-scoped Roles are preferred over ClusterRoles

### 4.5 Application-Level Security

The standards above cover how the platform protects workloads. These standards cover what happens **inside your code** — the security practices every developer owns:

| Practice | What to Do | What NOT to Do |
|----------|-----------|----------------|
| **Input validation** | Validate and sanitize all input at system boundaries (API endpoints, message consumers, file uploads). Use allow-lists, not deny-lists | Trust input from other internal services without validation. Internal ≠ trusted — a compromised upstream poisons everything downstream |
| **Avoid shelling out** | Use libraries and SDKs for file operations, HTTP calls, and system interactions | Call `Process.Start()`, `Runtime.exec()`, or backtick commands with user-supplied input. This is command injection waiting to happen |
| **Dependency hygiene** | Pin dependency versions. Run `dotnet list package --vulnerable` (or equivalent) in CI. Update regularly | Use wildcard version ranges in production. Ignore vulnerability scan results from ACS/Quay |
| **Don't log secrets** | Scrub sensitive fields (tokens, passwords, SSNs, PII) before logging. Use structured logging with explicit field selection | Log full request/response bodies in production. Use `ToString()` on objects that may contain credentials |
| **Read-only root filesystem** | Set `readOnlyRootFilesystem: true` in your security context. Write temp data to `EmptyDir` volumes mounted at `/tmp` | Assume your container needs a writable filesystem. Most apps don't — and a read-only root blocks many exploit techniques |
| **Minimize base image** | Use runtime-only images (`ubi8/dotnet-80-runtime`, not the SDK image). Use multi-stage builds to keep build tools out of production | Ship the SDK, build tools, or debugging utilities in your production image. Smaller image = smaller attack surface |

---

## 5. CI/CD & GitOps Standards

### 5.1 Pipeline Requirements

Every application has an automated CI/CD pipeline. Manual deployments to staging or production are not permitted.

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

### 5.2 GitOps Repository Structure

All application deployments are managed via a GitOps repository. ArgoCD is the standard deployment tool.

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

### 5.3 Environment Promotion

- **Dev → Stage**: Automated on merge to `main`. Image digest propagated via CI pipeline
- **Stage → Prod**: Requires manual approval gate (PR to `overlays/prod` in GitOps repo)
- Environment-specific configuration lives in Kustomize patches, not in the application code or image

---

## 6. Legacy Code Patterns → Cloud-Native Rewrites

Modernizing isn't just about putting your app in a container. These are the specific code-level changes that need to happen — think of them as migration rules, similar to what [Red Hat Migration Toolkit for Applications (MTA)](https://developers.redhat.com/products/mta/overview) flags automatically. This section answers: _"I write my legacy code this way — how do I rewrite it for cloud-native?"_

### Application Architecture

| Legacy Pattern | Why It Breaks on OpenShift | Cloud-Native Replacement |
|---------------|---------------------------|--------------------------|
| In-memory session state (`Session["user"]`, `HttpContext.Session`) | Lost on pod restart or reschedule; breaks with multiple replicas behind a load balancer | Externalize to Redis or SQL-backed session store. Use `IDistributedCache` in .NET |
| Hardcoded connection strings in `web.config` or `appsettings.json` | Image must be rebuilt per environment; secrets exposed in image layers | Inject via environment variables or mounted Secrets. Use `Microsoft.Extensions.Configuration` with env var providers |
| Singleton patterns holding runtime state | State lost on restart; conflicts across replicas | Move state to an external store (Redis, database). Keep singletons stateless |
| Writing to the local file system (logs, temp files, uploads) | Container filesystem is ephemeral; files disappear on restart; no sharing across replicas | Logs → stdout. Uploads → object storage. Temp → `EmptyDir` volumes |
| Windows Registry access | No registry in Linux containers | Environment variables or ConfigMaps |
| Windows Event Log writes | Not available on Linux; not collected by platform logging | `ILogger` → structured JSON to stdout |
| Scheduled tasks via Windows Task Scheduler or cron inside the container | Missed schedules on restart; duplicate execution across replicas | Kubernetes CronJobs for periodic tasks; Kubernetes Jobs for one-time work |
| COM/DCOM interop or .NET Remoting | Not supported on .NET 8+ or Linux | gRPC for inter-service communication; REST APIs; message queues |
| GAC (Global Assembly Cache) dependencies | Not available in containers | NuGet packages, self-contained deployments |

### Networking & Communication

| Legacy Pattern | Why It Breaks | Cloud-Native Replacement |
|---------------|---------------|--------------------------|
| Hardcoded IP addresses for services | IPs change as pods reschedule | Kubernetes Service DNS (`myservice.mynamespace.svc.cluster.local`) |
| Custom service discovery / registry | Unnecessary complexity; platform provides this | Kubernetes Services + DNS |
| Sticky sessions / server affinity | Prevents horizontal scaling; breaks on pod replacement | Stateless design + external session store. If truly unavoidable, use Route session affinity as a short-term bridge |
| Direct database connections from the UI tier | Security risk; no connection management | API tier mediates all database access; UI tier talks only to APIs |

### Security

| Legacy Pattern | Why It Fails | Cloud-Native Replacement |
|---------------|-------------|--------------------------|
| Running as Administrator / SYSTEM | OpenShift rejects root containers under `restricted-v2` SCC | Run as non-root (UID ≥ 1000). Build images with a non-root `USER` directive |
| Self-signed certs managed by the app | Operational burden; breaks automation | Platform TLS via Routes (edge or reencrypt termination) |
| Custom authentication / credential management | Security risk; duplicates platform capability | Integrate with platform OIDC/OAuth provider |
| Storing secrets in config files or environment variables in source control | Secrets leaked in Git history | External Secrets Operator → Vault/Key Vault; SealedSecrets for GitOps |

### Before & After: Common Rewrites

These code examples show the most impactful changes. Each is a real pattern teams encounter during modernization.

#### Session State

**Before** — in-memory session (breaks with multiple replicas):
```csharp
// Startup.cs — legacy pattern
builder.Services.AddSession(); // In-memory, lost on pod restart
```

**After** — externalized to Redis:
```csharp
// Program.cs — cloud-native
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration["Redis:ConnectionString"];
});
builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(20);
    options.Cookie.IsEssential = true;
});
```

#### Configuration

**Before** — hardcoded in `appsettings.json`, baked into image:
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=proddb01;Database=MyApp;User Id=sa;Password=hunter2;"
  }
}
```

**After** — injected via environment variables, secrets managed externally:
```csharp
// Program.cs
builder.Configuration.AddEnvironmentVariables();

// appsettings.json keeps structure only (no secrets, no env-specific values)
// Actual values come from:
//   - ConfigMap mounted as env vars (non-sensitive)
//   - Secret mounted as env vars (sensitive, sourced from ExternalSecret)
```

#### Logging

**Before** — writing to files or Windows Event Log:
```csharp
// Legacy pattern — logs to a file, not collected by platform
Log.Logger = new LoggerConfiguration()
    .WriteTo.File("C:\\Logs\\myapp.log")
    .CreateLogger();
```

**After** — structured JSON to stdout, collected automatically by the platform:
```csharp
// Program.cs
builder.Host.UseSerilog((context, config) => config
    .WriteTo.Console(new RenderedCompactJsonFormatter())
    .Enrich.WithProperty("ApplicationName", "my-api")
    .Enrich.WithCorrelationId());
```

#### Health Checks

**Before** — no health endpoints, or a simple "200 OK" that checks nothing:
```csharp
app.MapGet("/health", () => "OK"); // Tells you nothing useful
```

**After** — meaningful checks using ASP.NET Core health check framework:
```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "sqlserver",
        tags: new[] { "ready" })
    .AddRedis(
        builder.Configuration["Redis:ConnectionString"]!,
        name: "redis",
        tags: new[] { "ready" });

// Liveness — is the process alive? Don't check dependencies here.
app.MapHealthChecks("/healthz", new HealthCheckOptions
{
    Predicate = _ => false // No dependency checks — just "am I running?"
});

// Readiness — can I serve traffic? Check dependencies here.
app.MapHealthChecks("/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

#### Resilient HTTP Calls

**Before** — manually creating `HttpClient`, no retry logic:
```csharp
// Legacy — socket exhaustion, no resilience
var client = new HttpClient();
var response = await client.GetAsync("http://other-service/api/data");
```

**After** — `IHttpClientFactory` with Polly retry and circuit breaker:
```csharp
// Program.cs
builder.Services.AddHttpClient("OrderService", client =>
{
    client.BaseAddress = new Uri(
        builder.Configuration["Services:OrderService:BaseUrl"]!);
    client.Timeout = TimeSpan.FromSeconds(10);
})
.AddTransientHttpErrorPolicy(p => p.WaitAndRetryAsync(
    retryCount: 3,
    sleepDurationProvider: attempt => TimeSpan.FromMilliseconds(200 * Math.Pow(2, attempt))))
.AddTransientHttpErrorPolicy(p => p.CircuitBreakerAsync(
    handledEventsAllowedBeforeBreaking: 5,
    durationOfBreak: TimeSpan.FromSeconds(30)));

// In your service class — injected, pooled, resilient
public class OrderClient(IHttpClientFactory factory)
{
    public async Task<Order?> GetOrderAsync(int id)
    {
        var client = factory.CreateClient("OrderService");
        return await client.GetFromJsonAsync<Order>($"/api/orders/{id}");
    }
}
```

#### Service Discovery

**Before** — hardcoded IPs or hostnames:
```csharp
var client = new HttpClient { BaseAddress = new Uri("http://10.0.1.47:8080") };
```

**After** — Kubernetes DNS, injected via config:
```csharp
// appsettings.json (overridden per environment via ConfigMap)
// In dev overlay: "http://order-service.myapp-dev.svc.cluster.local:8080"
// In prod overlay: "http://order-service.myapp-prod.svc.cluster.local:8080"

builder.Services.AddHttpClient("OrderService", client =>
{
    client.BaseAddress = new Uri(builder.Configuration["Services:OrderService:BaseUrl"]!);
});
```

---

## 7. Modernization Maturity Model

Not every application starts cloud-native. Use this maturity model to assess where an application is today and define a concrete path forward.

### Level 1: Containerized (Minimum Viable)

The application runs in a Linux container on OpenShift but has not been redesigned. For .NET shops, this is typically the first milestone after migrating from .NET Framework on Windows to .NET 8+ on Linux.

| Characteristic | Status |
|----------------|--------|
| Runs in a Linux container | Yes |
| Runtime | .NET 8+ on UBI base image (migration from .NET Framework complete) |
| Non-root execution | Yes (required by platform) |
| Health probes | Basic (TCP or HTTP) |
| Logging | May still use `ILogger` writing to files or Windows Event Log patterns — needs stdout migration |
| Config | May still rely on `appsettings.json` or `web.config` patterns baked into the image |
| Database | Connects to existing external database (e.g., MSSQL on VM) — no changes to data tier |
| State | May be stateful (in-memory session, local file storage) |
| Scaling | Single replica, manual |
| Deployment | May be manual or semi-automated |

**What to focus on next:**
- Migrate `appsettings.json` values to environment variables / ConfigMaps (keep `appsettings.json` for structure, override via env vars)
- Configure `ILogger` + Serilog/NLog to write structured JSON to stdout
- Implement ASP.NET Core health checks (`/healthz`, `/ready`)
- Externalize session state to Redis or SQL if using in-memory sessions
- Replace `System.Data.SqlClient` with `Microsoft.Data.SqlClient`
- Tune connection pooling for container lifecycle (shorter connection lifetimes)

### Level 2: Cloud-Ready

The application meets all requirements in this document and can be deployed reliably via GitOps. The data tier remains where it is — what changes is how the application connects to it, monitors itself, and gets deployed.

| Characteristic | Status |
|----------------|--------|
| Runtime | .NET 8+ on UBI, Kestrel web server |
| Non-root, restricted SCC | Yes |
| Health probes | ASP.NET Core health check framework with readiness + liveness, meaningful checks |
| Logging | Structured JSON to stdout via Serilog or OpenTelemetry logging |
| Config | Externalized via environment variables, ConfigMaps, and Secrets |
| Database | External database with connection strings injected via Secrets, connection pooling tuned |
| State | Stateless application tier — sessions externalized, no local file dependencies |
| Scaling | HPA-enabled, minimum 2 replicas |
| Deployment | Fully automated via GitOps (ArgoCD) |
| Metrics | `/metrics` exposed via `prometheus-net` or OpenTelemetry, ServiceMonitor deployed |
| Resilience | `IHttpClientFactory` + Polly policies for downstream calls |
| Security | NetworkPolicies, ExternalSecrets, image scanning, UBI base images |

**What to focus on next:** Decompose monolithic components into bounded services, add distributed tracing via OpenTelemetry .NET SDK, implement more granular scaling.

### Level 3: Cloud-Native

The application is decomposed into independently deployable services and takes full advantage of the platform. The data tier may still be external — cloud-native does not require an in-cluster database.

| Characteristic | Status |
|----------------|--------|
| All Level 2 requirements | Yes |
| Architecture | Microservices or well-bounded modular services, each with its own repo, pipeline, and deployment |
| Communication | Async where appropriate (Kafka, AMQP via MassTransit or NServiceBus), sync via well-defined APIs |
| Database | Per-service data ownership; shared databases avoided or isolated via schema-per-service |
| Scaling | Custom metric scaling (KEDA), scale-to-zero where applicable |
| Resilience | Circuit breakers, retry budgets, bulkheads, graceful degradation (Polly v8 resilience pipelines) |
| Tracing | Full distributed tracing via OpenTelemetry .NET SDK |
| Deployment | Canary or blue-green release strategies |
| API management | Versioned APIs, OpenAPI specs via Swashbuckle/NSwag, deprecation lifecycle |
| Multi-environment | Consistent deployment across clusters via ApplicationSets |

---

## 8. Extension Catalog

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

## 9. Baseline Architecture Diagram

Every well-architected application on this platform follows this foundational pattern. Extensions from the catalog above are layered on top.

```
  ┌──────────────────────────────────────────────────────────────────────────┐
  │                         OpenShift Cluster                                │
  │                                                                          │
  │  ┌────────────────────────────────────────────────────────────────────┐  │
  │  │                Ingress (Router / HAProxy)                          │  │
  │  │            TLS termination · Route-based routing                   │  │
  │  └──────────┬─────────────────────────────┬───────────────────────────┘  │
  │             │                             │                              │
  │             ▼                             ▼                              │
  │  ┌──────────────────────┐    ┌──────────────────────────┐               │
  │  │  Presentation Tier   │    │  API / Business Logic     │               │
  │  │                      │    │                           │               │
  │  │  .NET 8+ / Kestrel   │    │  .NET 8+ / ASP.NET Core  │               │
  │  │  Deployment (≥2)     │    │  Deployment (≥2)          │               │
  │  │  Service + Route     │    │  Service + Route          │               │
  │  │  HPA (CPU ≤70%)      │    │  HPA (CPU ≤70%)           │               │
  │  │  /healthz + /ready   │    │  /healthz + /ready        │               │
  │  │  /metrics            │    │  /metrics                 │               │
  │  └──────────────────────┘    └──────────┬────────────────┘               │
  │                                         │                                │
  │  ┌────────────────────────────────────────────────────────────────────┐  │
  │  │                    Platform Services                               │  │
  │  │                                                                    │  │
  │  │  Logging (stdout → ClusterLogForwarder → Loki/Splunk)              │  │
  │  │  Monitoring (ServiceMonitor → Prometheus)                          │  │
  │  │  Alerting (PrometheusRule → AlertManager)                          │  │
  │  │  Secrets (ExternalSecret → Vault/Key Vault)                        │  │
  │  │  Deployment (ArgoCD → GitOps repo)                                 │  │
  │  │  Scanning (ACS → Quay)                                             │  │
  │  │  Network (default-deny + explicit allow)                            │  │
  │  └────────────────────────────────────────────────────────────────────┘  │
  └──────────────────────────────┬───────────────────────────────────────────┘
                                 │
                                 │  NetworkPolicy egress
                                 │  (explicit allow to DB host:port)
                                 │
                                 ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │                     External Data Tier                                   │
  │                                                                          │
  │  ┌──────────────────────┐    ┌──────────────────────┐                   │
  │  │  MSSQL Server (VM)   │    │  Other External       │                   │
  │  │                      │    │  Services              │                   │
  │  │  Existing infra      │    │  (APIs, LDAP, SMTP,    │                   │
  │  │  Managed by DBA team │    │   cloud services)      │                   │
  │  │  Backup/DR in place  │    │                        │                   │
  │  └──────────────────────┘    └──────────────────────┘                   │
  └──────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────┐
  │                         CI/CD Pipeline                                   │
  │                                                                          │
  │  Git push → Lint → Test → Build (.NET) → Scan → Push image             │
  │  → Update GitOps repo → ArgoCD sync → Smoke test                       │
  └──────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Application Onboarding Decision Tree

Use this decision tree when evaluating a custom application for deployment on the platform. It provides a standard intake assessment that spans development, security, and operations concerns.

```
Is the application containerized?
├── NO → Can it be containerized?
│   ├── NO (hard OS/kernel dependencies, proprietary runtime)
│   │   → Not a candidate for OpenShift. Escalate for alternative hosting.
│   └── YES → Proceed to containerization (target Maturity Level 1)
│       ├── .NET apps: target .NET 8+ on UBI base images
│       └── Other stacks: target supported runtime on UBI base images
│
└── YES → Does it run as non-root?
    ├── NO → Can it be modified to run non-root?
    │   ├── YES → Fix and re-assess
    │   └── NO → Requires SCC exception (security team approval required)
    │
    └── YES → Does it rely on local file storage or file shares?
        ├── YES → Identify usage patterns
        │   └── See Section 3.4 (Storage Tier) for migration alternatives
        │
        └── NO → Does it externalize all configuration?
            ├── NO → Migrate hardcoded config to env vars / ConfigMaps / Secrets
            │
            └── YES → Does it implement health probes?
                ├── NO → Implement /healthz and /ready endpoints
                │
                └── YES → Does it log structured output to stdout?
                    ├── NO → Migrate logging to structured JSON on stdout
                    │
                    └── YES → Does it expose /metrics?
                        ├── NO → Add Prometheus metrics endpoint
                        │
                        └── YES → ✅ Ready for platform onboarding
                            └── Proceed to Governance Checklist (Section 11)
```

### Intake Responsibilities by Role

Platform onboarding is not just a developer exercise. Successful onboarding requires coordination across teams:

| Role | Responsibility During Onboarding |
|------|----------------------------------|
| **Application Developer** | Containerize the app, implement health probes, externalize config, add metrics/logging, eliminate file share dependencies |
| **Security Engineer** | Review SCC requirements, validate NetworkPolicy design, approve image base and scanning results, confirm secrets management approach |
| **Operations / SRE** | Define resource quotas, validate scaling strategy, ensure monitoring/alerting is wired up, confirm runbook and on-call readiness |
| **Platform Team** | Provision namespace, configure GitOps sync, validate CI/CD pipeline integration, approve external network egress rules |
| **Database / Storage Team** | Validate connectivity from cluster to external databases, confirm backup/restore procedures, advise on storage alternatives for file shares |

---

## 11. Governance & Compliance Checklist

Use this checklist during application onboarding and periodic reviews.

### Pre-Deployment Gate (all required)

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
- [ ] Database connectivity tested under pod restart/reschedule conditions (connection pooling, transient fault handling)
- [ ] Backup and restore procedure documented and tested (data tier — whether in-cluster or external)

### Post-Deployment Verification (SHOULD)

- [ ] Load test completed against staging environment
- [ ] Alerts fire correctly under simulated failure conditions
- [ ] Runbook exists for common failure scenarios
- [ ] On-call team trained on application architecture and failure modes

### What Is a Runbook?

A runbook is a step-by-step guide that tells an on-call engineer what to do when something goes wrong with your application — **at 2 AM, without context, having never seen your code**. It's the difference between "figure it out" and "follow these steps." Every production application needs one.

A good runbook is:
- **Specific to your app**, not generic Kubernetes troubleshooting
- **Action-oriented** — tells you what to run, not what to think about
- **Kept next to the code** — in the app's GitOps repo or wiki, not a shared drive no one can find

#### Runbook Template

Use this as a starting point. Fill in the sections relevant to your application.

```markdown
# [App Name] — Operations Runbook

## Application Overview
- **What it does:** [1-2 sentence description]
- **Team / owner:** [team name, Slack channel, PagerDuty service]
- **Architecture:** [UI + API + external MSSQL, or whatever applies]
- **Namespace(s):** [myapp-dev, myapp-stage, myapp-prod]
- **GitOps repo:** [link]
- **Dashboards:** [link to Grafana/Prometheus dashboard]

## Key Endpoints
| Endpoint | Purpose |
|----------|---------|
| `/healthz` | Liveness — is the process alive? |
| `/ready` | Readiness — can it serve traffic? |
| `/metrics` | Prometheus metrics |
| [app-specific] | [e.g., `/api/v1/status` for business health] |

## Common Failure Scenarios

### Pods are crash-looping
1. Check pod logs: `oc logs -f deployment/myapp-api -n myapp-prod`
2. Check events: `oc get events -n myapp-prod --sort-by=.lastTimestamp`
3. Common causes:
   - Database unreachable → check connectivity to [DB host:port]
   - Config missing → verify ConfigMap/Secret exists and is mounted
   - OOM killed → check `oc describe pod` for OOMKilled reason, increase memory limit

### High error rate (5xx responses)
1. Check dashboard: [link]
2. Check logs for exceptions: `oc logs deployment/myapp-api -n myapp-prod | grep -i error`
3. Check downstream dependencies:
   - Database: [how to verify]
   - External API: [how to verify]
4. If isolated to this app: restart deployment `oc rollout restart deployment/myapp-api -n myapp-prod`

### High latency
1. Check dashboard: [link]
2. Check database query performance: [how to verify]
3. Check HPA status: `oc get hpa -n myapp-prod` — are we at max replicas?
4. Check resource usage: `oc adm top pods -n myapp-prod`

### Cannot connect to database
1. Verify database is reachable from cluster: `oc debug deployment/myapp-api -- curl -v telnet://[db-host]:[db-port]`
2. Check Secret has correct credentials: `oc get secret myapp-db-credentials -n myapp-prod -o jsonpath='{.data}'`
3. Check NetworkPolicy allows egress to database host
4. Contact DBA team: [contact info]

## Rollback Procedure
1. Identify last known good commit in GitOps repo
2. Revert in GitOps repo or use ArgoCD to sync to previous commit
3. Verify rollback: check `/ready` endpoint returns 200

## Escalation Path
1. **L1 (on-call):** [team/person], [Slack channel]
2. **L2 (app owner):** [team/person]
3. **L3 (platform team):** [Slack channel / PagerDuty]
4. **Database issues:** [DBA team contact]
```

---

## 12. Developer Inner Loop

The standards in this document describe what production looks like. This section covers how to develop against those standards locally — so issues are caught on your laptop, not in the CI pipeline.

### Local Development Tooling

| Tool | What It Does | When to Use |
|------|-------------|-------------|
| **Podman Desktop** | Build and run containers locally without Docker daemon | Building and testing your container image before pushing |
| **odo** (OpenShift Do) | Inner-loop dev tool that syncs code changes to a running container on the cluster | Iterating quickly against a real OpenShift environment without full CI/CD cycles |
| **Podman Compose** | Run multi-container apps locally (API + database + Redis) | Testing your app with its dependencies before deploying to the cluster |
| **Dev Containers** (VS Code / JetBrains) | Develop inside a container that matches your production environment | Ensuring your local dev environment matches prod (same OS, same runtime) |

### Local Checklist

Before you push, verify these locally — they're the same things the platform will check:

```bash
# 1. Does it build as a container?
podman build -t myapp:dev .

# 2. Does it run as non-root?
podman run --user 1001:0 myapp:dev

# 3. Does it start without hardcoded config?
podman run -e ConnectionStrings__Default="Server=localhost;..." myapp:dev

# 4. Do health endpoints work?
curl http://localhost:8080/healthz   # Should return 200
curl http://localhost:8080/ready     # Should return 200 (or 503 if no DB)

# 5. Are logs going to stdout (not files)?
podman logs <container-id>          # Should see structured JSON output

# 6. Does /metrics respond?
curl http://localhost:8080/metrics   # Should return Prometheus format
```

### Multi-Stage Dockerfile Template

This pattern keeps your production image small and secure:

```dockerfile
# Build stage — SDK image, not shipped to prod
FROM registry.access.redhat.com/ubi8/dotnet-80 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app

# Runtime stage — minimal image, what actually runs in prod
FROM registry.access.redhat.com/ubi8/dotnet-80-runtime AS runtime
WORKDIR /app
COPY --from=build /app .

# Run as non-root
USER 1001
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```
