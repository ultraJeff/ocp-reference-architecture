---
layout: default
title: Glossary
nav_order: 11
---

# Glossary

Quick reference for acronyms and technical terms used throughout this guide.

---

## OpenShift & Kubernetes

| Term | Definition |
|------|------------|
| **OCP** | OpenShift Container Platform — Red Hat's enterprise Kubernetes distribution |
| **Namespace** | A Kubernetes logical boundary that isolates resources, workloads, and quotas. Typically one per application or environment |
| **Pod** | The smallest deployable unit in Kubernetes — one or more containers sharing network and storage |
| **Deployment** | A Kubernetes controller that manages replica sets and rolling updates for stateless applications |
| **StatefulSet** | A Kubernetes controller for stateful workloads with stable pod identities and ordered rollouts |
| **Service** | A stable network endpoint that load-balances traffic across pod replicas |
| **Route** | An OpenShift resource that exposes a Service to external traffic with TLS termination and path-based routing |
| **ConfigMap** | A Kubernetes object for storing non-sensitive configuration (URLs, feature flags) injected as environment variables or mounted files |
| **Secret** | A Kubernetes object for storing sensitive data (credentials, API keys) with base64 encoding |
| **PVC** | PersistentVolumeClaim — a request for persistent storage with a specific access mode (RWO, RWX, ROX) and storage class |
| **RWO** | ReadWriteOnce — PVC access mode allowing a single node to mount the volume as read-write |
| **RWX** | ReadWriteMany — PVC access mode allowing multiple nodes to mount the volume as read-write simultaneously |
| **ROX** | ReadOnlyMany — PVC access mode allowing multiple nodes to mount the volume as read-only |
| **Job** | A Kubernetes object that runs a container to completion for batch or one-time tasks |
| **CronJob** | A Kubernetes object that runs Jobs on a schedule, similar to Unix cron |
| **EmptyDir** | A temporary volume tied to a pod's lifecycle — discarded when the pod is deleted |
| **NetworkPolicy** | A Kubernetes resource defining firewall rules for pod-to-pod and pod-to-external communication |
| **ServiceAccount** | A Kubernetes identity assigned to pods that defines RBAC permissions for API access |
| **Ingress** | Inbound network traffic entering pods or the cluster from external sources |
| **Egress** | Outbound network traffic leaving pods or the cluster to external destinations |
| **CoreDNS** | Kubernetes' built-in DNS server that resolves Service names (e.g., `myapp.mynamespace.svc.cluster.local`) |
| **Operator** | A Kubernetes pattern where a custom controller manages the full lifecycle of an application or service |

## Security

| Term | Definition |
|------|------------|
| **SCC** | Security Context Constraint — OpenShift's mechanism for defining what security privileges a pod can request. `restricted-v2` is the enforced default |
| **RBAC** | Role-Based Access Control — Kubernetes mechanism for controlling who can perform which actions on cluster resources |
| **TLS** | Transport Layer Security (HTTPS). OpenShift Router handles TLS termination so applications don't manage certificates |
| **mTLS** | Mutual TLS — bidirectional encryption where both client and server authenticate via certificates, typically provided by a service mesh |
| **OIDC** | OpenID Connect — an identity layer on OAuth 2.0 for single sign-on and external identity provider integration |
| **CVE** | Common Vulnerabilities and Exposures — industry-standard identifiers for known security flaws |
| **Seccomp** | Secure Computing — a Linux kernel feature restricting which syscalls a process can make |
| **UBI** | Universal Base Image — Red Hat's minimal, security-maintained container base image |

## Observability

| Term | Definition |
|------|------------|
| **Liveness Probe** | A health check (`/livez`) that tells Kubernetes the process is alive. Should *not* check downstream dependencies |
| **Readiness Probe** | A health check (`/readyz`) that tells Kubernetes the pod can accept traffic. Return 503 when dependencies are unavailable |
| **Startup Probe** | An optional health check for slow-starting apps, preventing premature liveness probe kills |
| **ServiceMonitor** | A Prometheus Operator resource declaring that Prometheus should scrape a service's `/metrics` endpoint |
| **Structured Logging** | Emitting logs as JSON with explicit fields (timestamp, level, message, correlationId) rather than plain text |
| **RED Method** | A metrics methodology: **R**ate (requests/sec), **E**rrors (error rate), **D**uration (latency) |
| **OpenTelemetry** | An open-source observability framework for metrics, logs, and distributed traces across multiple languages |
| **SRE** | Site Reliability Engineering — a discipline focused on reliability, observability, and incident response |

