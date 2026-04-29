# Application Modernization & Cloud-Native Standards for OpenShift

Best practices and prescriptive standards for application modernization and cloud-native development on Red Hat OpenShift Container Platform.

## Purpose

This document answers: **"What does a well-architected application look like on OpenShift?"**

It defines the non-negotiable requirements, recommended practices, and architectural patterns that every application team should follow — whether modernizing an existing app or building something new. It is organized around standards and policies, not just YAML examples.

## What's Covered

| Section | What It Answers |
|---------|-----------------|
| **Application Readiness Requirements** | What MUST my app do before it can run on OpenShift? |
| **Platform-Provided Capabilities & Citizenship** | What does the platform give me, what must I NOT reimplement, and what's my responsibility as a tenant? |
| **Architecture Standards by Tier** | What are the prescriptive patterns for UI, API, Data, and Storage tiers? |
| **Security Standards** | What are the non-negotiable security policies? |
| **CI/CD & GitOps Standards** | How MUST applications be built and deployed? |
| **Legacy Code Patterns → Cloud-Native Rewrites** | I write my legacy code this way — how do I rewrite it? |
| **Modernization Maturity Model** | Where is my app today, and what's the path to cloud-native? |
| **Extension Catalog** | What additional components are available when justified? |
| **Application Onboarding Decision Tree** | Is my app ready for the platform, and who needs to do what? |
| **Governance & Compliance Checklist** | How do I verify my app meets these standards? |

## Live Site

**[https://ultrajeff.github.io/ocp-reference-architecture/](https://ultrajeff.github.io/ocp-reference-architecture/)**

## How to Use

1. **Onboarding a new app?** Start with the [Onboarding Decision Tree](onboarding.md)
2. **Modernizing a legacy app?** Use the [Legacy Patterns → Cloud-Native Rewrites](legacy-patterns.md) and [Maturity Model](maturity-model.md)
3. **Building cloud-native?** Follow [Application Readiness](readiness.md) and [Architecture Standards](architecture.md)
4. **Setting up CI/CD?** See [CI/CD & GitOps Standards](cicd.md)
5. **Need more than the baseline?** Check the [Extension Catalog](extensions.md)
6. **Local dev workflow?** See the [Developer Inner Loop](developer-guide.md)

## Related Red Hat Resources

- [OCP 4.21 Architecture Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/architecture/index) — current platform architecture reference
- [Validated Patterns](https://validatedpatterns.io/patterns/) — GitOps-based, tested application stacks maintained across product lifecycles
- [Multicloud GitOps Pattern](https://validatedpatterns.io/patterns/multicloud-gitops/) — foundational pattern for GitOps-driven multi-cluster app management
- [Multicluster DevSecOps Pattern](https://validatedpatterns.io/patterns/devsecops/) — CI/CD pipelines with security gates, image scanning, and secured deployment
- [Red Hat Architecture and Design Patterns](https://developers.redhat.com/topics/red-hat-architecture-and-design-patterns) — portfolio architectures and solution patterns hub
