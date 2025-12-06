# OpenChoreo GitOps Demo

This repository contains a GitOps configuration for deploying OpenChoreo platform resources using Flux.

## Overview

OpenChoreo is an internal developer platform built on Kubernetes. This repository manages platform-level configurations including component types, traits, environments, dataplanes, and deployment pipelines.

## Repository Structure

```
./
├── namespace.yaml                    # Namespace definition
├── organization.yaml                 # Organization CRD
├── kustomization.yaml               # Root: namespace + organization
│
├── platform/                        # Platform-level resources
│   ├── kustomization.yaml
│   ├── component-types/             # ComponentType definitions
│   ├── traits/                      # Trait definitions
│   └── infrastructure/              # DataPlanes, Environments, Pipelines
│
├── projects/                        # Project resources (Components, Releases)
│   ├── kustomization.yaml
│   └── demo-project-1/
│       ├── project.yaml
│       └── components/
│           └── greeter-service/
│               ├── component.yaml
│               ├── workload.yaml
│               └── releases/
│
├── projects-release-bindings/       # ReleaseBindings (separate for webhook compatibility)
│   ├── kustomization.yaml
│   └── demo-project-1/
│       └── components/
│           └── greeter-service/
│               └── greeter-service-development.yaml
│
└── flux/                           # Flux CD resources (applied manually)
    ├── gitrepository.yaml
    ├── namespace-org-kustomization.yaml
    ├── platform-kustomization.yaml
    ├── projects-kustomization.yaml
    └── projects-release-bindings-kustomization.yaml
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

# 4. Create projects and components (ComponentReleases)
kubectl apply -f flux/projects-kustomization.yaml

# 5. Create release bindings (must be after step 4)
kubectl apply -f flux/projects-release-bindings-kustomization.yaml
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
- **Sample Project**: demo-app component
  - Component definition and workload
  - 3 releases
  - Environment bindings for dev, staging, and prod

All resources are automatically labeled with `openchoreo.dev/organization: openchoreo`.

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
5. **Release bindings** are created last (via `projects-release-bindings-kustomization.yaml`) with dependency on step 4
6. **Automatic sync** happens based on the configured interval (5 minutes)
7. **Changes pushed to Git** are automatically applied to the cluster

### Kustomize Integration

The repository uses Kustomize to:
- Set namespace for all resources (`openchoreo`)
- Apply common labels (`openchoreo.dev/organization: openchoreo`)
- Organize resources hierarchically (platform → projects → components)
- Enable modular management (each project/component has its own kustomization)

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
Kustomization (openchoreo-projects)          → projects, components, workloads, componentreleases
       │
       ▼
Kustomization (openchoreo-projects-release-bindings) → releasebindings
```

### Why ReleaseBindings Are Separate

ReleaseBindings are managed in a separate Flux Kustomization (`projects-release-bindings/`) due to OpenChoreo's validating admission webhook. The webhook validates that the `ComponentRelease` referenced by a `ReleaseBinding` exists before allowing creation.

When Flux applies resources, it performs a server-side dry-run first. If `ComponentRelease` and `ReleaseBinding` are in the same Kustomization, the dry-run may validate the `ReleaseBinding` before the `ComponentRelease` exists, causing the webhook to reject it.

By splitting them into separate Kustomizations with a dependency chain, we ensure:
1. `ComponentRelease` resources are fully created first
2. `ReleaseBinding` resources are only applied after step 1 completes
3. The webhook validation passes because the referenced release exists

## Making Changes

### Adding New Resources

**Platform Resources:**
1. Add your resource YAML files to the appropriate directory under `platform/`
2. Update `platform/kustomization.yaml` to include the new resource
3. Commit and push to the repository
4. Flux will automatically sync the changes

**New Project:**
1. Create a new project directory under `projects/`
2. Create a `kustomization.yaml` in the project directory
3. Add the project directory to `projects/kustomization.yaml`
4. Commit and push to the repository

**New Component:**
1. Create a component directory under `projects/<project-name>/components/`
2. Create a `kustomization.yaml` listing component resources (component, workload, releases)
3. Add the component directory to the project's `kustomization.yaml`
4. Commit and push to the repository

**New ReleaseBinding:**
1. Create the ReleaseBinding YAML file (do NOT include `namespace:` field)
2. Place it in `projects-release-bindings/<project-name>/components/<component-name>/`
3. Update the component's `kustomization.yaml` in `projects-release-bindings/` to include it
4. Ensure the referenced `ComponentRelease` exists in `projects/<project-name>/components/<component-name>/releases/`
5. Commit and push to the repository

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
kubectl describe kustomization -n flux-system openchoreo-projects-release-bindings
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

### Change Organization/Namespace

1. Update `namespace.yaml` and `organization.yaml` with the new name
2. Update the `namespace` field in `platform/kustomization.yaml`
3. Update the `commonLabels` in `platform/kustomization.yaml`
4. Commit and push changes

### Disable Auto-Pruning

Set `prune: false` in the Flux Kustomization specs to prevent automatic deletion of resources removed from Git.

## Documentation

- [Flux Resources Documentation](flux/README.md) - Detailed Flux setup and usage

## Related Resources

- [OpenChoreo Documentation](https://github.com/VajiraPrabuddhaka/choreo)
- [Flux Documentation](https://fluxcd.io/docs/)
- [Kustomize Documentation](https://kustomize.io/)
