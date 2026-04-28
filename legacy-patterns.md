---
layout: default
title: Legacy → Cloud-Native Rewrites
nav_order: 6
---

# Legacy Code Patterns → Cloud-Native Rewrites

Modernizing isn't just about putting your app in a container. These are the specific code-level changes that need to happen — think of them as migration rules, similar to what [Red Hat Migration Toolkit for Applications (MTA)](https://developers.redhat.com/products/mta/overview) flags automatically. This section answers: _"I write my legacy code this way — how do I rewrite it for cloud-native?"_

## Application Architecture

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

## Networking & Communication

| Legacy Pattern | Why It Breaks | Cloud-Native Replacement |
|---------------|---------------|--------------------------|
| Hardcoded IP addresses for services | IPs change as pods reschedule | Kubernetes Service DNS (`myservice.mynamespace.svc.cluster.local`) |
| Custom service discovery / registry | Unnecessary complexity; platform provides this | Kubernetes Services + DNS |
| Sticky sessions / server affinity | Prevents horizontal scaling; breaks on pod replacement | Stateless design + external session store. If truly unavoidable, use Route session affinity as a short-term bridge |
| Direct database connections from the UI tier | Security risk; no connection management | API tier mediates all database access; UI tier talks only to APIs |

## Security

| Legacy Pattern | Why It Fails | Cloud-Native Replacement |
|---------------|-------------|--------------------------|
| Running as Administrator / SYSTEM | OpenShift rejects root containers under `restricted-v2` SCC | Run as non-root (UID ≥ 1000). Build images with a non-root `USER` directive |
| Self-signed certs managed by the app | Operational burden; breaks automation | Platform TLS via Routes (edge or reencrypt termination) |
| Custom authentication / credential management | Security risk; duplicates platform capability | Integrate with platform OIDC/OAuth provider |
| Storing secrets in config files or environment variables in source control | Secrets leaked in Git history | External Secrets Operator → Vault/Key Vault; SealedSecrets for GitOps |

---

## Before & After: Common Rewrites

These code examples show the most impactful changes. Each is a real pattern teams encounter during modernization.

### Session State

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

### Configuration

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

### Logging

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

### Health Checks

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

### Resilient HTTP Calls

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

### Service Discovery

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
