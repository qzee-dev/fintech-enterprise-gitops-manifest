Step 6 — Test Helm locally BEFORE Argo CD

This is important.

Don't immediately give this to Argo CD.

Go to:

cd chart/microservice

Run:

helm lint .

You want:

1 chart(s) linted, 0 chart(s) failed

Then render payment-service:

helm template payment-service . \
  -f ../../environments/dev/payment-service/values.yaml

This should produce Kubernetes YAML containing:

kind: Deployment

and the correct image:

image: ".../payment-service:1.0.0"

Also verify:

helm template payment-service . \
  -f ../../environments/dev/payment-service/values.yaml \
  | grep "image:"

You should see your payment-service ECR image.


7. DEV promotion workflow

Your CI can update DEV automatically after successful CI.

Conceptually:

name: Update DEV image

on:
  workflow_run:
    workflows: ["CI"]
    types:
      - completed

jobs:
  update-dev:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}

    runs-on: ubuntu-latest

    steps:
      - name: Checkout GitOps repository
        uses: actions/checkout@v4

      - name: Update image tag
        run: |
          sed -i \
            's/tag: .*/tag: "${IMAGE_TAG}"/' \
            environments/dev/payment-service/values.yaml

      - name: Commit DEV promotion
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

          git add environments/dev/payment-service/values.yaml

          git commit -m "Promote payment-service ${IMAGE_TAG} to dev"

          git push

But don't copy this blindly yet because your existing CI may build all seven services and use a specific tag variable. We should connect the promotion workflow to your actual CI output.

8. STAGING should be a promotion, not another build

Create:

.github/workflows/promote-staging.yml

The basic logic should be:

Input:
    service
    image tag

        ↓

Validate image exists in ECR

        ↓

Update:
environments/staging/<service>/values.yaml

        ↓

Create PR

        ↓

Review

        ↓

Merge

        ↓

Argo CD deploys STAGING

For example:

Promote:

payment-service
25-a82c911

DEV → STAGING

The PR changes:

 image:
   repository: ".../payment-service"
-  tag: "24-91bc123"
+  tag: "25-a82c911"

That's an excellent GitOps audit trail.

9. STAGING testing

After the PR is merged:

Git
 ↓
Argo CD
 ↓
STAGING

Check:

kubectl get applications -n argocd

Then:

kubectl get pods -n staging-payment-service

And your automated staging tests should run.

For example:

STAGING
   ↓
Smoke tests
   ↓
API tests
   ↓
Integration tests
   ↓
Security checks

Only if these pass should production promotion be allowed.

10. Production must be a PR

This is where we make the architecture production-grade.

Do not allow:

CI
 ↓
direct production modification

Instead:

STAGING PASSED
       ↓
Create Production PR
       ↓
GitHub required review
       ↓
Approval
       ↓
Merge
       ↓
Argo CD
       ↓
Production

Example PR:

Promote payment-service 25-a82c911 to production

Change:

 image:
   repository: ".../payment-service"
-  tag: "24-91bc123"
+  tag: "25-a82c911"
11. Protect the production branch

Your GitOps repository should have a protected branch, typically:

main

Require:

Pull request before changes
At least one approval
Required status checks
No force pushes
No direct developer pushes

For stronger production control, you can also use a separate production branch, but I would not introduce that complexity yet.

A clean setup is:

main
 ├── dev
 ├── staging
 └── production

represented by directories.
