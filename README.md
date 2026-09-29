# fintech-enterprise-gitops-manifest

Enterprise GitOps repository for managing Kubernetes deployments across **Dev**, **Staging**, and **Production** environments using **ArgoCD**, **ApplicationSets**, and **Helm**.

---

## Repository Structure

```text
enterprise-gitops/
├── argocd/
│   ├── applicationset.yaml
│   └── project.yaml
│
├── chart/
│   └── microservice/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── hpa.yaml
│           ├── service.yaml
│           └── serviceaccount.yaml
│
├── environments/
│   ├── dev/
│   │   ├── payment-service/
│   │   │   └── values.yaml
│   │   ├── fraud-service/
│   │   │   └── values.yaml
│   │   └── ...
│   │
│   ├── staging/
│   │   └── ...
│   │
│   └── production/
│       └── ...
│
└── README.md
```

---

## Directory Overview

### `chart/microservice/`

Shared Helm chart used by all microservices.

#### Templates

| File | Purpose |
|--------|---------|
| `deployment.yaml` | Kubernetes Deployment definition |
| `hpa.yaml` | Horizontal Pod Autoscaler |
| `service.yaml` | ClusterIP Service |
| `serviceaccount.yaml` | Service Account and identity |
| `_helpers.tpl` | Reusable Helm template functions |

#### Chart Files

| File | Purpose |
|--------|---------|
| `Chart.yaml` | Helm chart metadata |
| `values.yaml` | Default chart values |

---

### `argocd/`

Contains ArgoCD configuration.

| File | Purpose |
|--------|---------|
| `applicationset.yaml` | Automatically generates ArgoCD Applications from environment folders |
| `project.yaml` | ArgoCD Project with RBAC and deployment boundaries |

---

### `environments/`

Contains environment-specific configuration.

```text
environments/
├── dev/
│   ├── payment-service/
│   │   └── values.yaml
│   ├── fraud-service/
│   │   └── values.yaml
│   └── ...
│
├── staging/
│   └── ...
│
└── production/
    └── ...
```

Example:

```yaml
# environments/dev/payment-service/values.yaml

replicaCount: 2

resources:
  requests:
    cpu: 100m
  limits:
    cpu: 500m

springProfile: dev
```

---

## How It Works

### 1. ApplicationSet Discovers Services

`applicationset.yaml` scans the repository and automatically creates ArgoCD Applications.

```text
environments/
├── dev/payment-service
├── dev/fraud-service
├── staging/payment-service
└── production/payment-service
```

Each discovered folder becomes its own ArgoCD Application.

---

### 2. ApplicationSet Passes Environment Values

For each service, ApplicationSet tells Helm which values file to use.

Example:

```text
dev/payment-service/values.yaml
```

or

```text
production/payment-service/values.yaml
```

---

### 3. Helm Renders the Templates

Helm combines:

```text
Chart.yaml
+
values.yaml
+
Environment-specific values.yaml
+
templates/*.yaml
```

and*produces the final Kubernetes mani*ests.

---

### 4. ArgoCD Deploys *o Kubernetes

After Helm renders*the manifests:

```text*ApplicationSet
        ↓
Argo*D Application
        ↓
Helm*Chart
       *↓
Rendered Man*fests
        ↓
Kubernetes*Cluster
*``

---

## Component Responsibili*ies

### ApplicationSet

Responsib*e for:

- Discovering environments*and services
- Creating*ArgoCD Applications
* Selecting the correct values file*- Passing values into Helm

###*Chart.yaml

Responsible for:

-*Helm*chart metadata
- Chart name
- Vers*oning
- Dependencies

###*values.yaml

Responsible for:

- D*fault*configuration values
- Shared*settings*across all environments

### Envir*nment Values

Responsible for:

* Environment-specific overrides
- *eplica counts
- Resource limits
- *nvironment variables
- Spring prof*les

### templates/

Responsible f*r:

- Deployment manifests
- Servi*e manifests
- HPA manifests
- Serv*ceAccount*manifests

---

*# Deployment Flow

```text*en*ironments/dev/payment-service/valu*s.yaml
                    │
     *              ▼
            Applic*tionSet
                    │
    *               ▼
          Ar*oCD Application
                  * │
                    ▼
         *    Helm Chart
                   *│
                    ├*─ Chart.yaml
                    ├*─ values.yaml
                    *── templates/*.yaml
              *     │
                    ▼
     *Rendered Kubernetes Manifests
    *               │
                 *  ▼
            Kubernetes Cluster*```

---

## Key Point

**Applicat***Set is not the Helm template its***.**

ApplicationSet's job is to:*
1. Discover services and environm*nts.
2. Generate ArgoCD Applicatio*s.
3. Tell Helm which values file *o use.

Helm's job is to:

1.*Read `Chart.yaml`.
2. Read `values*yaml`.
3. Read the environment-spe*ific values file.
4. Render*`templates/*.yaml`.

The rendered*Kubernetes*manifests are then synchronized an* deployed by ArgoCD.
````*
