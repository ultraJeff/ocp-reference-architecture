---
layout: default
title: CI/CD & GitOps
nav_order: 5
---

# CI/CD & GitOps Standards

## Pipeline Requirements

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

## GitOps Repository Structure

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

## Environment Promotion

- **Dev → Stage**: Automated on merge to `main`. Image digest propagated via CI pipeline
- **Stage → Prod**: Requires manual approval gate (PR to `overlays/prod` in GitOps repo)
- Environment-specific configuration lives in Kustomize patches, not in the application code or image
