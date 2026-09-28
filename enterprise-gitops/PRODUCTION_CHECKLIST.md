# Production Checklist for fintech-enterprise-gitops-manifest

## Infrastructure Foundation (from eks-production-platform)
- [ ] EKS cluster deployed for dev, staging, and production environments
- [ ] VPC with proper CIDR ranges: 10.10.0.0/16 (dev), 10.20.0.0/16 (staging), 10.30.0.0/16 (prod)
- [ ] Node groups configured with appropriate sizing per environment
- [ ] KMS encryption enabled for Kubernetes secrets
- [ ] CloudWatch logging enabled for all control plane components
- [ ] NAT gateways configured for outbound traffic
- [ ] Security groups follow least-privilege rules
- [ ] Terraform state backed up and locked per environment

## Argo CD Installation & Configuration
- [ ] Argo CD installed on all clusters (dev, staging, production)
- [ ] Argo CD admin credentials securely stored (not in Git)
- [ ] Repository registered with GitHub PAT (not hardcoded)
- [ ] TLS certificates configured for Argo CD server
- [ ] RBAC roles created per environment (dev/staging/prod tiers)
- [ ] Argo CD backed up daily
- [ ] Argo CD service account has limited cluster permissions

## Application Manifests & Helm Charts
- [x] Chart template fixed: `.Values.image.tag` instead of `.Values.image.digest`
- [x] Liveness and readiness probes added to all deployments
- [x] Resource requests and limits defined for all services
- [x] HPA configured with CPU and memory thresholds
- [x] PodDisruptionBudget configured for high availability
- [x] NetworkPolicy configured for security isolation
- [x] Pod security context enforced (runAsNonRoot, fsGroup)
- [x] Prometheus annotations for monitoring
- [x] SecurityContext properly set

## Environment-Specific Configuration
- [x] Dev values populated for all 7 services
- [x] Staging values populated for all 7 services
- [x] Production values populated for all 7 services with higher resources
- [x] Pod anti-affinity configured in production
- [x] Environment variables correctly set (SPRING_PROFILES_ACTIVE)
- [x] Staging resources calibrated at ~60% of production
- [x] Dev resources minimal but suitable for testing

## Namespaces & RBAC
- [ ] All 21 namespaces created (dev, staging, prod × 7 services)
- [ ] Namespace labels applied correctly
- [ ] RBAC roles configured per environment tier
- [ ] ServiceAccounts created in each namespace
- [ ] Argo CD service account bound to appropriate roles
- [ ] CODEOWNERS file configured for environment-specific approvals

## GitOps Project & ApplicationSet
- [ ] Argo CD project `fintech-microservices` created
- [ ] ApplicationSet generator deployed and creating apps
- [ ] All 21 applications auto-discovered and syncing
- [ ] Sync policies automated for dev/staging, manual for production
- [ ] Namespace creation enabled via CreateNamespace policy

## Secrets Management
- [ ] AWS Secrets Manager integration planned/configured
- [ ] External Secrets Operator installed (or Sealed Secrets alternative)
- [ ] SecretStore created for each environment
- [ ] Database credentials stored in AWS Secrets Manager (not Git)
- [ ] API keys stored securely
- [ ] No secrets committed to repository
- [ ] Secret rotation policy documented

## CI/CD & Promotion Workflow
- [ ] `.github/workflows/promote-dev.yml` created (auto on build success)
- [ ] `.github/workflows/promote-staging.yml` created (manual dispatch)
- [ ] `.github/workflows/promote-production.yml` created (manual + approval)
- [ ] Helm lint validation in CI pipeline
- [ ] Argo CD dry-run validation in CI pipeline
- [ ] Image scanning (Trivy/Snyk) in CI pipeline
- [ ] Cost estimation for production changes

## Branch Protection & Governance
- [ ] Branch protection enabled on `main` branch
- [ ] Require PRs before merging enabled
- [ ] Minimum 1 approval required
- [ ] CODEOWNERS auto-request reviews enabled
- [ ] Dismiss stale PR approvals enabled
- [ ] Require up-to-date branches enabled
- [ ] Signed commits required for production changes

## Observability & Monitoring
- [ ] Prometheus scrape endpoints configured (`/metrics`)
- [ ] ServiceMonitor manifests added for each service
- [ ] Structured logging enabled (JSON format)
- [ ] Log level configurable via environment variable
- [ ] CloudWatch Container Insights enabled
- [ ] Dashboards created for key services
- [ ] Alerts configured for critical thresholds
- [ ] Tracing (OpenTelemetry) integration planned

## Disaster Recovery & Backup
- [ ] Terraform state backed up automatically
- [ ] Argo CD backed up daily (etcd snapshot)
- [ ] GitOps repo protected as source of truth
- [ ] Runbook for cluster recovery documented
- [ ] Runbook for Argo CD recovery documented
- [ ] Disaster recovery tested (RTO/RPO validation)

## Documentation
- [x] ARGOCD_INTEGRATION.md created with full setup guide
- [x] CODEOWNERS file created
- [ ] Architecture diagram added (with integration)
- [ ] Runbook for common operations documented
- [ ] Security considerations documented
- [ ] Troubleshooting guide completed
- [ ] Team trained on GitOps workflow

## Testing & Validation
- [ ] Dev environment: all 7 services deployed and healthy
- [ ] Staging environment: smoke tests passed
- [ ] Production dry-run: Helm renders correctly
- [ ] Cross-environment promotion: dev → staging → prod pipeline works
- [ ] Rollback procedure tested
- [ ] HPA scaling tested under load
- [ ] NetworkPolicy rules validated
- [ ] Pod anti-affinity spreading verified

## Security & Compliance (Fintech-Specific)
- [ ] Least-privilege RBAC enforced
- [ ] Pod security standards enforced
- [ ] Network policies restrict lateral movement
- [ ] Audit logging enabled for all changes
- [ ] Image scanning for vulnerabilities required
- [ ] Secrets encryption at rest and in transit
- [ ] Compliance audit trail maintained in Git commits
- [ ] Security scanning integrated into promotion pipeline

## Final Sign-Off
- [ ] Dev environment approved for testing
- [ ] Staging environment approved for pre-production validation
- [ ] Production environment approved for live traffic (SRE/Security/Finance teams)
- [ ] Team training completed
- [ ] On-call procedures documented
- [ ] Incident response plan documented

---

## Notes

### Critical Path to Production
1. Infrastructure: Deploy EKS cluster (eks-production-platform)
2. GitOps: Install Argo CD
3. Manifests: Deploy ApplicationSet + Project
4. Validation: Verify all 21 apps syncing
5. Secrets: Integrate AWS Secrets Manager
6. Promotion: Test dev→staging→prod workflow
7. Governance: Enable branch protection
8. Monitoring: Verify Prometheus + logs flowing
9. Sign-off: Get security + ops approval

### High-Risk Items
- Production deployments must require PR review + approval gate
- Database migrations must be tested in staging before prod
- Image tag promotion must be audited and approved
- Rollback procedure must be tested monthly

### Ongoing Maintenance
- Weekly: Check for security updates (Kubernetes, add-ons)
- Monthly: Test disaster recovery procedures
- Quarterly: Review and audit IAM permissions
- Continuously: Monitor metrics and alert on anomalies

