# OpenShift Reference Architecture

A baseline application reference architecture for deploying well-architected applications on Red Hat OpenShift Container Platform.

## What This Is

This repo provides a foundational, opinionated architecture for how a typical application should land on OpenShift — covering the components most applications need before any app-specific customization:

- **Presentation Tier** — Deployment, Service, Route, HPA
- **API / Business Logic Tier** — Deployment, Service, Route, HPA
- **Data Tier** — Operator-managed database or StatefulSet with persistent storage
- **Observability** — Structured logging, Prometheus metrics (ServiceMonitor), distributed tracing
- **Security** — Restricted SCCs, NetworkPolicies (default-deny), ExternalSecrets
- **CI/CD** — Tekton or GitHub Actions + ArgoCD GitOps deployment
- **Environment Promotion** — Kustomize overlays for dev / stage / prod

## Contents

| File | Description |
|------|-------------|
| [baseline-app-architecture.md](baseline-app-architecture.md) | Full reference architecture with diagrams, YAML examples, and extension points |

## How to Use

1. Start with the [baseline architecture](baseline-app-architecture.md) as your foundation
2. Identify which [extension points](baseline-app-architecture.md#9-extension-points) apply to your application (Kafka, Service Mesh, KEDA, etc.)
3. Layer those components on top of the baseline

## Related Red Hat Resources

- [OCP 4.21 Architecture Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/architecture/index) — current platform architecture reference
- [Validated Patterns](https://validatedpatterns.io/patterns/) — GitOps-based, tested application stacks maintained across product lifecycles
- [Multicloud GitOps Pattern](https://validatedpatterns.io/patterns/multicloud-gitops/) — foundational pattern for GitOps-driven multi-cluster app management
- [Multicluster DevSecOps Pattern](https://validatedpatterns.io/patterns/devsecops/) — CI/CD pipelines with security gates, image scanning, and secured deployment
- [Red Hat Architecture and Design Patterns](https://developers.redhat.com/topics/red-hat-architecture-and-design-patterns) — portfolio architectures and solution patterns hub
