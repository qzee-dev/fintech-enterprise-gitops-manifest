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
