# FastAPI Application Deployment Guide

This guide provides step-by-step instructions to deploy your FastAPI application to EKS using ArgoCD.

## Prerequisites Checklist

- [x] EKS cluster is running
- [x] AWS Load Balancer Controller is installed
- [x] ArgoCD is installed and running
- [x] kubectl configured to access the cluster
- [ ] Docker image is available in ECR Public
- [ ] Git repository contains this code

## Step 1: Push Code to Git Repository

If you haven't already, push this code to a Git repository:

```bash
# Initialize git (if not already done)
git init

# Add all files
git add .

# Commit
git commit -m "Add ArgoCD configuration for FastAPI app"

# Add remote (replace with your repo URL)
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git

# Push to main branch
git push -u origin main
```

## Step 2: Get Your ECR Registry Information

Find your ECR Public registry alias:

```bash
# List your ECR Public repositories
aws ecr-public describe-repositories --region us-east-1

# The output will show your registry alias
# Example: public.ecr.aws/x1y2z3a4/fastapi-eks-app
#                              ^^^^^^^^ this is your alias
```

## Step 3: Update ArgoCD Application Configuration

Edit `argo/apps/fastapi-app.yaml` and update:

1. **Git Repository URL** (line 10):
   ```yaml
   repoURL: https://github.com/YOUR_USERNAME/YOUR_REPO.git
   ```

2. **ECR Registry Alias** (line 16):
   ```yaml
   registryAlias: YOUR_ECR_REGISTRY_ALIAS
   ```

Example:
```yaml
source:
  repoURL: https://github.com/johndoe/eks-fastapi.git
  targetRevision: main
  path: argo/fastapi-app
  helm:
    values: |
      image:
        registry: public.ecr.aws
        registryAlias: x1y2z3a4
        repository: fastapi-eks-app
        tag: latest
```

## Step 4: Commit and Push Changes

```bash
git add argo/apps/fastapi-app.yaml
git commit -m "Update ArgoCD config with repo and ECR details"
git push
```

## Step 5: Verify Cluster Access

```bash
# Check cluster connection
kubectl get nodes

# Verify ArgoCD is running
kubectl get pods -n argocd

# Verify AWS Load Balancer Controller
kubectl get deployment -n kube-system aws-load-balancer-controller

# Check IngressClass
kubectl get ingressclass
```

Expected output for IngressClass:
```
NAME   CONTROLLER            PARAMETERS   AGE
alb    ingress.k8s.aws/alb   <none>       1d
```

## Step 6: Deploy the Application

Apply the ArgoCD Application manifest:

```bash
kubectl apply -f argo/apps/fastapi-app.yaml
```

Expected output:
```
application.argoproj.io/fastapi-app created
```

## Step 7: Monitor the Deployment

### Check Application Status

```bash
# Get application status
kubectl get application -n argocd

# Detailed application info
kubectl describe application fastapi-app -n argocd

# Watch application sync
kubectl get application fastapi-app -n argocd -w
```

### Check Pods

```bash
# Watch pods being created
kubectl get pods -n default -w

# Check pod details
kubectl get pods -n default
kubectl describe pod <pod-name> -n default
kubectl logs <pod-name> -n default
```

### Check Service and Ingress

```bash
# Check service
kubectl get svc -n default

# Check ingress (ALB)
kubectl get ingress -n default

# Detailed ingress info
kubectl describe ingress -n default
```

## Step 8: Access ArgoCD UI

### Port Forward to ArgoCD

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Login to ArgoCD

1. Open browser: https://localhost:8080
2. Accept self-signed certificate warning
3. Login with credentials:
   - **Username**: `admin`
   - **Password**: `aT2wQ2-UROcr1k-I`

### View Application in UI

- Click on **fastapi-app** to see the application details
- View the resource tree showing all Kubernetes resources
- Check sync status and health status
- Review sync history and events

## Step 9: Get Application URL

Wait for the ALB to be provisioned (2-3 minutes):

```bash
# Get ingress details
kubectl get ingress -n default

# Wait for ADDRESS to be populated
kubectl get ingress -n default -w
```

The output will show:
```
NAME                CLASS   HOSTS   ADDRESS                                    PORTS   AGE
fastapi-app-xxxxx   alb     *       k8s-default-fastapi-xxxxx.us-west-2.elb.amazonaws.com   80      5m
```

Copy the ADDRESS URL and open it in your browser.

## Step 10: Test the Application

Once the ALB is ready, test the endpoints:

