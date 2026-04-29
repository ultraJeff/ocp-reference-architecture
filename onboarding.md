---
layout: default
title: Onboarding & Governance
nav_order: 9
---

# Application Onboarding & Governance

## Application Onboarding Decision Tree

Use this decision tree when evaluating a custom application for deployment on the platform. It provides a standard intake assessment that spans development, security, and operations concerns.

```
Is the application containerized?
├── NO → Can it be containerized?
│   ├── NO (hard OS/kernel dependencies, proprietary runtime)
│   │   → Not a candidate for OpenShift. Escalate for alternative hosting.
│   └── YES → Proceed to containerization (target Maturity Level 1)
│       ├── .NET apps: target .NET 10 (LTS) on UBI 9 base images
│       └── Other stacks: target supported runtime on UBI base images
│
└── YES → Does it run as non-root?
    ├── NO → Can it be modified to run non-root?
    │   ├── YES → Fix and re-assess
    │   └── NO → Requires SCC exception (security team approval required)
    │
    └── YES → Does it rely on local file storage or file shares?
        ├── YES → Identify usage patterns
        │   └── See Storage Tier for migration alternatives
        │
        └── NO → Does it externalize all configuration?
            ├── NO → Migrate hardcoded config to env vars / ConfigMaps / Secrets
            │
            └── YES → Does it implement health probes?
                ├── NO → Implement /livez and /readyz endpoints
                │
                └── YES → Does it log structured output to stdout?
                    ├── NO → Migrate logging to structured JSON on stdout
                    │
                    └── YES → Does it expose /metrics?
                        ├── NO → Add Prometheus metrics endpoint
                        │
                        └── YES → ✅ Ready for platform onboarding
                            └── Proceed to Governance Checklist below
```

### Intake Responsibilities by Role

Platform onboarding is not just a developer exercise. Successful onboarding requires coordination across teams:

| Role | Responsibility During Onboarding |
|------|----------------------------------|
| **Application Developer** | Containerize the app, implement health probes, externalize config, add metrics/logging, eliminate file share dependencies |
| **Security Engineer** | Review SCC requirements, validate NetworkPolicy design, approve image base and scanning results, confirm secrets management approach |
| **Operations / SRE** | Define resource quotas, validate scaling strategy, ensure monitoring/alerting is wired up, confirm runbook and on-call readiness |
| **Platform Team** | Provision namespace, configure GitOps sync, validate CI/CD pipeline integration, approve external network egress rules |
| **Database / Storage Team** | Validate connectivity from cluster to external databases, confirm backup/restore procedures, advise on storage alternatives for file shares |

---

## Governance & Compliance Checklist

Use this checklist during application onboarding and periodic reviews.

### Pre-Deployment Gate (all required)

- [ ] Container runs as non-root with `restricted-v2` SCC
- [ ] Readiness and liveness probes implemented and tested
- [ ] Structured logging to stdout (JSON with correlation IDs)
- [ ] `/metrics` endpoint exposed with RED metrics
- [ ] ServiceMonitor and PrometheusRule deployed
- [ ] All config externalized (no hardcoded URLs, credentials, or feature flags)
- [ ] Secrets sourced from External Secrets Operator (prod) or Kubernetes Secrets (dev/stage)
- [ ] No secrets in container images or Git repositories
- [ ] NetworkPolicies deployed (default-deny + explicit allow)
- [ ] Images built in CI pipeline from approved base images (UBI)
- [ ] Images scanned by ACS with no critical/high unpatched CVEs
- [ ] Production image references use SHA digests, not mutable tags
- [ ] GitOps repo with Kustomize overlays for all target environments
- [ ] ArgoCD Application configured with `selfHeal` and `prune` enabled
- [ ] Minimum 2 replicas in production with HPA configured
- [ ] Resource requests and limits set on all containers
- [ ] Graceful shutdown handler implemented (handles SIGTERM)
- [ ] Database connectivity tested under pod restart/reschedule conditions (connection pooling, transient fault handling)
- [ ] Backup and restore procedure documented and tested (data tier — whether in-cluster or external)

### Post-Deployment Verification (SHOULD)

- [ ] Load test completed against staging environment
- [ ] Alerts fire correctly under simulated failure conditions
- [ ] Runbook exists for common failure scenarios
- [ ] On-call team trained on application architecture and failure modes

---

## What Is a Runbook?

A runbook is a step-by-step guide that tells an on-call engineer what to do when something goes wrong with your application — **at 2 AM, without context, having never seen your code**. It's the difference between "figure it out" and "follow these steps." Every production application needs one.

A good runbook is:
- **Specific to your app**, not generic Kubernetes troubleshooting
- **Action-oriented** — tells you what to run, not what to think about
- **Kept next to the code** — in the app's GitOps repo or wiki, not a shared drive no one can find

### Runbook Template

Use this as a starting point. Fill in the sections relevant to your application.

```markdown
# [App Name] — Operations Runbook

## Application Overview
- **What it does:** [1-2 sentence description]
- **Team / owner:** [team name, Slack channel, PagerDuty service]
- **Architecture:** [UI + API + external MSSQL, or whatever applies]
- **Namespace(s):** [myapp-dev, myapp-stage, myapp-prod]
- **GitOps repo:** [link]
- **Dashboards:** [link to Grafana/Prometheus dashboard]

## Key Endpoints
| Endpoint | Purpose |
|----------|---------|
| `/livez` | Liveness — is the process alive? |
| `/readyz` | Readiness — can it serve traffic? |
| `/metrics` | Prometheus metrics |
| [app-specific] | [e.g., `/api/v1/status` for business health] |

## Common Failure Scenarios

### Pods are crash-looping
1. Check pod logs: `oc logs -f deployment/myapp-api -n myapp-prod`
2. Check events: `oc get events -n myapp-prod --sort-by=.lastTimestamp`
3. Common causes:
   - Database unreachable → check connectivity to [DB host:port]
   - Config missing → verify ConfigMap/Secret exists and is mounted
   - OOM killed → check `oc describe pod` for OOMKilled reason, increase memory limit

### High error rate (5xx responses)
1. Check dashboard: [link]
2. Check logs for exceptions: `oc logs deployment/myapp-api -n myapp-prod | grep -i error`
3. Check downstream dependencies:
   - Database: [how to verify]
   - External API: [how to verify]
4. If isolated to this app: restart deployment `oc rollout restart deployment/myapp-api -n myapp-prod`

### High latency
1. Check dashboard: [link]
2. Check database query performance: [how to verify]
3. Check HPA status: `oc get hpa -n myapp-prod` — are we at max replicas?
4. Check resource usage: `oc adm top pods -n myapp-prod`

### Cannot connect to database
1. Verify database is reachable from cluster: `oc debug deployment/myapp-api -- curl -v telnet://[db-host]:[db-port]`
2. Check Secret has correct credentials: `oc get secret myapp-db-credentials -n myapp-prod -o jsonpath='{.data}'`
3. Check NetworkPolicy allows egress to database host
4. Contact DBA team: [contact info]

## Rollback Procedure
1. Identify last known good commit in GitOps repo
2. Revert in GitOps repo or use ArgoCD to sync to previous commit
3. Verify rollback: check `/readyz` endpoint returns 200

## Escalation Path
1. **L1 (on-call):** [team/person], [Slack channel]
2. **L2 (app owner):** [team/person]
3. **L3 (platform team):** [Slack channel / PagerDuty]
4. **Database issues:** [DBA team contact]
```
