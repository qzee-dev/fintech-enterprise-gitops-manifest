# Enterprise manifest repository runbook

## What this repository owns

`qzee-dev/fintech-enterprise-gitops-manifest` is the configuration source of
truth for the shared Helm chart and environment-specific application values. It
does not build images, create ECR repositories, or provision EKS clusters.

| Path | Responsibility |
| --- | --- |
| `enterprise-gitops/chart/microservice/` | Shared Helm templates for Deployments, Services, probes, resource limits, HPA/KEDA, security controls, and service accounts. |
| `enterprise-gitops/environments/<env>/<service>/values.yaml` | Image identity and runtime/scaling configuration for a service in an environment. |
| `enterprise-gitops/argocd/project.yaml` | Argo CD repository/destination boundaries for the project. |
| `enterprise-gitops/argocd/applicationset.yaml` | Generates applications for staging and production only. |
| `enterprise-gitops/namespace/`, `enterprise-gitops/rbac/` | Optional namespace and Argo CD RBAC bootstrap resources. |

### System behavior and repository handoffs

1. `qzee-dev/fintech-platform` builds and scans one service image, publishes it
   to `us-east-1` ECR under `fintech-platform/<service>`, and stores a release
   artifact containing release ID, tag, digest, commit, and run ID.
2. **Dev is not deployed from this repo.** The source repo's manually
   dispatched dev workflow uses `kubectl` to set the existing
   `deployment/<service>` in namespace `dev` to the immutable image digest.
   `environments/dev/` values are reference configuration. The ApplicationSet
   does not scan them.
3. For staging, the source repo's manual promotion workflow verifies the
   release artifact and ECR digest, writes the repository/tag/digest into
   `environments/staging/<service>/values.yaml`, and commits to `main`.
4. For production, the source repo's manual promotion workflow validates the
   same release identity, requires any configured `production` Environment
   approval, updates `environments/production/<service>/values.yaml`, and
   commits to `main`.
5. The ApplicationSet sees the new Git revision and creates or updates
   `staging-<service>` or `production-<service>`. Argo CD renders the shared
   Helm chart, syncs Kubernetes, and self-heals drift.

The promotion workflows push directly to `main`; they do not open pull
requests. GitHub Environment protection is the production approval boundary.
Argo CD's current destination is its local cluster. Thus staging and production
are separate namespaces/configurations, not separate clusters, unless cluster
destinations are explicitly registered and configured.

## Tagging and promotion policy

The source repo assigns each release an immutable ECR tag:
`<service>-<semantic-version>-<7-char-source-commit>` (for example
`payment-service-v1.2.3-a1b2c3d`). The release metadata and this repository's
release-history records retain the release run ID, source commit, image tag,
and digest. Git commits in this repository identify configuration changes;
they are not image release tags.

Treat the digest as the deployed artifact identity. The values files record
both `image.tag` and `image.digest`; when a digest is present, the shared chart
renders the image as `repository@digest`. Promotion updates the repository,
tag, and digest together. Use the same image tag and digest for dev, staging,
and production; do not rebuild, retag, or use `latest` between environments.
For rollback, promote a prior recorded digest. ECR currently retains only the
latest 30 tagged images per service repository, so older releases may no longer
be available.

AWS resource tags such as `Owner`, `ManagedBy`, `Environment`, `CostCenter`,
and `Project` are distinct from image tags. They are applied by the
infrastructure Terraform stacks. Environment values in this repo configure
workloads; they do not by themselves guarantee propagation of AWS resource
tags to Kubernetes objects or cloud resources.

## First-time setup

