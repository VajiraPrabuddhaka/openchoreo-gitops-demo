# OpenChoreo GitOps Demo

This repository contains a GitOps configuration for deploying OpenChoreo platform resources using Flux.

## Overview

OpenChoreo is an internal developer platform built on Kubernetes. This repository manages platform-level configurations including component types, traits, environments, dataplanes, and deployment pipelines.

## Repository Structure

```
./
├── organization/                    # Organization and namespace resources
│   ├── namespace.yaml              # Namespace definition
│   └── organization.yaml           # Organization CRD
│
├── platform/                        # Platform-level resources
│   ├── component-types/             # ComponentType definitions
│   │   ├── http-service.yaml
│   │   ├── scheduled-task.yaml
│   │   └── web-app.yaml
│   ├── traits/                      # Trait definitions
│   │   ├── persistent-volume.yaml
│   │   └── emptydir-volume.yaml
│   └── infrastructure/              # DataPlanes, Environments, Pipelines
│       ├── dataplanes/
│       ├── environments/
│       └── deployment-pipelines/
│
├── projects/                        # Project resources
│   ├── demo-project-1/
│   │   ├── project.yaml
│   │   └── components/
│   │       └── greeter-service/
│   │           ├── component.yaml
│   │           ├── workload.yaml
│   │           ├── releases/
│   │           └── release-bindings/
│   └── ecommerce-demo/
│       ├── project.yaml
│       └── components/
│           ├── order-api/
│           ├── product-api/
│           └── redis-cache/
│
└── flux/                           # Flux CD resources (applied manually)
    ├── gitrepository.yaml
    ├── namespace-org-kustomization.yaml
    ├── platform-kustomization.yaml
    └── projects-kustomization.yaml
```

## Prerequisites

- Kubernetes cluster with OpenChoreo installed
- Flux installed in the cluster (in `flux-system` namespace)
- Access to this Git repository

## Deployment

### Quick Start

Deploy the platform resources using Flux:

```bash
# 1. Create the GitRepository source
kubectl apply -f flux/gitrepository.yaml

# 2. Create namespace and organization
kubectl apply -f flux/namespace-org-kustomization.yaml

# 3. Create platform resources
kubectl apply -f flux/platform-kustomization.yaml

# 4. Create projects and components along with component releases and release bindings
kubectl apply -f flux/projects-kustomization.yaml
```

### What Gets Deployed

**Organization & Namespace:**
- Organization: `openchoreo`
- Namespace: `openchoreo`

**Platform Resources** (in `openchoreo` namespace):
- **Component Types**: http-service, scheduled-task, web-app
- **Traits**: persistent-volume, emptydir-volume
- **DataPlanes**: non-prod, prod
- **Environments**: development, staging, production
- **Deployment Pipelines**: fast-track, standard

**Project Resources** (in `openchoreo` namespace):
- **demo-project-1**: greeter-service component
  - Component definition and workload
  - Releases and release bindings
- **ecommerce-demo**: order-api, product-api, redis-cache components
  - Component definitions and workloads
  - Releases and release bindings

## Verification

### Check Deployment Status

```bash
# Check Flux resources
kubectl get gitrepository -n flux-system
kubectl get kustomization -n flux-system

# Check OpenChoreo resources
kubectl get organizations
kubectl get namespaces openchoreo
kubectl get componenttypes,traits,dataplanes,environments,deploymentpipelines -n openchoreo
kubectl get components,releases -n openchoreo
```

## How It Works

### GitOps Workflow

1. **Flux watches this Git repository** via the GitRepository resource
2. **Namespace & Organization** are created first (via `namespace-org-kustomization.yaml`)
3. **Platform resources** are created next (via `platform-kustomization.yaml`) with dependency on step 2
4. **Project resources** are created (via `projects-kustomization.yaml`) with dependency on step 3
5. **Automatic sync** happens based on the configured interval (5 minutes)
6. **Changes pushed to Git** are automatically applied to the cluster

### Directory-Based Resource Discovery

Flux recursively discovers and applies all YAML files in the `organization/`, `platform/`, and `projects/` directories. This simplifies resource management:
- Add new resources by placing YAML files in the appropriate directory
- Resources are automatically picked up on the next sync cycle
- Resources must include their own `namespace` field when required

### Resource Dependencies

```
GitRepository (openchoreo-gitops)
       │
       ▼
Kustomization (openchoreo-namespace-org)     → namespace, organization
       │
       ▼
Kustomization (openchoreo-platform)          → componenttypes, traits, dataplanes, environments, pipelines
       │
       ▼
Kustomization (openchoreo-projects)          → projects, components, workloads, componentreleases, releasebindings
```

## Making Changes

### Adding New Resources

**Platform Resources:**
1. Add your resource YAML files to the appropriate directory under `platform/`
2. Commit and push to the repository
3. Flux will automatically discover and sync the changes

**New Project:**
1. Create a new project directory under `projects/`
2. Add a `project.yaml` with the Project resource
3. Commit and push to the repository

**New Component:**
1. Create a component directory under `projects/<project-name>/components/`
2. Add `component.yaml` and `workload.yaml` files
3. Create `releases/` and `release-bindings/` subdirectories as needed
4. Commit and push to the repository

**New Release or ReleaseBinding:**
1. Create the YAML file in the appropriate directory (`releases/` or `release-bindings/`)
2. Commit and push to the repository

### Modifying Existing Resources

1. Edit the resource file directly
2. Commit and push to the repository
3. Flux will automatically apply the changes

### Force Immediate Sync

```bash
# Trigger immediate reconciliation
kubectl annotate gitrepository -n flux-system openchoreo-gitops reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
kubectl annotate kustomization -n flux-system openchoreo-platform reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

## Troubleshooting

### View Flux Logs

```bash
kubectl logs -n flux-system deploy/source-controller --follow
kubectl logs -n flux-system deploy/kustomize-controller --follow
```

### Check Resource Status

```bash
kubectl describe gitrepository -n flux-system openchoreo-gitops
kubectl describe kustomization -n flux-system openchoreo-namespace-org
kubectl describe kustomization -n flux-system openchoreo-platform
kubectl describe kustomization -n flux-system openchoreo-projects
```

### Common Issues

**Kustomization stuck or failing:**
- Check the kustomization status: `kubectl get kustomization -n flux-system openchoreo-platform -o yaml`
- Look for error messages in the status conditions
- Verify all resource files are valid YAML

**Resources not appearing in cluster:**
- Verify Flux is running: `kubectl get pods -n flux-system`
- Check if GitRepository is synced: `kubectl get gitrepository -n flux-system openchoreo-gitops`
- Ensure the Git repository URL and branch are correct

## Configuration

### Change Sync Interval

Edit the Flux Kustomization files in `flux/`:

```yaml
spec:
  interval: 5m  # Change to desired interval (e.g., 1m, 10m, 30m)
```

### Disable Auto-Pruning

Set `prune: false` in the Flux Kustomization specs to prevent automatic deletion of resources removed from Git.

## Documentation

- [Flux Resources Documentation](flux/README.md) - Detailed Flux setup and usage

## Related Resources

- [OpenChoreo Documentation](https://github.com/VajiraPrabuddhaka/choreo)
- [Flux Documentation](https://fluxcd.io/docs/)
- [Kustomize Documentation](https://kustomize.io/)
