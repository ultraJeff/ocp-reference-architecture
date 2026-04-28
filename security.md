---
layout: default
title: Security Standards
nav_order: 4
---

# Security Standards

These are non-negotiable. Exceptions require documented approval from the security team.

## Pod Security

All workloads comply with the `restricted-v2` Security Context Constraint:

- `runAsNonRoot: true`
- `allowPrivilegeEscalation: false`
- `capabilities.drop: [ALL]`
- `seccompProfile.type: RuntimeDefault`

Any request for elevated SCCs (`anyuid`, `privileged`) requires a written exception with justification, time-bound approval, and a remediation plan.

## Network Segmentation

Every application namespace deploys NetworkPolicies:

1. **Default deny all ingress** — baseline policy, no exceptions
2. **Explicit allow rules** — whitelist traffic between tiers:
   - Router → UI (from `network.openshift.io/policy-group: ingress` namespace)
   - UI → API (pod-to-pod within namespace)
   - API → external database (egress to specific host IP and port, e.g., MSSQL on port 1433)
   - API → other external services (egress rules scoped to specific hosts/CIDRs as needed)
3. **No open namespaces** — a namespace without NetworkPolicies will not pass security review

## Image Provenance

- Images are pulled from an approved registry (Quay, OpenShift internal registry, Red Hat registry)
- Images are built in a CI pipeline — no `docker build` on developer laptops pushed to prod
- Images pass ACS vulnerability scanning before deployment
- Base images use Red Hat UBI (Universal Base Image) or an approved alternative. For .NET workloads, use `registry.access.redhat.com/ubi8/dotnet-80-runtime` (runtime) or `ubi8/dotnet-80` (SDK) images
- Image tags use immutable references (SHA digests) in production GitOps manifests, not `:latest`

## RBAC

- Application ServiceAccounts follow least-privilege — only the permissions the workload actually needs
- Do not use `cluster-admin` or `admin` ClusterRoles for application workloads
- Namespace-scoped Roles are preferred over ClusterRoles

## Application-Level Security

The standards above cover how the platform protects workloads. These standards cover what happens **inside your code** — the security practices every developer owns:

| Practice | What to Do | What NOT to Do |
|----------|-----------|----------------|
| **Input validation** | Validate and sanitize all input at system boundaries (API endpoints, message consumers, file uploads). Use allow-lists, not deny-lists | Trust input from other internal services without validation. Internal ≠ trusted — a compromised upstream poisons everything downstream |
| **Avoid shelling out** | Use libraries and SDKs for file operations, HTTP calls, and system interactions | Call `Process.Start()`, `Runtime.exec()`, or backtick commands with user-supplied input. This is command injection waiting to happen |
| **Dependency hygiene** | Pin dependency versions. Run `dotnet list package --vulnerable` (or equivalent) in CI. Update regularly | Use wildcard version ranges in production. Ignore vulnerability scan results from ACS/Quay |
| **Don't log secrets** | Scrub sensitive fields (tokens, passwords, SSNs, PII) before logging. Use structured logging with explicit field selection | Log full request/response bodies in production. Use `ToString()` on objects that may contain credentials |
| **Read-only root filesystem** | Set `readOnlyRootFilesystem: true` in your security context. Write temp data to `EmptyDir` volumes mounted at `/tmp` | Assume your container needs a writable filesystem. Most apps don't — and a read-only root blocks many exploit techniques |
| **Minimize base image** | Use runtime-only images (`ubi8/dotnet-80-runtime`, not the SDK image). Use multi-stage builds to keep build tools out of production | Ship the SDK, build tools, or debugging utilities in your production image. Smaller image = smaller attack surface |
