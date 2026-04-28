---
layout: default
title: Maturity Model
nav_order: 7
---

# Modernization Maturity Model

Not every application starts cloud-native. Use this maturity model to assess where an application is today and define a concrete path forward.

## Level 1: Containerized (Minimum Viable)

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

## Level 2: Cloud-Ready

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

## Level 3: Cloud-Native

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