## CI/CD & GitOps

| Term | Definition |
|------|------------|
| **GitOps** | A deployment pattern where desired state is declared in Git and a controller (ArgoCD) continuously syncs the cluster to match |
| **ArgoCD** | A Kubernetes-native GitOps controller that watches a Git repo and reconciles cluster state. Distributed by Red Hat as OpenShift GitOps |
| **Kustomize** | A template-free Kubernetes manifest customization tool using base + overlay directories (e.g., base, dev, stage, prod) |
| **Tekton** | A cloud-native CI/CD framework that runs pipeline tasks as Kubernetes pods |
| **DevSecOps** | Integrating security checks (scanning, compliance, signing) into every stage of the CI/CD pipeline |
| **Quay** | Red Hat's container image registry with vulnerability scanning and access control |

## Scaling & Resilience

| Term | Definition |
|------|------------|
| **HPA** | Horizontal Pod Autoscaler — automatically scales pod replicas based on CPU, memory, or custom metrics |
| **KEDA** | Kubernetes Event-Driven Autoscaling — scales based on external signals like queue depth or message count |
| **Circuit Breaker** | A resilience pattern that stops requests to a failing service and returns a fallback, preventing cascading failures |
| **Graceful Shutdown** | Responding to SIGTERM by draining in-flight requests, closing connections, and flushing buffers before exiting |
| **SIGTERM** | The signal Kubernetes sends to a pod before killing it, triggering graceful shutdown (default 30s grace period) |
| **Bulkhead** | A resilience pattern that isolates resources (threads, connections) to prevent one component's failure from affecting others |

## Red Hat Platform Services

| Term | Definition |
|------|------------|
| **ACS** | Advanced Cluster Security — Red Hat's container security platform for image scanning, admission control, and runtime protection |
| **ACM** | Advanced Cluster Management — Red Hat's multi-cluster management solution for deployment, governance, and DR |
| **ODF** | OpenShift Data Foundation — Red Hat's software-defined storage providing block, file, and S3-compatible object storage |
| **RHOAI** | Red Hat OpenShift AI — an integrated AI/ML platform for model serving, notebooks, and training workloads |
| **3scale** | Red Hat's API management platform for rate limiting, authentication, analytics, and developer portals |
| **AMQ Streams** | Red Hat's Kafka distribution for event streaming, log aggregation, and data pipelines |
| **ClusterLogForwarder** | An OpenShift resource that collects container logs and forwards them to external systems (Loki, Splunk, etc.) |
| **External Secrets Operator** | A Kubernetes Operator that syncs secrets from external vaults (HashiCorp Vault, AWS Secrets Manager) into Kubernetes Secrets |
| **MTA** | Migration Toolkit for Applications — a Red Hat tool that scans legacy code and identifies patterns needing modernization |

## .NET Specific

| Term | Definition |
|------|------------|
| **Kestrel** | The lightweight, cross-platform HTTP server built into ASP.NET Core — the recommended server for containers (not IIS or HTTP.sys) |
| **Polly** | A .NET resilience library implementing retry, circuit breaker, timeout, and bulkhead patterns |
| **Serilog** | A .NET structured logging library for writing JSON logs to stdout with enrichment (correlation IDs, app name) |
| **IHttpClientFactory** | A .NET factory for creating pooled HttpClient instances with built-in resilience policies |
| **IDistributedCache** | A .NET abstraction for distributed caching (Redis, SQL) enabling externalized session state |
| **Microsoft.Data.SqlClient** | The modern .NET SQL Server driver, replacing the legacy System.Data.SqlClient |
