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
| COM/DCOM interop or .NET Remoting | Not supported on modern .NET or Linux | gRPC for inter-service communication; REST APIs; message queues |
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

<details open>
<summary><strong>.NET</strong></summary>

```csharp
// Startup.cs — legacy pattern
builder.Services.AddSession(); // In-memory, lost on pod restart
```
</details>

<details>
<summary><strong>Java (Spring Boot)</strong></summary>

```java
// Legacy — in-memory HttpSession, lost on pod restart
@GetMapping("/dashboard")
public String dashboard(HttpSession session) {
    session.setAttribute("user", currentUser);
    return "dashboard";
}
```
</details>

<details>
<summary><strong>Node.js (Express)</strong></summary>

```javascript
// Legacy — in-memory session store, lost on pod restart
const session = require('express-session');
app.use(session({ secret: 'keyboard-cat', resave: false }));
```
</details>

**After** — externalized to Redis:

<details open>
<summary><strong>.NET</strong></summary>

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
</details>

<details>
<summary><strong>Java (Spring Boot)</strong></summary>

```java
// application.yaml — externalized to Redis via Spring Session
// spring.data.redis.host is injected via ConfigMap/env var
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1200)
@Configuration
public class SessionConfig {
    @Bean
    public RedisConnectionFactory connectionFactory(
            @Value("${spring.data.redis.host}") String host,
            @Value("${spring.data.redis.port:6379}") int port) {
        return new LettuceConnectionFactory(host, port);
    }
}
```
</details>

<details>
<summary><strong>Node.js (Express)</strong></summary>

```javascript
// Cloud-native — Redis-backed sessions
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');

const redisClient = createClient({ url: process.env.REDIS_URL });
await redisClient.connect();

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
}));
```
</details>

### Configuration

**Before** — hardcoded config baked into image:

<details open>
<summary><strong>.NET</strong></summary>

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=proddb01;Database=MyApp;User Id=sa;Password=hunter2;"
  }
}
```
</details>

<details>
<summary><strong>Java</strong></summary>

```properties
# application.properties — baked into JAR
spring.datasource.url=jdbc:postgresql://proddb01:5432/myapp
spring.datasource.username=admin
spring.datasource.password=hunter2
```
</details>

<details>
<summary><strong>Node.js</strong></summary>

```javascript
// config.js — hardcoded
module.exports = {
  db: { host: 'proddb01', port: 5432, password: 'hunter2' }
};
```
</details>

**After** — injected via environment variables, secrets managed externally:

<details open>
<summary><strong>.NET</strong></summary>

```csharp
// Program.cs
builder.Configuration.AddEnvironmentVariables();

// appsettings.json keeps structure only (no secrets, no env-specific values)
// Actual values come from:
//   - ConfigMap mounted as env vars (non-sensitive)
//   - Secret mounted as env vars (sensitive, sourced from ExternalSecret)
```
</details>

<details>
<summary><strong>Java (Spring Boot)</strong></summary>

```yaml
# application.yaml — structure only, values from env vars
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```
</details>

<details>
<summary><strong>Node.js</strong></summary>

```javascript
// config.js — reads from environment
module.exports = {
  db: {
    host: process.env.DB_HOST,
    port: parseInt(process.env.DB_PORT || '5432'),
    password: process.env.DB_PASSWORD,
  }
};
```
</details>

### Logging

**Before** — writing to files or Windows Event Log:

<details open>
<summary><strong>.NET</strong></summary>

```csharp
// Legacy pattern — logs to a file, not collected by platform
Log.Logger = new LoggerConfiguration()
    .WriteTo.File("C:\\Logs\\myapp.log")
    .CreateLogger();
```
</details>

<details>
<summary><strong>Java</strong></summary>

```xml
<!-- logback.xml — writing to file -->
<appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>/var/log/myapp.log</file>
    <encoder><pattern>%d %level %msg%n</pattern></encoder>
</appender>
```
</details>

<details>
<summary><strong>Node.js</strong></summary>

```javascript
// Legacy — writing to file
const fs = require('fs');
const logStream = fs.createWriteStream('/var/log/myapp.log', { flags: 'a' });
logStream.write(`${new Date().toISOString()} INFO: ${message}\n`);
```
</details>

**After** — structured JSON to stdout, collected automatically by the platform:

<details open>
<summary><strong>.NET</strong></summary>

```csharp
// Program.cs
builder.Host.UseSerilog((context, config) => config
    .WriteTo.Console(new RenderedCompactJsonFormatter())
    .Enrich.WithProperty("ApplicationName", "my-api")
    .Enrich.WithCorrelationId());
```
</details>

<details>
<summary><strong>Java (Spring Boot / Logback)</strong></summary>

```xml
<!-- logback-spring.xml — structured JSON to stdout -->
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <customFields>{"applicationName":"my-api"}</customFields>
    </encoder>
</appender>
<root level="INFO">
    <appender-ref ref="STDOUT"/>
