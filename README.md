# Application GitOps Repository

Kubernetes manifests managed by ArgoCD for the argocd-experiment.

## Overview

This repository contains Kubernetes manifests that are automatically deployed to the karpenter-experiment cluster via ArgoCD. It demonstrates GitOps principles where Git is the single source of truth for cluster state.

## Structure

```
application-gitops/
├── bootstrap/
│   ├── infrastructure-apps.yaml    # Infrastructure app-of-apps
│   └── services-apps.yaml          # Services app-of-apps
├── infrastructure/
│   └── nginx-ingress/              # (Optional) Infrastructure components
└── services/
    ├── demo-nginx/                 # Auto-sync demo application
    │   ├── base/                   # Base Kubernetes manifests
    │   └── overlays/
    │       └── experiment/         # Experiment-specific overrides
    └── demo-redis/                 # Manual-sync demo application
        ├── deployment.yaml
        └── service.yaml
```

## App-of-Apps Hierarchy

```
root (managed by Terraform)
├── infrastructure-apps (from bootstrap/infrastructure-apps.yaml)
│   └── nginx-ingress (commented out)
└── services-apps (from bootstrap/services-apps.yaml)
    ├── demo-nginx (auto-sync enabled)
    └── demo-redis (manual-sync)
```

## Applications

### demo-nginx
- **Type**: Web server
- **Sync Policy**: Automatic (prune + self-heal enabled)
- **Configuration**: Kustomize with base + experiment overlay
- **Purpose**: Demonstrate auto-sync and Kustomize overlays

**Try it:**
```bash
# Edit services/demo-nginx/base/deployment.yaml
# Change replicas from 2 to 5
git add .
git commit -m "Scale demo-nginx to 5 replicas"
git push

# Watch ArgoCD sync automatically
kubectl get applications -n argocd --watch
```

### demo-redis
- **Type**: In-memory data store
- **Sync Policy**: Manual (requires manual approval)
- **Configuration**: Raw Kubernetes manifests
- **Purpose**: Demonstrate manual sync workflow

**Try it:**
```bash
# Edit services/demo-redis/deployment.yaml
# Change image tag from redis:7-alpine to redis:7.2-alpine
git add .
git commit -m "Update Redis version"
git push

# In ArgoCD UI, manually sync the demo-redis application
# Or use CLI: argocd app sync demo-redis
```

## Making Changes

### Auto-Sync Applications (demo-nginx)

1. Edit manifests locally
2. Commit and push to main branch
3. ArgoCD detects changes within 3 minutes
4. Changes automatically sync to cluster
5. If sync fails, ArgoCD retries

### Manual-Sync Applications (demo-redis)

1. Edit manifests locally
2. Commit and push to main branch
3. ArgoCD detects "OutOfSync" status
4. Manually approve sync in UI or CLI
5. Changes deploy to cluster

## Kustomize Usage

The `demo-nginx` application uses Kustomize for environment-specific configuration:

**Base** (`services/demo-nginx/base/`):
- Core deployment and service manifests
- nginx:1.25 image

**Experiment Overlay** (`services/demo-nginx/overlays/experiment/`):
- Overrides image tag to nginx:1.26
- Adds environment-specific labels and annotations
- Adds node selector for Karpenter spot instances

To customize:
```bash
# Edit overlay configuration
vim services/demo-nginx/overlays/experiment/kustomization.yaml

# Or edit patches
vim services/demo-nginx/overlays/experiment/patches.yaml

# Commit and push
git add .
git commit -m "Customize nginx overlay"
git push
```

## Experiments to Try

### 1. Test Auto-Sync

```bash
# Scale demo-nginx
# Edit services/demo-nginx/base/deployment.yaml: replicas: 5
git add . && git commit -m "Scale nginx" && git push
kubectl get pods -n demo-apps --watch
```

### 2. Test Manual Sync

```bash
# Update Redis
# Edit services/demo-redis/deployment.yaml: image: redis:7.2-alpine
git add . && git commit -m "Update redis" && git push
# In ArgoCD UI: Sync demo-redis application
```

### 3. Test Self-Healing

```bash
# Manually change cluster state
kubectl scale deployment demo-nginx -n demo-apps --replicas=10

# Watch ArgoCD detect drift and correct it
kubectl get applications -n argocd --watch
```

### 4. Test Rollback

```bash
# Make a breaking change
# Edit services/demo-nginx/base/deployment.yaml: image: nginx:invalid-tag
git add . && git commit -m "Break nginx" && git push

# Wait for sync failure
# In ArgoCD UI: View History -> Select previous version -> Rollback
```

### 5. Test Kustomize Overlays

```bash
# Add custom annotations
# Edit services/demo-nginx/overlays/experiment/patches.yaml
# Add your custom metadata

git add . && git commit -m "Add annotations" && git push
kubectl describe deployment demo-nginx -n demo-apps | grep Annotations
```

## Best Practices

1. **Commit Messages**: Use clear, descriptive commit messages
2. **Small Changes**: Make incremental changes for easier rollback
3. **Test Locally**: Use `kubectl kustomize` to validate before committing
4. **Review History**: Check ArgoCD UI to see sync history
5. **Monitor Health**: Watch application health in ArgoCD

## Validating Changes Locally

Before pushing:

```bash
# Validate Kustomize build
kubectl kustomize services/demo-nginx/overlays/experiment

# Dry-run apply
kubectl kustomize services/demo-nginx/overlays/experiment | kubectl apply --dry-run=client -f -
```

## Troubleshooting

### Sync Fails

```bash
# Check ArgoCD application status
kubectl describe application demo-nginx -n argocd

# View sync logs in ArgoCD UI
# Or use CLI: argocd app get demo-nginx
```

### Manifest Errors

```bash
# Validate YAML syntax
kubectl apply --dry-run=client -f services/demo-redis/deployment.yaml

# Validate Kustomize
kubectl kustomize services/demo-nginx/overlays/experiment
```

### Rollback to Previous Version

```bash
# Via CLI
argocd app rollback demo-nginx <revision>

# Via UI
# Go to application -> History -> Select revision -> Rollback
```

## Access ArgoCD UI

```bash
# Port-forward to ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Open http://localhost:8080
# Username: admin
# Password: <from command above>
```

## Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kustomize.io/)
- [GitOps Principles](https://www.gitops.tech/)
