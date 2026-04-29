---
layout: default
title: Planned Additions
nav_order: 99
---

# Planned Additions

These sections are scoped and ready to be developed. Each outlines the problem, proposed structure, and key content.

---

## 1. Troubleshooting Guide

**Problem:** Teams deploying to OpenShift for the first time hit the same issues repeatedly. There's no centralized guide for diagnosing common failures.

**Proposed structure:**

```
troubleshooting.md
├── Pod won't start
│   ├── CrashLoopBackOff — reading logs, common causes (non-root, missing config, port mismatch)
│   ├── ImagePullBackOff — registry auth, image name typos, SHA digest mismatches
│   ├── CreateContainerConfigError — missing ConfigMaps/Secrets
│   └── Pending — insufficient resources, node affinity, PVC binding
├── Pod starts but traffic doesn't reach it
│   ├── Service selector mismatch
│   ├── NetworkPolicy blocking traffic
│   ├── Route misconfiguration (wrong port, TLS issues)
│   └── Readiness probe failing
├── Application errors
│   ├── OOMKilled — memory limits too low, memory leaks
│   ├── Connection refused to database — NetworkPolicy egress, DNS resolution
│   ├── Intermittent 503s — HPA scaling lag, readiness probe timing
│   └── Slow startup kills — startup probe configuration
└── Debugging commands
    ├── oc logs / oc describe / oc get events
    ├── oc debug (run a debug container)
    ├── oc rsh (exec into running pod)
    └── NetworkPolicy troubleshooting with oc adm policy
```

**Format:** Problem → Symptoms → Diagnosis commands → Fix. Short, scannable entries — not long explanations.

---

## 2. Alerting Catalog

**Problem:** The guide requires PrometheusRules but doesn't specify *what* to alert on. Teams either alert on nothing or create noisy, unhelpful alerts.

**Proposed structure:**

```
alerting.md
├── Baseline alerts (every app should have these)
│   ├── High error rate (5xx > 5% for 5m) — critical
│   ├── High latency (p99 > 2s for 5m) — warning
│   ├── Pod restarts (> 3 in 15m) — warning
│   ├── Pod not ready (> 5m) — critical
│   ├── HPA at max replicas (sustained 15m) — warning
│   └── Container OOMKilled — critical
├── Database connectivity alerts
│   ├── Connection pool exhaustion — critical
│   ├── Query latency spike — warning
│   └── Connection errors — critical
├── Resource alerts
│   ├── CPU throttling (> 25% for 10m) — warning
│   ├── Memory approaching limit (> 90% for 5m) — warning
│   └── PVC usage > 80% — warning
├── SLA/SLO alerts
│   ├── Availability (success rate < 99.9% over 30m) — critical
│   └── Error budget burn rate — warning
└── Alert hygiene
    ├── Every alert must have a runbook link
    ├── Severity definitions (critical = pages someone, warning = ticket)
    └── Anti-patterns: alerting on every 500, alerting on metrics you can't act on
```

**Format:** Each alert includes: PromQL expression, threshold, severity, why it matters, and what to do when it fires. Include a complete PrometheusRule YAML that teams can customize.

---

## 3. Resource Sizing Guide

**Problem:** The guide says "set accurate CPU/memory requests" but doesn't help teams figure out what those numbers should be. Under-requesting causes OOMKills; over-requesting wastes cluster capacity.

**Proposed structure:**

```
resource-sizing.md
├── Concepts
│   ├── Requests vs. limits — what they mean for scheduling and throttling
│   ├── CPU throttling vs. OOMKill — different failure modes
│   └── Quality of Service classes (Guaranteed, Burstable, BestEffort)
├── Starting points by runtime
│   ├── .NET 10 API:     requests 250m/256Mi, limits 1/512Mi
│   ├── .NET 10 UI:      requests 100m/128Mi, limits 500m/256Mi
│   ├── Java (Quarkus): requests 250m/256Mi, limits 1/512Mi
│   ├── Java (Spring):  requests 500m/512Mi, limits 1/1Gi
│   ├── Node.js:        requests 100m/128Mi, limits 500m/256Mi
│   └── Static nginx:   requests 10m/32Mi,   limits 100m/64Mi
├── Right-sizing process
│   ├── Deploy with generous limits
│   ├── Run realistic load (not just smoke tests)
│   ├── Read actual usage from Prometheus (container_cpu_usage_seconds_total, container_memory_working_set_bytes)
│   ├── Set requests to p95 of observed usage
│   ├── Set limits to 2x requests (or p99 + headroom)
│   └── Re-evaluate after each major release
├── Java-specific: JVM heap vs. container limits
│   ├── -XX:MaxRAMPercentage=75.0
│   ├── Why container memory must exceed heap (metaspace, threads, native)
│   └── Common pitfall: 512Mi limit with default heap = OOMKilled
└── HPA tuning
    ├── CPU target 70% — why, and when to adjust
    ├── Scale-up vs. scale-down behavior
    └── Custom metrics via KEDA (when CPU isn't the bottleneck)
```

**Format:** Decision-oriented, not reference-oriented. "If you see X, do Y." Include PromQL queries teams can run to measure their actual usage, and a table of starting-point values they can copy.

---

## Priority Order

1. **Troubleshooting** — highest impact, directly unblocks teams stuck on deployment failures
2. **Alerting catalog** — builds on the existing PrometheusRule example and fills the "what to monitor" gap
3. **Resource sizing** — important but less urgent; teams can iterate on sizing once deployed
