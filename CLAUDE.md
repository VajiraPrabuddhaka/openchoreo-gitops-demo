# Claude Context: OpenChoreo GitOps Repository

This document provides context for Claude (or other AI assistants) when working with this repository.

## Project Overview

This is a **GitOps repository** for managing OpenChoreo platform resources using **Flux CD**. It follows GitOps principles where Git is the single source of truth for declarative infrastructure and applications.

### Key Technologies
- **OpenChoreo**: An internal developer platform built on Kubernetes
- **Flux CD**: GitOps tool that watches this repository and applies changes to Kubernetes
- **Kustomize**: Tool for customizing Kubernetes configurations (built into kubectl)
- **Kubernetes**: The underlying platform

## Architecture & Design Decisions

### 1. GitOps Pattern
- All platform resources are defined as YAML files in this repository
- Flux watches this repository and automatically applies changes to the cluster
- No manual `kubectl apply` needed after initial Flux setup
- Changes are made by committing to Git, not by directly modifying the cluster

### 2. Namespace Strategy
- **Organization**: `openchoreo` (cluster-scoped resource)
- **Namespace**: `openchoreo` (where all platform resources live)
- Organization name logically maps to namespace name
- All platform resources are deployed to the `openchoreo` namespace

### 3. Kustomize Usage
- **Root kustomization** (`./kustomization.yaml`): Applies namespace and organization
- **Platform kustomization** (`./platform/kustomization.yaml`): Applies all platform resources
  - Sets `namespace: openchoreo` for all resources
  - Adds common label `openchoreo.dev/organization: openchoreo` to all resources
  - Resource files do NOT have hardcoded namespaces (managed by Kustomize)

### 4. Flux Resource Hierarchy
```
GitRepository (openchoreo-gitops)
  └─> Points to: https://github.com/VajiraPrabuddhaka/openchoreo-gitops-demo
      Branch: main

Kustomization (openchoreo-namespace-org)
  └─> Path: ./
  └─> Creates: namespace + organization
  └─> No dependencies

Kustomization (openchoreo-platform)
  └─> Path: ./platform
  └─> Creates: all platform resources
  └─> Depends on: openchoreo-namespace-org
```

## OpenChoreo Custom Resource Definitions (CRDs)

### Resource Types

1. **Organization** (cluster-scoped)
   - Represents a tenant/organization
   - Maps to a Kubernetes namespace
   - Example: `openchoreo` organization → `openchoreo` namespace

2. **ComponentType** (namespaced)
   - Templates for different types of workloads
   - Examples: `http-service`, `scheduled-task`, `web-app`
   - Defines how Components are deployed (Deployment, Service, HTTPRoute, etc.)
   - Uses template expressions like `${metadata.name}`, `${parameters.port}`

3. **Trait** (namespaced)
   - Reusable capabilities that can be attached to Components
   - Examples: `persistent-volume`, `emptydir-volume`
   - Uses patches to modify generated resources

4. **DataPlane** (namespaced)
   - Represents a data plane for running workloads
   - Contains gateway configuration, secret store references
   - Examples: `non-prod`, `prod`

5. **Environment** (namespaced)
   - Represents deployment environments
   - References a DataPlane
   - Examples: `development`, `staging`, `production`
   - Has `isProduction` flag

6. **DeploymentPipeline** (namespaced)
   - Defines promotion paths between environments
   - Examples: `fast-track`, `standard`
   - Specifies which environments can promote to which, with approval requirements

### Important Labels & Annotations

**Labels:**
- `openchoreo.dev/organization: <org-name>` - Links resource to organization (applied by Kustomize)
- `openchoreo.dev/name: <resource-name>` - Resource identifier

**Annotations:**
- `openchoreo.dev/display-name: <name>` - Human-readable name
- `openchoreo.dev/description: <desc>` - Resource description

## Directory Structure

```
./
├── namespace.yaml                    # Namespace definition (managed by root kustomization)
├── organization.yaml                 # Organization CRD (managed by root kustomization)
├── kustomization.yaml               # Root: namespace + organization
│
├── platform/                        # Platform-level resources
│   ├── kustomization.yaml          # Sets namespace + labels for all platform resources
│   ├── component-types/
│   │   ├── services/
│   │   │   └── http-service.yaml
│   │   ├── tasks/
│   │   │   └── scheduled-task.yaml
│   │   └── webapps/
│   │       └── web-app.yaml
│   ├── traits/
│   │   ├── persistent-volume.yaml
│   │   └── emptydir-volume.yaml
│   └── infrastructure/
│       ├── dataplanes/
│       │   ├── non-prod-dataplane.yaml
│       │   └── prod-dataplane.yaml
│       ├── environments/
│       │   ├── dev-environment.yaml
│       │   ├── staging-environment.yaml
│       │   └── prod-environment.yaml
│       └── deployment-pipelines/
│           ├── fast-track-pipeline.yaml
│           └── standard-pipeline.yaml
│
├── projects/                        # Project-specific resources (future)
│   └── sample-project/
│
└── flux/                           # Flux CD resources (NOT managed by Flux)
    ├── README.md
    ├── gitrepository.yaml
    ├── namespace-org-kustomization.yaml
    └── platform-kustomization.yaml
```

## Important Patterns & Conventions

### 1. No Hardcoded Namespaces
- Resource files in `platform/` do NOT specify `namespace:` in metadata
- Namespace is set by `platform/kustomization.yaml`
- Exception: Template variables like `${metadata.namespace}` in ComponentType templates are kept

