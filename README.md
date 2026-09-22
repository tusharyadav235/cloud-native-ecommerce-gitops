# Cloud-Native E-Commerce Platform - GitOps Repository

This is the central GitOps configuration repository for the **Cloud-Native E-Commerce Platform**. It contains all Kubernetes manifests, configurations, and deployment definitions to securely and consistently deploy the platform across different environments (Dev and Prod).

By isolating the Kubernetes configuration from the application source code (the App Repo), we establish a clear separation of concerns, prevent CI/CD loop issues, and allow for a clean, auditable history of infrastructure changes.

## Architecture & Tooling

This repository leverages the following Cloud-Native stack:
- **Kubernetes (K8s)**: Container orchestration.
- **Kustomize**: Template-free, declarative configuration management (using `base` and `overlays`).
- **ArgoCD**: Continuous Delivery (CD) engine that monitors this repository and syncs the cluster state.
- **External Secrets Operator**: Syncs database credentials directly from AWS Secrets Manager into native Kubernetes Secrets.
- **AWS RDS (MySQL)**: External managed database for persistence.

## Repository Structure

The configuration follows standard Kustomize architecture:

```text
.
├── base/                   # Core deployment definitions for all microservices
│   ├── cart-deployment.yaml
│   ├── frontend-deployment.yaml
│   ├── order-deployment.yaml
│   ├── product-deployment.yaml
│   ├── ingress.yaml        # NGINX Ingress rules
│   ├── db-configmap.yaml   # Common database configuration
│   ├── secret-store.yaml   # External Secrets AWS integration
│   ├── external-secret.yaml# Secret sync definitions
│   └── kustomization.yaml
│
└── overlays/               # Environment-specific overrides
    ├── dev/                # Development environment (e.g. connects to Dev RDS)
    │   └── kustomization.yaml
    └── prod/               # Production environment (e.g. connects to Prod RDS)
        └── kustomization.yaml
```

## Prerequisites

Before deploying these manifests, ensure the target Kubernetes cluster has the following installed and configured:
1. **ArgoCD**: Running in the cluster.
2. **Ingress Controller**: (e.g., NGINX Ingress Controller) to satisfy the `Ingress` resource.
3. **External Secrets Operator (ESO)**: Installed and configured with appropriate IAM permissions (IRSA on AWS EKS) to read from AWS Secrets Manager.
4. **AWS Secrets Manager**: A secret named `ecommerce/db-credentials` containing a `password` property.
5. **AWS RDS**: MySQL databases provisioned (e.g., `product_db`, `order_db`, `cart_db`) and reachable from the cluster.

## Deployment Instructions

### 1. Fully Automated (GitOps via ArgoCD)
The recommended way to deploy this platform is by applying the ArgoCD Application definitions to your cluster.

1. Create a file named `argocd-app.yaml` in your cluster (refer to the `argocd-app.yaml` located in the App Repo).
2. Ensure the `repoURL` points to *this* GitOps repository.
3. Apply it:
   ```bash
   kubectl apply -f argocd-app.yaml
   ```
ArgoCD will automatically detect the environments (Dev/Prod) and sync all deployments, services, and Horizontal Pod Autoscalers (HPAs).

### 2. Manual Deployment (Kustomize)
If you want to apply these configurations manually (for testing or debugging):

Deploy the Dev environment:
```bash
kubectl apply -k overlays/dev
```

Deploy the Prod environment:
```bash
kubectl apply -k overlays/prod
```

## CI/CD Workflow

1. A developer pushes code to the **App Repo**.
2. Jenkins (or GitHub Actions) runs tests and builds new Docker images for the modified microservices.
3. The CI pipeline runs `kustomize edit set image` locally.
4. The CI pipeline commits and pushes the updated `kustomization.yaml` back to **this GitOps repo**.
5. **ArgoCD** detects the new commit in this repo and automatically syncs the cluster, rolling out the new Docker images with zero downtime.

