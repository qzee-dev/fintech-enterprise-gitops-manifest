# Enterprise GitOps layout

This directory is the source for the shared Helm chart, Argo CD configuration,
and per-environment values. See the [repository runbook](../RUNBOOK.md) for the
operating procedure and cross-repository handoffs.

## Components

- `chart/microservice/`: shared Kubernetes templates, including Deployment,
  Service, probes, resource controls, HPA/KEDA, NetworkPolicy, and service
  account.
- `environments/<environment>/<service>/values.yaml`: image identity,
  environment variables, ports, resource sizing, and scaling configuration.
- `argocd/project.yaml`: source repository and destination policy.
- `argocd/applicationset.yaml`: generates Argo CD applications for staging and
  production. It intentionally does not scan `environments/dev/`.
- `namespace/` and `rbac/`: optional namespace and Argo CD RBAC resources.

## Environment behavior

The source repo's manual dev workflow deploys a release digest directly into
the existing EKS namespace `dev`. Dev values in this repository are reference
configuration only.

For staging and production, the source repo's promotion workflows update the
matching values file with the ECR repository, immutable image tag, and digest.
Argo CD detects the commit, renders the shared chart, and synchronizes the
application. Application names and namespaces are `staging-<service>` or
`production-<service>`. Automated sync and self-heal are enabled.

The ApplicationSet destination is the local cluster where Argo CD runs.
Staging and production are therefore separate namespaces/configurations, not
separate EKS clusters unless Argo CD cluster destinations are explicitly
configured.

## Local validation

From `chart/microservice/`:

```bash
helm lint .
helm template staging-payment-service . \
  --namespace staging-payment-service \
  -f ../../environments/staging/payment-service/values.yaml
```

Before first sync, set real staging/production ECR account IDs and ensure the
selected image tag exists. Configure service Secrets outside Git; values files
must not contain credentials. KEDA and its CRDs must exist on the destination
cluster before syncing KEDA-enabled consumer services.
