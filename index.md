---
layout: default
title: Home
nav_order: 1
---

# Application Modernization Standards for OpenShift

This site defines the standards, policies, and architectural patterns for deploying well-architected custom applications on Red Hat OpenShift Container Platform. It focuses on what happens **inside the application** — the code patterns, design decisions, and operational behaviors that make an app a good citizen on a shared platform — not just the infrastructure around it.

**Audience:** Application developers, security engineers, and operations/SRE teams. Successful modernization is a cross-functional effort — developers write the code, security validates the posture, and operations ensures the app is observable and sustainable in production.

It is prescriptive by design: **Required** items are non-negotiable for production workloads, **SHOULD** items are strongly recommended, and **MAY** items are available when justified by application needs.

---

## How to Use This Guide

| Starting Point | Where to Go |
|----------------|-------------|
| **Onboarding a new app?** | Start with the [Onboarding Decision Tree](onboarding.html) to assess readiness, then use the [Governance Checklist](onboarding.html#governance--compliance-checklist) |
| **Modernizing a legacy app?** | Use the [Legacy Patterns → Cloud-Native Rewrites](legacy-patterns.html) for code-level changes, and the [Maturity Model](maturity-model.html) to plan the journey |
| **Building cloud-native from scratch?** | Follow [Application Readiness](readiness.html) and [Architecture Standards](architecture.html) |
| **Setting up CI/CD?** | See [CI/CD & GitOps Standards](cicd.html) |
| **Need more than the baseline?** | Check the [Extension Catalog](extensions.html) |
| **Local dev workflow?** | See the [Developer Inner Loop](developer-guide.html) |
| **Need copy-paste manifests?** | Grab from the [Examples](examples/README.html) directory |
| **Looking up an acronym?** | Check the [Glossary](glossary.html) |

---

## Related Red Hat Resources

- [OCP 4.21 Architecture Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/architecture/index)
- [Validated Patterns](https://validatedpatterns.io/patterns/)
- [Multicloud GitOps Pattern](https://validatedpatterns.io/patterns/multicloud-gitops/)
- [Multicluster DevSecOps Pattern](https://validatedpatterns.io/patterns/devsecops/)
- [Red Hat Architecture and Design Patterns](https://developers.redhat.com/topics/red-hat-architecture-and-design-patterns)
- [OpenShift Runtime Security Best Practices](https://www.redhat.com/en/blog/openshift-runtime-security-best-practices)
- [14 Best Practices for Developing Applications on OpenShift](https://www.redhat.com/en/blog/14-best-practices-for-developing-applications-on-openshift)
