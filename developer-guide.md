---
layout: default
title: Developer Inner Loop
nav_order: 10
---

# Developer Inner Loop

The standards in this document describe what production looks like. This section covers how to develop against those standards locally — so issues are caught on your laptop, not in the CI pipeline.

## Local Development Tooling

| Tool | What It Does | When to Use |
|------|-------------|-------------|
| **Podman Desktop** | Build and run containers locally without Docker daemon | Building and testing your container image before pushing |
| **odo** (OpenShift Do) | Inner-loop dev tool that syncs code changes to a running container on the cluster | Iterating quickly against a real OpenShift environment without full CI/CD cycles |
| **Podman Compose** | Run multi-container apps locally (API + database + Redis) | Testing your app with its dependencies before deploying to the cluster |
| **Dev Containers** (VS Code / JetBrains) | Develop inside a container that matches your production environment | Ensuring your local dev environment matches prod (same OS, same runtime) |

## Local Checklist

Before you push, verify these locally — they're the same things the platform will check:

```bash
# 1. Does it build as a container?
podman build -t myapp:dev .

# 2. Does it run as non-root?
podman run --user 1001:0 myapp:dev

# 3. Does it start without hardcoded config?
podman run -e ConnectionStrings__Default="Server=localhost;..." myapp:dev

# 4. Do health endpoints work?
curl http://localhost:8080/livez    # Should return 200
curl http://localhost:8080/readyz   # Should return 200 (or 503 if no DB)

# 5. Are logs going to stdout (not files)?
podman logs <container-id>          # Should see structured JSON output

# 6. Does /metrics respond?
curl http://localhost:8080/metrics   # Should return Prometheus format
```

## Multi-Stage Dockerfile Template

This pattern keeps your production image small and secure:

```dockerfile
# Build stage — SDK image, not shipped to prod
FROM registry.access.redhat.com/ubi9/dotnet-100 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app

# Runtime stage — minimal image, what actually runs in prod
FROM registry.access.redhat.com/ubi9/dotnet-100-runtime AS runtime
WORKDIR /app
COPY --from=build /app .

# Run as non-root
USER 1001
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```
