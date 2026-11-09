# Fintech Enterprise GitOps Manifests

This repository contains the shared microservice Helm chart, environment
values, and Argo CD configuration for the fintech platform.

- Dev is deployed manually by the source repository's GitHub Actions workflow
  to the existing EKS namespace `dev`; `enterprise-gitops/environments/dev/`
  is reference configuration and is not watched by the ApplicationSet.
- Staging and production are managed by Argo CD from
  `enterprise-gitops/environments/<environment>/<service>/values.yaml`.
  The source repository's promotion workflows update the image repository,
  immutable tag, and digest.
- The ApplicationSet targets the local cluster where Argo CD runs. Environment
  names provide distinct namespaces/configuration, not separate clusters.

See the [manifest repository runbook](RUNBOOK.md) for bootstrap, operating,
validation, promotion handoffs, rollback, and troubleshooting. The
[chart and environment notes](enterprise-gitops/README.md) describe the
templates and values.
