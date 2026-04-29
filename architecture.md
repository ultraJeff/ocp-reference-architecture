---
layout: default
title: Architecture Standards
nav_order: 3
---

# Architecture Standards by Tier

## Presentation Tier (UI)

| Standard | Requirement Level |
|----------|-------------------|
| Serve static assets via a lightweight HTTP server (nginx, Caddy) or SSR runtime (Node.js, Kestrel) | Required |
| Externalize API base URLs via environment variables injected at build or runtime | Required |
| Set minimum 2 replicas in production | Required |
| Configure HPA with CPU target ≤ 70% | SHOULD |
| Use a Route with TLS edge termination | Required |
| Return a meaningful `/readyz` and `/livez` | Required |

**.NET-specific guidance:**
- Use ASP.NET Core with Kestrel as the web server — do not use IIS or HTTP.sys in-container
- For Blazor Server or MVC apps, ensure session state is externalized (Redis, SQL) rather than held in-process
- For Blazor WebAssembly or Angular/React SPAs served from a .NET backend, consider splitting the static assets into a separate lightweight container

## API / Business Logic Tier

| Standard | Requirement Level |
|----------|-------------------|
| Stateless design — no local session state, no local file storage for runtime data | Required |
| Expose a RESTful or gRPC API with versioning (URI path or header) | Required |
| Connect to databases and external services via injected configuration (never hardcoded) | Required |
| Set minimum 2 replicas in production | Required |
| Configure HPA with CPU target ≤ 70% | SHOULD |
| Implement circuit breakers for downstream calls | SHOULD |
| Expose `/metrics` with RED metrics (rate, errors, duration) | Required |
| Implement request timeouts and retry budgets | SHOULD |
| Use an internal Service (ClusterIP) for intra-cluster traffic; Route only for external-facing APIs | Required |

**.NET-specific guidance:**
- Target .NET 10 (LTS) for all new development. .NET 8 (LTS) remains supported through November 2026 for existing applications. .NET Framework 4.x applications are migrated to .NET 10 to run on Linux containers
- Use the `Microsoft.Extensions.Configuration` abstraction to bind environment variables and ConfigMaps to strongly-typed options classes
- Use `IHttpClientFactory` with `AddStandardResilienceHandler()` from `Microsoft.Extensions.Http.Resilience` for resilient HTTP calls (retry, circuit breaker, timeout) — do not instantiate `HttpClient` directly
- Use ASP.NET Core's built-in health check framework (`Microsoft.Extensions.Diagnostics.HealthChecks`) for `/livez` and `/readyz` endpoints
- Expose Prometheus metrics via OpenTelemetry .NET SDK (preferred) or `prometheus-net.AspNetCore`
- Connection strings for external databases (e.g., MSSQL) are injected via environment variables or mounted Secrets — never in `appsettings.json` baked into the image

## Data Tier

Databases frequently live outside the OpenShift cluster — on VMs, managed cloud services, or existing enterprise infrastructure. This is normal and expected. The standards below apply regardless of where the database runs.

### External Database (Common Pattern)

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

### In-Cluster Database (When Justified)

For dev/test environments, or when the architecture requires a database co-located with the application, in-cluster databases are acceptable.

| Standard | Requirement Level |
|----------|-------------------|
| Use an Operator-managed database (Crunchy PGO, MongoDB Enterprise, Redis Enterprise) when running databases on-cluster in production | SHOULD |
| Self-managed StatefulSets are acceptable in dev/stage | MAY |
| Configure automated backups with a tested restore procedure | Required |
| Use a dedicated StorageClass with appropriate IOPS and reclaim policy (`Retain` for prod) | Required |
| Use headless Services for StatefulSet DNS | Required |

## Storage Tier

Many legacy applications rely on file shares (SMB/CIFS, NFS, Windows shared drives) for inter-process communication, document storage, or configuration sharing. This pattern does not translate well to containers — pods are ephemeral, shared file systems create tight coupling, and horizontal scaling breaks assumptions about file-based coordination.

### Legacy File Share Pattern → Cloud-Native Alternative

| Legacy Pattern | Problem in Containers | Cloud-Native Alternative |
|---------------|----------------------|--------------------------|
| Shared network drive for document storage | Tight coupling between pods, no lifecycle management, poor performance at scale | **Object storage** (S3-compatible via ODF/NooBaa or external S3) with application-level access via SDK |
| File-based inter-process communication (drop files, pickup directories) | Race conditions across replicas, no ordering guarantees, breaks with horizontal scaling | **Message queue** (AMQ Streams/Kafka, AMQ Broker) or **event-driven architecture** |
| Local file storage for user uploads or generated reports | Data lost on pod restart, cannot scale horizontally | **Object storage** with pre-signed URLs for upload/download, or **PVC** (RWO) for single-writer workloads |
| Config files on shared drives | Environment drift, no auditability, manual updates | **ConfigMaps** and **Secrets** managed via GitOps |
| Log files written to shared volumes | No structured indexing, hard to search, fills disk | **Structured logging to stdout** → platform log aggregation (Loki/Splunk) |
| Temp files for processing | Breaks with multiple replicas, fills local storage | **EmptyDir volumes** (ephemeral, per-pod) for scratch space, or offload to object storage for durable processing |

### When Persistent Storage Is Justified

Some workloads legitimately need persistent, file-system-like storage. Use PersistentVolumeClaims (PVCs) with the appropriate access mode:

| Access Mode | Use Case | Example |
|-------------|----------|---------|
| **RWO** (ReadWriteOnce) | Single pod writes, others read from API/service | Database data directory, single-instance processing |
| **RWX** (ReadWriteMany) | Multiple pods need concurrent file access | Shared content repository (use sparingly — prefer object storage) |
| **ROX** (ReadOnlyMany) | Multiple pods read static content | Shared reference data, ML models |

> **Decision rule:** If the data is unstructured (documents, images, reports), default to **object storage**. If the data is structured and queried, use a **database**. Use PVCs with RWX only when object storage or a database genuinely cannot serve the access pattern — and document the justification.
