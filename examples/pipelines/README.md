# CI/CD Pipeline Templates

These templates implement the pipeline stages described in [CI/CD & GitOps Standards](../../cicd.html).

| File | Platform | Description |
|------|----------|-------------|
| `github-actions.yml` | GitHub Actions | Full CI pipeline for .NET 8 apps — lint, test, build, scan, push, GitOps update |
| `tekton-pipeline.yaml` | Tekton (OpenShift Pipelines) | Kubernetes-native pipeline with PipelineRun example |

**Before using:** configure the secrets and parameters documented in each file's header comments.
