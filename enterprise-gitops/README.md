
DEPLOYMENT SEQUENCE

# 1. Update Argo CD ApplicationSet (corrected paths)
kubectl apply -f enterprise-gitops/argocd/project.yaml
kubectl apply -f enterprise-gitops/argocd/applicationset.yaml

# 2. Create namespaces (the ApplicationSet also uses CreateNamespace=true)
kubectl apply -f enterprise-gitops/namespace/all-namespaces.yaml

# 3. Apply RBAC
kubectl apply -f enterprise-gitops/rbac/argocd-rbac.yaml

# 4. Verify applications auto-discovered
kubectl get applications -n argocd

# 5. Check Helm rendering
cd enterprise-gitops/chart/microservice
helm lint .
helm template dev-payment-service . \
  --namespace dev-payment-service \
  -f ../../environments/dev/payment-service/values.yaml

# 6. Monitor deployments
kubectl get pods -n dev-payment-service
kubectl get pods -n staging-payment-service
kubectl get pods -n production-payment-service

KEDA is installed as a cluster add-on, separately from this application chart.
Install KEDA and its CRDs in each destination cluster before syncing any
application with `keda.enabled: true`. The chart renders Kafka `ScaledObject`
resources for transaction, fraud, notification, and DLQ consumers; non-consumer
services retain CPU HPA. The chart does not install or configure Kafka.

Each environment values file specifies the broker address, topics, consumer
groups, and scaling bounds. The committed `kafka:29092` address matches the
fintech-platform Compose network. Replace it with a routable cluster broker
endpoint if EKS does not expose Kafka under that DNS name and port. Configure
trigger authentication separately if the broker requires credentials. Apply
the platform Terraform stack and verify
`kubectl get crd scaledobjects.keda.sh` before applying or syncing the
ApplicationSet. Consumer workloads also require their environment's database
and other credentials through externally managed Kubernetes Secrets; this
chart does not create those Secrets. Replace the sample AWS account ID
`123456789012` in image repositories with the account that owns the ECR
repositories before syncing workloads.
