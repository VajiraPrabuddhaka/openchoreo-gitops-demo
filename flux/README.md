# Flux Resources for OpenChoreo GitOps

This directory contains Flux resources to deploy the OpenChoreo platform configuration.

## Structure

- `gitrepository.yaml` - Flux GitRepository pointing to this repo
- `namespace-org-kustomization.yaml` - Flux Kustomization for namespace and organization
- `platform-kustomization.yaml` - Flux Kustomization for platform resources
- `projects-kustomization.yaml` - Flux Kustomization for projects and components

## Quick Start

Apply the Flux resources in order:

```bash
# 1. Create the GitRepository source
kubectl apply -f flux/gitrepository.yaml

# 2. Create namespace and organization
kubectl apply -f flux/namespace-org-kustomization.yaml

# 3. Create platform resources (depends on namespace-org)
kubectl apply -f flux/platform-kustomization.yaml

# 4. Create projects and components (depends on platform)
kubectl apply -f flux/projects-kustomization.yaml
```

## What Gets Deployed

### Phase 1: Namespace & Organization
- Kubernetes namespace: `openchoreo`
- OpenChoreo Organization: `openchoreo`

### Phase 2: Platform Resources
All resources deployed to the `openchoreo` namespace:

**Component Types:**
- http-service
- scheduled-task
- web-app

**Traits:**
- persistent-volume
- emptydir-volume

**Infrastructure - DataPlanes:**
- non-prod
- prod

**Infrastructure - Environments:**
- development
- staging
- production

**Infrastructure - Deployment Pipelines:**
- fast-track
- standard

### Phase 3: Projects & Components
All project resources deployed to the `openchoreo` namespace:

**Sample Project:**
- Component: `demo-app` (web-service)
- Workload definitions
- Releases (3 releases)
- Environment bindings (dev, staging, prod)

## Verification

### Check Flux Status

```bash
# Check GitRepository sync
kubectl get gitrepository -n flux-system openchoreo-gitops

# Check Kustomizations
kubectl get kustomization -n flux-system
```

### Check Applied Resources

```bash
# Namespace and Organization
kubectl get namespace openchoreo
kubectl get organizations openchoreo

# Platform resources
kubectl get componenttypes -n openchoreo
kubectl get traits -n openchoreo
kubectl get dataplanes -n openchoreo
kubectl get environments -n openchoreo
kubectl get deploymentpipelines -n openchoreo

# Project resources
kubectl get components -n openchoreo
kubectl get releases -n openchoreo
```

## Troubleshooting

### View Flux Controller Logs
```bash
kubectl logs -n flux-system deploy/source-controller --follow
kubectl logs -n flux-system deploy/kustomize-controller --follow
```

### Check Specific Kustomization
```bash
kubectl get kustomization -n flux-system openchoreo-namespace-org -o yaml
kubectl get kustomization -n flux-system openchoreo-platform -o yaml
kubectl get kustomization -n flux-system openchoreo-projects -o yaml
```

### Describe Resources
```bash
kubectl describe gitrepository -n flux-system openchoreo-gitops
kubectl describe kustomization -n flux-system openchoreo-namespace-org
kubectl describe kustomization -n flux-system openchoreo-platform
kubectl describe kustomization -n flux-system openchoreo-projects
```

### Force Reconciliation
```bash
# Trigger immediate reconciliation by annotating the resource
kubectl annotate gitrepository -n flux-system openchoreo-gitops reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
kubectl annotate kustomization -n flux-system openchoreo-namespace-org reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
kubectl annotate kustomization -n flux-system openchoreo-platform reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
kubectl annotate kustomization -n flux-system openchoreo-projects reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

## Resource Dependencies

```
GitRepository (openchoreo-gitops)
    │
    ├── Kustomization (openchoreo-namespace-org)
    │   ├── namespace.yaml
    │   └── organization.yaml
    │
    ├── Kustomization (openchoreo-platform)
    │   └── platform/**/*.yaml
    │       (depends on openchoreo-namespace-org)
    │
    └── Kustomization (openchoreo-projects)
        └── projects/**/kustomization.yaml (hierarchical)
            (depends on openchoreo-platform)
```

## Configuration

### Change Sync Interval

Edit the `interval` field in the Kustomization specs:
- `1m` - Frequent updates (development)
- `5m` - Moderate updates (default for platform)
- `10m` - Less frequent (default for namespace/org)

### Change Git Branch

Edit `gitrepository.yaml`:
```yaml
spec:
  ref:
    branch: main  # Change to your branch name
```

### Disable Auto-Pruning

Set `prune: false` in the Kustomization specs to prevent automatic deletion of resources removed from Git.