### 2. No Hardcoded Organization Labels
- The label `openchoreo.dev/organization: openchoreo` is NOT in resource files
- It's applied by `platform/kustomization.yaml` using the `labels` field
- Resource-specific labels (like `openchoreo.dev/name`) are kept in files

### 3. Resource Naming
- Files named after the resource: `http-service.yaml`, `staging-environment.yaml`
- Resource names match file names (without extension)
- Use lowercase with hyphens

### 4. Flux Resources Location
- Flux resources are in `flux/` directory
- These are NOT managed by Flux itself (applied manually with `kubectl apply`)
- They configure Flux to watch and apply the rest of the repository

## Common Tasks

### Adding a New Platform Resource

1. Create YAML file in appropriate subdirectory under `platform/`
2. Do NOT include `namespace:` or `openchoreo.dev/organization` label
3. Add the file path to `platform/kustomization.yaml` under `resources:`
4. Commit and push - Flux will automatically apply

### Modifying an Existing Resource

1. Edit the resource file directly
2. Do NOT add `namespace:` or change organization labels
3. Commit and push - Flux will automatically apply

### Changing Organization/Namespace

1. Update `namespace.yaml` with new namespace name
2. Update `organization.yaml` with new organization name
3. Update `platform/kustomization.yaml`:
   - Change `namespace:` field
   - Change `labels.openchoreo.dev/organization` value
4. Update environment labels if they reference the organization
5. Commit all changes together

### Testing Kustomize Build Locally

```bash
# Test root kustomization
kubectl kustomize .

# Test platform kustomization
kubectl kustomize ./platform

# See what would be applied
kubectl kustomize ./platform | kubectl diff -f -
```

### Force Flux to Sync Immediately

```bash
# Annotate to trigger reconciliation
kubectl annotate gitrepository -n flux-system openchoreo-gitops \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite

kubectl annotate kustomization -n flux-system openchoreo-platform \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

## Important Notes & Caveats

### 1. Organization is Cluster-Scoped
- Organization resources do NOT have a namespace in their metadata
- They are cluster-wide resources
- Only one organization per cluster in this setup

### 2. Health Checks Removed
- Flux Kustomizations DO NOT include `healthChecks`
- OpenChoreo CRDs may not have status conditions that Flux can check
- Resources are considered applied when created, not when "ready"

### 3. No Flux CLI Dependency
- All documentation uses `kubectl` commands
- No `flux` CLI commands in READMEs
- Flux CLI is convenient but not required

### 4. Prune Enabled
- Flux Kustomizations have `prune: true`
- Deleting a resource from Git will delete it from the cluster
- Be careful when removing resources

### 5. Cross-Namespace References
- Some resources reference other namespaces (e.g., `openchoreo-control-plane`)
- These are intentional and should NOT be changed to `openchoreo`
- Example: DataPlane references secrets in `openchoreo-control-plane` namespace

## Design Rationale

### Why Separate Flux Resources?
- Flux resources (`flux/`) configure Flux itself
- They are applied manually once
- They are NOT managed by Flux (bootstrap problem)
- Rest of the repository IS managed by Flux

### Why Two Kustomizations?
1. **namespace-org**: Creates namespace and organization first
2. **platform**: Creates platform resources that depend on namespace existing
3. Ensures proper ordering without race conditions

### Why Kustomize for Labels?
- Keeps resource files clean and focused on their core definition
- Makes it easy to change organization name (one place)
- Follows DRY principle
- Standard Kustomize pattern

### Why No Health Checks?
- OpenChoreo CRDs may not implement status conditions
- Flux would wait indefinitely or timeout
- Creation is sufficient for this use case
- Can be added later if CRDs support it

## Troubleshooting Hints

### Flux Not Applying Changes
- Check GitRepository sync status
- Verify branch name is correct (`main`)
- Check Flux controller logs
- Ensure kustomization doesn't have errors

### Kustomization Failing
- Run `kubectl kustomize ./platform` locally to test
- Look for YAML syntax errors
- Verify all files listed in `resources:` exist
- Check for duplicate resource names

### Resources Missing Namespace
- Verify `platform/kustomization.yaml` has `namespace:` field
- Ensure resource file doesn't have conflicting `namespace:` in metadata

### Wrong Organization Label
- Check `platform/kustomization.yaml` has correct `labels:` section
- Ensure resource files don't have conflicting labels

## Working with Claude

When asking Claude to help with this repository:

### Good Prompts
- "Add a new environment called 'qa' based on staging"
- "Update the prod dataplane to use a different gateway host"
- "Create a new component type for a cron job"
- "Why is my kustomization failing?"

### Provide Context
- Mention which resource type you're working with
- Share error messages from `kubectl describe`
- Indicate if you want to test locally first or push to Git

### Claude Should Remember
- Never add hardcoded namespaces to platform resources
- Never add organization labels manually to resource files
- Always update `platform/kustomization.yaml` when adding new resources
- Use `kubectl` commands, not `flux` CLI commands in documentation
- Organization is cluster-scoped (no namespace field)
- Templates in ComponentTypes use `${...}` expressions

## Related Documentation

- `README.md` - Main repository documentation for users
- `flux/README.md` - Detailed Flux setup and troubleshooting
- OpenChoreo repo: `/Users/vajira/wso2-dev-work/git-repos/choreo` (in workspace)

## Version Information

- **Kustomize API Version**: `kustomize.config.k8s.io/v1beta1`
- **Flux API Versions**:
  - GitRepository: `source.toolkit.fluxcd.io/v1`
  - Kustomization: `kustomize.toolkit.fluxcd.io/v1`
- **OpenChoreo API Version**: `openchoreo.dev/v1alpha1`