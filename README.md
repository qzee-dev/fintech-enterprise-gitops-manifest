# fintech-enterprise-gitops-manifest

enterprise-gitops/
  chart/microservice/           Shared Helm chart (reusable template for all services)
    template/
      deployment.yaml           Pod spec with resource limits, auto-scaling
      hpa.yaml                  Horizontal Pod Autoscaler (CPU-driven, 2-5 replicas)
      service.yaml              ClusterIP service
      serviceaccount.yaml       RBAC identity
    values.yaml                 Default/skeleton values
    Chart.yaml                  Helm metadata (v1.0.0)
  
  argocd/
    applicationset.yaml         Generator: discovers env/*/service/* dirs, creates apps automatically
    project.yaml                RBAC project: allows source repo, destined to dev-*, staging-*, prod-*
  
  environments/
    dev/
      payment-service/values.yaml       Concrete values: 2 replicas, CPU 100m-500m, Spring profile dev
      fraud-service/values.yaml         ... (same pattern)
      ... (7 services total)
    staging/                    (empty structure, ready for values)
    production/                 (empty structure, ready for values)
