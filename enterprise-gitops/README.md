
DEPLOYMENT SEQUENCE

# 1. Update Argo CD ApplicationSet (corrected paths)
kubectl apply -f enterprise-gitops/argocd/project.yaml
kubectl apply -f enterprise-gitops/argocd/applicationset.yaml

# 2. Create namespaces
kubectl apply -f enterprise-gitops/namespaces/all-namespaces.yaml

# 3. Apply RBAC
kubectl apply -f enterprise-gitops/rbac/argocd-rbac.yaml

# 4. Verify applications auto-discovered
kubectl get applications -n argocd

# 5. Check Helm rendering
cd enterprise-gitops/chart/microservice
helm lint .
helm template payment-service . \
  -f ../../environments/dev/payment-service/values.yaml

# 6. Monitor deployments
kubectl get pods -n dev-payment-service
kubectl get pods -n staging-payment-service
kubectl get pods -n production-payment-service
