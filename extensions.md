---
layout: default
title: Extensions & Reference Architecture
nav_order: 8
---

# Extension Catalog & Baseline Architecture

## Extension Catalog

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

## Baseline Architecture Diagram

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
  │  │  .NET 10 / Kestrel   │    │  .NET 10 / ASP.NET Core  │               │
  │  │  Deployment (≥2)     │    │  Deployment (≥2)          │               │
  │  │  Service + Route     │    │  Service + Route          │               │
  │  │  HPA (CPU ≤70%)      │    │  HPA (CPU ≤70%)           │               │
  │  │  /livez + /readyz    │    │  /livez + /readyz         │               │
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