```bash
# Get the ALB URL
ALB_URL=$(kubectl get ingress -n default -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')

# Test root endpoint
curl http://$ALB_URL/

# Test health endpoint
curl http://$ALB_URL/health

# Test info endpoint
curl http://$ALB_URL/api/info

# Open Swagger UI in browser
open http://$ALB_URL/docs
```

Expected responses:

**Root** (`/`):
```json
{"message": "Welcome to FastAPI on EKS"}
```

**Health** (`/health`):
```json
{"status": "healthy", "message": "Application is running"}
```

**Info** (`/api/info`):
```json
{
  "app": "FastAPI EKS Application",
  "version": "1.0.0",
  "environment": "production"
}
```

## Troubleshooting

### Application Status is "OutOfSync"

```bash
# Manually sync
kubectl patch application fastapi-app -n argocd --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'

# Or use ArgoCD UI to click "SYNC" button
```

### Pods in ImagePullBackOff

Check if ECR image exists and is accessible:

```bash
# Verify image exists
aws ecr-public describe-images \
  --repository-name fastapi-eks-app \
  --region us-east-1

# Check pod events
kubectl describe pod <pod-name> -n default
```

If image is private, you need to add ECR pull permissions to the service account.

### Ingress Not Creating ALB

Check AWS Load Balancer Controller:

```bash
# Check controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Verify controller is running
kubectl get deployment -n kube-system aws-load-balancer-controller

# Check ingress events
kubectl describe ingress -n default
```

Common issues:
- Subnet tags missing (`kubernetes.io/role/elb` for public subnets)
- IAM role not properly configured
- Security groups blocking traffic

### Pods Not Healthy

Check liveness/readiness probes:

```bash
# Check pod status
kubectl get pods -n default

# View pod logs
kubectl logs <pod-name> -n default

# Check probe configuration
kubectl describe pod <pod-name> -n default | grep -A 10 "Liveness\|Readiness"
```

## Updating the Application

### Update Image Tag

1. Edit `argo/apps/fastapi-app.yaml`:
   ```yaml
   image:
     tag: "v1.0.1"  # Change from latest
   ```

2. Commit and push:
   ```bash
   git add argo/apps/fastapi-app.yaml
   git commit -m "Update to version v1.0.1"
   git push
   ```

3. ArgoCD will automatically sync (or manually sync via UI)

### Update Resource Limits

Edit values in `argo/apps/fastapi-app.yaml`:

```yaml
resources:
  limits:
    cpu: 1000m        # Increase from 500m
    memory: 1Gi       # Increase from 512Mi
  requests:
    cpu: 500m
    memory: 512Mi
```

Commit and push - ArgoCD will apply changes.

## Rolling Back

### Via ArgoCD UI

1. Go to Application → History
2. Select previous version
3. Click "Rollback"

### Via CLI

```bash
# Get revision history
kubectl get application fastapi-app -n argocd -o jsonpath='{.status.history}'

# Rollback to previous revision
kubectl patch application fastapi-app -n argocd --type merge \
  -p '{"operation":{"rollback":{"id":"REVISION_ID"}}}'
```

## Scaling the Application

### Manual Scaling

```bash
# Scale replicas
kubectl scale deployment -n default -l app.kubernetes.io/name=fastapi-app --replicas=3
```

Note: ArgoCD will revert this change (self-healing). To persist:

1. Update `apps/fastapi-app.yaml`: `replicaCount: 3`
2. Commit and push

### Enable Auto-Scaling

Edit `argo/apps/fastapi-app.yaml`:

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

## Clean Up

Remove the application:

```bash
# Delete via ArgoCD
kubectl delete -f argo/apps/fastapi-app.yaml

# Verify resources are removed
kubectl get all -n default
```

## Next Steps

1. **Add Custom Domain**: Update ingress with your domain and ACM certificate
2. **Enable HTTPS**: Add SSL/TLS certificate from ACM
3. **Add Monitoring**: Install Prometheus/Grafana for observability
4. **Configure Secrets**: Use AWS Secrets Manager or External Secrets
5. **Set up CI/CD**: Automate image builds and ArgoCD sync
6. **Environment Separation**: Create dev/staging/prod namespaces

## Support

For issues:
- Check ArgoCD logs: `kubectl logs -n argocd deployment/argocd-server`
- Check controller logs: `kubectl logs -n kube-system deployment/aws-load-balancer-controller`
- Review pod events: `kubectl describe pod <pod-name> -n default`
