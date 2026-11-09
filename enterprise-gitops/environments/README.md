# Environment values

Each values file follows this layout:

```text
<environment>/<service>/values.yaml
```

The staging and production Argo CD `ApplicationSet` watches their directories
and creates `<environment>-<service>` applications. The platform workflows
promote an immutable release by updating the image repository, tag, and digest
in the matching values file, then verify the Argo CD application. The dev
workflow deploys manually with `kubectl` to the `dev` namespace; dev values
remain available as reference but are not watched by the ApplicationSet.

The supported service set is:

```text
api-gateway
dlq-processor
ledger-service
mock-service
user-service
payment-service
wallet-service
transaction-service
notification-service
fraud-service
web
```

## Initial setup

Staging and production `image.repository` values use the explicit marker
`AWS_ACCOUNT_ID_REQUIRED`; replace it with the 12-digit AWS account ID that
owns the repositories before the first Argo CD sync. Staging and production
promotion workflows set the correct repository for the promoted service from
the source repository's `AWS_ACCOUNT_ID` secret. The dev values are not
consumed by the direct EKS deployment workflow.

Set each service's immutable `image.tag` and verified `image.digest` through a
promotion workflow rather than manually rebuilding or retagging an image.
Containers that require external services also need environment-scoped
Kubernetes Secrets configured separately; these values files do not contain
credentials.

## Validation

From `enterprise-gitops/chart/microservice`, lint the chart and render each
environment values file before syncing:

```bash
helm lint .
helm template staging-payment-service . \
  --namespace staging-payment-service \
  -f ../../environments/staging/payment-service/values.yaml
```

Confirm the rendered image points to the expected `fintech-platform/<service>`
ECR repository and digest.
