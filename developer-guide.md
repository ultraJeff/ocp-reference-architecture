---
layout: default
title: Developer Inner Loop
nav_order: 10
---

# Developer Inner Loop

The standards in this document describe what production looks like. This section covers how to develop against those standards — whether in a cloud workspace or on your laptop — so issues are caught early, not in the CI pipeline.

## OpenShift Dev Spaces

[OpenShift Dev Spaces](https://developers.redhat.com/products/openshift-dev-spaces/overview) is the platform-provided cloud development environment. It runs on the cluster and gives every developer a consistent, pre-configured workspace without requiring local tooling setup.

| Capability | How It Works |
|------------|-------------|
| **Web IDE** | Browser-based VS Code experience — no local install required. Open a workspace from a Git repo URL and start coding immediately |
| **Remote connection from local IDE** | Connect VS Code or JetBrains locally to a Dev Spaces workspace running on the cluster. You get your preferred IDE with cluster-side compute, networking, and dependencies |
| **Devfile-driven** | Workspaces are defined by a `devfile.yaml` in your repo — runtime, tools, endpoints, and commands are version-controlled and shared across the team |
| **Pre-built dependencies** | Container images, SDKs, CLIs, and database clients are available in the workspace without manual setup |
| **Direct cluster access** | Workspaces run inside the cluster with access to services, databases, and APIs via internal DNS — no VPN or port-forwarding required |

**When to use Dev Spaces:** When you want a zero-setup development experience, need direct access to cluster services, or want to standardize the dev environment across the team. Especially valuable for onboarding new developers — they open a browser and start working.

## Local Development Tooling

For developers who prefer working on their laptop, these tools support the same inner-loop workflow:

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
#    .NET:    podman run -e ConnectionStrings__Default="Server=localhost;..." myapp:dev
#    Java:    podman run -e SPRING_DATASOURCE_URL="jdbc:postgresql://localhost/mydb" myapp:dev
#    Node.js: podman run -e DB_HOST=localhost -e DB_PORT=5432 myapp:dev
podman run -e DB_HOST=localhost myapp:dev

# 4. Do health endpoints work?
curl http://localhost:8080/livez     # Should return 200
curl http://localhost:8080/readyz    # Should return 200 (or 503 if no DB)

# 5. Are logs going to stdout (not files)?
podman logs <container-id>          # Should see structured JSON output

# 6. Does /metrics respond?
curl http://localhost:8080/metrics   # Should return Prometheus format
```

## Multi-Stage Dockerfile Templates

These patterns keep your production image small and secure. Pick the one that matches your runtime.

### .NET 10

```dockerfile
FROM registry.access.redhat.com/ubi9/dotnet-100 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app

FROM registry.access.redhat.com/ubi9/dotnet-100-runtime AS runtime
WORKDIR /app
COPY --from=build /app .
USER 1001
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Java (Quarkus)

```dockerfile
FROM registry.access.redhat.com/ubi9/openjdk-21 AS build
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests -Dquarkus.package.jar.type=uber-jar

FROM registry.access.redhat.com/ubi9/openjdk-21-runtime AS runtime
WORKDIR /app
COPY --from=build /build/target/*-runner.jar app.jar
USER 1001
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Java (Spring Boot)

```dockerfile
FROM registry.access.redhat.com/ubi9/openjdk-21 AS build
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests

FROM registry.access.redhat.com/ubi9/openjdk-21-runtime AS runtime
WORKDIR /app
COPY --from=build /build/target/*.jar app.jar
USER 1001
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Node.js

```dockerfile
FROM registry.access.redhat.com/ubi9/nodejs-20 AS build
WORKDIR /build
COPY package*.json .
RUN npm ci
COPY . .
RUN npm run build --if-present && npm prune --production

FROM registry.access.redhat.com/ubi9/nodejs-20-minimal AS runtime
WORKDIR /app
COPY --from=build /build/node_modules ./node_modules
COPY --from=build /build/dist ./dist
COPY --from=build /build/package.json .
USER 1001
EXPOSE 8080
ENTRYPOINT ["node", "dist/server.js"]
```
