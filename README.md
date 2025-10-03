# ArgoCD GitOps Configuration

This directory contains the ArgoCD configuration and Helm charts for deploying applications to the EKS cluster.

## Directory Structure

```
argo/
├── apps/                           # ArgoCD Application manifests
│   └── fastapi-app.yaml           # FastAPI application ArgoCD config
├── fastapi-app/                   # Helm chart for FastAPI application
│   ├── Chart.yaml                 # Helm chart metadata
│   ├── values.yaml                # Default values
│   ├── values-dev.yaml            # Development environment values
│   ├── values-prod.yaml           # Production environment values
│   └── templates/                 # Kubernetes manifests templates
│       ├── _helpers.tpl           # Template helpers
│       ├── deployment.yaml        # Deployment configuration
│       ├── service.yaml           # Service configuration
│       ├── serviceaccount.yaml    # Service account
│       └── ingress.yaml           # ALB Ingress configuration
└── README.md                      # This file
```

## Prerequisites

1. **EKS Cluster**: Running with AWS Load Balancer Controller installed
2. **ArgoCD**: Installed and running in the cluster
3. **ECR Repository**: FastAPI image pushed to ECR Public
4. **GitHub Repository**: This code pushed to a Git repository

## Quick Start

### 1. Update Configuration

Before deploying, update the following values in `apps/fastapi-app.yaml`:

```yaml
source:
  repoURL: https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPO_NAME.git  # Your Git repo URL
  helm:
    values: |
      image:
        registryAlias: YOUR_ECR_REGISTRY_ALIAS  # Your ECR registry alias
```

### 2. Deploy the Application

Apply the ArgoCD Application manifest:

```bash
kubectl apply -f argo/apps/fastapi-app.yaml
```

### 3. Access ArgoCD UI

Get the ArgoCD admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Port-forward to access ArgoCD UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open https://localhost:8080 in your browser:
- Username: `admin`
- Password: `aT2wQ2-UROcr1k-I`

### 4. Get Application Load Balancer URL

Once the application is synced, get the ALB URL:

```bash
kubectl get ingress -n default
```

Look for the `ADDRESS` column to find your ALB endpoint.

## ArgoCD Application Features

The ArgoCD Application is configured with:

- **Automated Sync**: Automatically deploys changes from Git
- **Self-Healing**: Automatically corrects drift from desired state
- **Pruning**: Removes resources that are no longer defined in Git
- **Retry Logic**: Automatically retries failed syncs with exponential backoff
- **Revision History**: Keeps last 3 revisions for rollback

## Helm Chart Configuration

### Default Values (values.yaml)

- 2 replicas
- 500m CPU / 512Mi memory limits
- ALB Ingress with health checks
- Service type: ClusterIP
- Auto-scaling disabled by default

### Development (values-dev.yaml)

```bash
helm install fastapi-app ./fastapi-app -f ./fastapi-app/values-dev.yaml
```

- 1 replica
- Lower resource limits
- Latest image tag

### Production (values-prod.yaml)

```bash
helm install fastapi-app ./fastapi-app -f ./fastapi-app/values-prod.yaml
```

- 3 replicas
- Higher resource limits
- Horizontal Pod Autoscaler enabled (3-10 pods)
- Specific image tags (not latest)
- SSL redirect enabled

## Manual Helm Deployment (Alternative to ArgoCD)

If you want to deploy directly with Helm:

```bash
# Install
helm install fastapi-app ./argo/fastapi-app \
  --set image.registryAlias=YOUR_REGISTRY_ALIAS \
  --namespace default

# Upgrade
helm upgrade fastapi-app ./argo/fastapi-app \
  --set image.registryAlias=YOUR_REGISTRY_ALIAS \
  --set image.tag=v1.0.1 \
  --namespace default

# Uninstall
helm uninstall fastapi-app --namespace default
```

## Monitoring ArgoCD Application

### Via CLI

```bash
# Get application status
kubectl get application -n argocd

# Describe application
kubectl describe application fastapi-app -n argocd

# Watch sync status
kubectl get application fastapi-app -n argocd -w
```

### Via ArgoCD CLI

```bash
# Login to ArgoCD
argocd login localhost:8080

# Get application details
argocd app get fastapi-app

# Sync application
argocd app sync fastapi-app

# Get sync history
argocd app history fastapi-app

# Rollback to previous version
argocd app rollback fastapi-app
```

## Troubleshooting

### Application not syncing

Check ArgoCD application status:
```bash
kubectl describe application fastapi-app -n argocd
```

### Pods not starting

Check pod logs:
```bash
kubectl logs -l app.kubernetes.io/name=fastapi-app -n default
```

### Ingress not creating ALB

Check AWS Load Balancer Controller logs:
```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

Verify IngressClass exists:
```bash
kubectl get ingressclass
```

### Image pull errors

Ensure ECR repository is public or service account has ECR pull permissions.

## Updating the Application

### Update Image Tag

1. Modify `apps/fastapi-app.yaml`:
   ```yaml
   image:
     tag: "v1.0.1"  # New version
   ```

2. Commit and push to Git
3. ArgoCD will automatically sync (or manually sync via UI/CLI)

### Update Configuration

1. Modify values in `apps/fastapi-app.yaml` under `spec.source.helm.values`
2. Commit and push
3. ArgoCD will automatically apply changes

## Best Practices

1. **Use Specific Tags**: In production, use specific image tags instead of `latest`
2. **Resource Limits**: Always set resource requests and limits
3. **Health Checks**: Configure liveness and readiness probes
4. **Auto-scaling**: Enable HPA for production workloads
5. **SSL/TLS**: Use ACM certificates for HTTPS in production
6. **Monitoring**: Integrate with Prometheus/Grafana for observability
7. **Secrets**: Use AWS Secrets Manager or External Secrets Operator for sensitive data

## Clean Up

Remove the application:

```bash
# Via ArgoCD (recommended)
kubectl delete -f argo/apps/fastapi-app.yaml

# Via Helm (if deployed manually)
helm uninstall fastapi-app --namespace default
```
