---
layout: default
title: Examples
nav_order: 12
---

# Example Manifests & Pipelines

Copy-paste-ready Kubernetes/OpenShift manifests and CI/CD pipelines that implement the standards described in this guide.

## Kubernetes Manifests (Kustomize)

The `base/` directory contains shared manifests for a two-tier app (UI + API). Overlays customize for each environment.

```bash
# Preview dev manifests
kustomize build examples/overlays/dev

# Preview prod manifests
kustomize build examples/overlays/prod

# Apply to cluster
kustomize build examples/overlays/dev | oc apply -f -
```

**Customize before using:** replace image references, database addresses, namespace names, and DNS hostnames with your actual values.

## CI/CD Pipelines

The `pipelines/` directory includes templates for:
- **GitHub Actions** — `.github/workflows`-compatible workflow for .NET 8 apps
- **Tekton** — Kubernetes-native pipeline with PipelineRun example

See [CI/CD & GitOps Standards](cicd.html) for the full pipeline requirements these templates implement.