</root>
```
</details>

<details>
<summary><strong>Node.js (Pino)</strong></summary>

```javascript
// Structured JSON to stdout via Pino
const pino = require('pino');
const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label }),
  },
  base: { applicationName: 'my-api' },
});
logger.info({ correlationId: req.headers['x-correlation-id'] }, 'Request received');
```
</details>

### Health Checks

**Before** — no health endpoints, or a simple "200 OK" that checks nothing:

<details open>
<summary><strong>.NET</strong></summary>

```csharp
app.MapGet("/livez", () => "OK"); // Tells you nothing useful
```
</details>

<details>
<summary><strong>Java</strong></summary>

```java
@GetMapping("/livez")
public String health() { return "OK"; } // Tells you nothing useful
```
</details>

<details>
<summary><strong>Node.js</strong></summary>

```javascript
app.get('/livez', (req, res) => res.send('OK')); // Tells you nothing useful
```
</details>

**After** — meaningful checks with separate liveness and readiness:

<details open>
<summary><strong>.NET</strong></summary>

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
app.MapHealthChecks("/livez", new HealthCheckOptions
{
    Predicate = _ => false // No dependency checks — just "am I running?"
});

// Readiness — can I serve traffic? Check dependencies here.
app.MapHealthChecks("/readyz", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```
</details>

<details>
<summary><strong>Java (Spring Boot Actuator)</strong></summary>

```yaml
# application.yaml — Spring Boot Actuator health groups
management:
  endpoints.web.exposure.include: health,prometheus
  endpoint.health:
    show-details: never
    group:
      liveness:
        include: ping          # /actuator/health/liveness — no dependency checks
      readiness:
        include: db,redis      # /actuator/health/readiness — checks dependencies
  health:
    livenessstate.enabled: true
    readinessprobe.enabled: true
```
</details>

<details>
<summary><strong>Node.js (Express)</strong></summary>

```javascript
// Liveness — is the process alive? Don't check dependencies.
app.get('/livez', (req, res) => res.status(200).json({ status: 'ok' }));

// Readiness — can I serve traffic? Check dependencies.
app.get('/readyz', async (req, res) => {
  try {
    await db.query('SELECT 1');
    await redisClient.ping();
    res.status(200).json({ status: 'ready' });
  } catch (err) {
    res.status(503).json({ status: 'not ready', error: err.message });
  }
});
```
</details>

### Resilient HTTP Calls

**Before** — manually creating HTTP clients, no retry logic:

<details open>
<summary><strong>.NET</strong></summary>

```csharp
// Legacy — socket exhaustion, no resilience
var client = new HttpClient();
var response = await client.GetAsync("http://other-service/api/data");
```
</details>

<details>
<summary><strong>Java</strong></summary>

```java
// Legacy — no retry, no timeout, no circuit breaker
HttpURLConnection conn = (HttpURLConnection)
    new URL("http://other-service/api/data").openConnection();
InputStream is = conn.getInputStream();
```
</details>

<details>
<summary><strong>Node.js</strong></summary>

```javascript
// Legacy — no retry, no timeout management
const resp = await fetch('http://other-service/api/data');
```
</details>

**After** — managed clients with retry and circuit breaker:

<details open>
<summary><strong>.NET (IHttpClientFactory + Standard Resilience)</strong></summary>

```csharp
// Program.cs — uses Microsoft.Extensions.Http.Resilience (not the legacy Microsoft.Extensions.Http.Polly)
builder.Services.AddHttpClient("OrderService", client =>
{
    client.BaseAddress = new Uri(
        builder.Configuration["Services:OrderService:BaseUrl"]!);
})
.AddStandardResilienceHandler();
// Includes: rate limiter, total timeout (30s), retry (3x exponential + jitter),
// circuit breaker (10% failure ratio), and per-attempt timeout (10s) — all preconfigured.

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
</details>

<details>
<summary><strong>Java (Spring Boot + Resilience4j)</strong></summary>

```java
// application.yaml — resilience config
resilience4j:
  retry:
    instances:
      orderService:
        maxAttempts: 3
        waitDuration: 200ms
        exponentialBackoffMultiplier: 2
  circuitbreaker:
    instances:
      orderService:
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        slidingWindowSize: 10

// OrderClient.java
@Service
public class OrderClient {
    private final WebClient webClient;

    public OrderClient(WebClient.Builder builder,
                       @Value("${services.order.base-url}") String baseUrl) {
        this.webClient = builder.baseUrl(baseUrl).build();
    }

    @Retry(name = "orderService")
    @CircuitBreaker(name = "orderService", fallbackMethod = "fallback")
    public Mono<Order> getOrder(int id) {
        return webClient.get().uri("/api/orders/{id}", id)
            .retrieve().bodyToMono(Order.class);
    }
}
```
</details>

<details>
<summary><strong>Node.js (cockatiel)</strong></summary>

```javascript
const { retry, circuitBreaker, wrap, handleAll, ExponentialBackoff } = require('cockatiel');

const retryPolicy = retry(handleAll, {
  maxAttempts: 3,
  backoff: new ExponentialBackoff({ initialDelay: 200 }),
});
const breaker = circuitBreaker(handleAll, {
  halfOpenAfter: 30_000,
  breaker: { threshold: 0.5, duration: 10_000, minimumRps: 5 },
});
const policy = wrap(retryPolicy, breaker);

async function getOrder(id) {
  return policy.execute(() =>
    fetch(`${process.env.ORDER_SERVICE_URL}/api/orders/${id}`, {
      signal: AbortSignal.timeout(10_000),
    }).then(r => r.json())
  );
}
```
</details>

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