1. **Provision prerequisites elsewhere.** Use
   [`gitops-repo`'s infrastructure runbook](https://github.com/qzee-dev/gitops-repo/blob/main/RUNBOOK.md)
   to provision ECR and install the Argo CD/KEDA platform add-ons on an existing
   EKS cluster. The source repo's
   [deployment runbook](https://github.com/qzee-dev/fintech-platform/blob/main/docs/DEPLOYMENT.md)
   documents GitHub Environments, IAM roles, GitHub App access, and required
   variables/secrets.
2. **Set the actual ECR account.** Replace `AWS_ACCOUNT_ID_REQUIRED` in all
   staging and production `image.repository` values with the 12-digit account
   that owns `fintech-platform/*` in `us-east-1`. The account ID was not
   supplied when these files were prepared. Do not invent it or commit secrets.
   The staging/production workflows rewrite the image repository for the
   service they promote using the `AWS_ACCOUNT_ID` secret.
3. **Ensure bootstrap images exist.** Before first sync, each staging and
   production values file must point to an image tag that exists in ECR. Values
   currently use sample `1.0.0` tags and an empty digest; these are not
   guaranteed to exist. Either seed the values with real approved image
   identities or allow the promotion workflow to write a valid release first,
   then create/sync the application.
4. **Ensure runtime dependencies exist.** Supply database, Kafka, JWT, external
   provider, and other credentials using externally managed Kubernetes
   Secrets. The chart does not create credentials or backing services. Install
   KEDA and its CRDs before syncing any service with KEDA enabled.
5. **Bootstrap Argo CD project and ApplicationSet.** With a kubectl context for
   the cluster that hosts Argo CD, run from the repository root:

   ```bash
   kubectl apply -f enterprise-gitops/argocd/project.yaml
   kubectl apply -f enterprise-gitops/argocd/applicationset.yaml
   ```

   The ApplicationSet uses the `main` branch and this repository's HTTPS URL.
   It scans only `staging/*` and `production/*` and creates namespaces with
   `CreateNamespace=true`.
6. **Check the first render before syncing production.**

   ```bash
   cd enterprise-gitops/chart/microservice
   helm lint .
   helm template staging-payment-service . \
     --namespace staging-payment-service \
     -f ../../environments/staging/payment-service/values.yaml
   ```

   Verify the rendered image, port, probes, resources, secrets references, and
   scaling objects against the target service and cluster.

## Normal operating procedure

### Scaling policy

Use the workload's actual scaling signal:

| Workload | Services | Scaling policy |
| --- | --- | --- |
| HTTP-only request path | `api-gateway`, `payment-service`, `user-service`, `wallet-service`, `ledger-service`, `web` | CPU HPA; target 70% of requested CPU. |
| Kafka consumers | `transaction-service`, `fraud-service`, `notification-service`, `dlq-processor` | KEDA Kafka lag trigger. Consumer values also configure a 70% CPU trigger, so mixed HTTP/consumer pods can scale on either signal. |
| Test/mock | `mock-service` | Fixed replicas by default; enable autoscaling only if it serves real, variable production traffic. |

`transaction-service`, `fraud-service`, and `notification-service` run both
HTTP endpoints and Kafka consumers in the same process/deployment; transaction
also publishes events. KEDA scales those shared pods from Kafka lag and CPU,
but cannot independently scale the HTTP and consumer roles. Split each into
separate API and worker deployments if those workloads need independent
capacity, resource sizing, or release cadence.

The configured minimum/maximum replicas, `lagThreshold`, CPU requests, and
70%-CPU targets are starting guardrails, not measured production capacity.
Validate them with representative load tests and production metrics. For Kafka
consumers, keep `maxReplicaCount` at or below useful consumer parallelism
(partition capacity across subscribed topics), and also below database and
downstream-service capacity. Confirm the HPA/KEDA targets have CPU requests,
the cluster metrics API is working, and KEDA can authenticate to the actual
Kafka endpoint before enabling these settings in production.

`dlq-processor` currently consumes DLQ messages but only logs them; its KEDA
configuration does not make it a functioning retry/recovery worker. Implement
and test idempotent DLQ handling before treating its scaling behavior as
operationally useful.

### Reviewing a manifest change

1. Identify the environment and service. Only staging/production folders are
   auto-deployed by Argo CD.
2. For application image promotion, use the source repository's workflow. Do
   not rebuild or move an immutable tag, and do not manually change just the
   tag while leaving a stale digest.
3. For chart/configuration changes, make a reviewed Git change, lint the chart,
   render affected values, and inspect the diff. Follow repository branch
   protection policy; ensure staging validation before production changes.
4. Confirm no secrets are present in values, templates, or commits.

### Verify an Argo CD deployment

For service `<service>`:

```bash
argocd app get staging-<service> --server "$ARGOCD_SERVER"
kubectl get pods -n staging-<service>
kubectl get events -n staging-<service> --sort-by=.lastTimestamp
```

Use `production-<service>` and `production-<service>` namespace for production.
Confirm the app is `Synced` and `Healthy`, the Git revision matches the
promotion commit, all replicas are ready, and the pod image is the approved
digest. The source promotion workflow also waits for Argo CD and records release
history in this repository.

### Rollback

Use the source repo's promotion workflow with a prior successful release
artifact to restore a previously verified tag/digest. Production rollback
requires the configured production approval. Since automated sync/self-heal are
enabled, restore the desired state in Git rather than editing live objects with
kubectl.

## Directory and application contract

The currently supported services are:

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

For every service, the repo maintains values under dev, staging, and
production. Only staging and production are Argo-managed. App names and
namespaces are `<environment>-<service>`. ECR repository names are
`fintech-platform/<service>`.

## Operational cautions

- The chart's `NetworkPolicy` defaults, container ports, health probes, and
  service dependencies must be checked against the actual cluster networking
  and runtime setup.
- The committed `kafka:29092` values match the local Compose network, not
  necessarily an EKS broker. Set a routable cluster broker endpoint and
  credentials before enabling Kafka consumers.
- The manifests currently use a single Argo CD local-cluster destination.
  Directory names alone do not provide cluster, account, or network isolation.
- If an ApplicationSet directory is removed, Argo CD may prune its generated
  application/resources under automated prune. Review removal plans carefully.
