# Claude Context: OpenChoreo GitOps Repository

This document provides context for Claude (or other AI assistants) when working with this repository.

## Project Overview

This is a **GitOps repository** for managing OpenChoreo platform resources using **Flux CD**. It follows GitOps principles where Git is the single source of truth for declarative infrastructure and applications.

### Key Technologies
- **OpenChoreo**: An internal developer platform built on Kubernetes
- **Flux CD**: GitOps tool that watches this repository and applies changes to Kubernetes
- **Kubernetes**: The underlying platform

## Architecture & Design Decisions

### 1. GitOps Pattern
- All platform resources are defined as YAML files in this repository
- Flux watches this repository and automatically applies changes to the cluster
- No manual `kubectl apply` needed after initial Flux setup
- Changes are made by committing to Git, not by directly modifying the cluster

### 2. Directory-Based Resource Discovery
- Flux recursively discovers and applies all YAML files in each directory
- **No Kustomize files** are used in this repository
- Resources must include their own `namespace` field when required
- This simplifies resource management - just add YAML files to the appropriate directory

### 3. Namespace Strategy
- **Organization**: `openchoreo` (cluster-scoped resource)
- **Namespace**: `openchoreo` (where all platform and project resources live)
- Organization name logically maps to namespace name
- All namespaced resources explicitly specify `namespace: openchoreo`

### 4. Three-Phase Deployment
The repository uses three Flux Kustomizations with dependencies to ensure proper ordering:

```
GitRepository (openchoreo-gitops)
  └─> Points to: https://github.com/VajiraPrabuddhaka/openchoreo-gitops-demo
      Branch: main

Phase 1: Kustomization (openchoreo-namespace-org)
  └─> Path: ./organization
  └─> Creates: namespace + organization
  └─> No dependencies

Phase 2: Kustomization (openchoreo-platform)
  └─> Path: ./platform
  └─> Creates: component types, traits, dataplanes, environments, pipelines
  └─> Depends on: openchoreo-namespace-org

Phase 3: Kustomization (openchoreo-projects)
  └─> Path: ./projects
  └─> Creates: projects, components, workloads, releases, release bindings
  └─> Depends on: openchoreo-platform
```

## Directory Structure

```
./
├── organization/                        # Namespace and organization resources
│   ├── namespace.yaml                   # Namespace definition
│   └── organization.yaml                # Organization CRD
│
├── platform/                            # Platform-level resources
│   ├── component-types/                 # ComponentType definitions
│   │   ├── http-service.yaml
│   │   ├── scheduled-task.yaml
│   │   └── web-app.yaml
│   ├── traits/                          # Trait definitions
│   │   ├── persistent-volume.yaml
│   │   └── emptydir-volume.yaml
│   └── infrastructure/                  # Infrastructure resources
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
├── projects/                            # Project resources
│   ├── demo-project-1/
│   │   ├── project.yaml                 # Project definition
│   │   └── components/
│   │       └── greeter-service/
│   │           ├── component.yaml       # Component definition
│   │           ├── workload.yaml        # Workload specification
│   │           ├── releases/            # ComponentRelease resources
│   │           │   └── greeter-service-20251205-1.yaml
│   │           └── release-bindings/    # ReleaseBinding resources
│   │               ├── greeter-service-development.yaml
│   │               └── greeter-service-staging.yaml
│   └── ecommerce-demo/
│       ├── project.yaml
│       └── components/
│           ├── order-api/
│           ├── product-api/
│           └── redis-cache/
│
├── flux/                                # Flux CD resources (applied manually)
│   ├── README.md
│   ├── gitrepository.yaml
│   ├── namespace-org-kustomization.yaml
│   ├── platform-kustomization.yaml
│   └── projects-kustomization.yaml
│
├── README.md
└── CLAUDE.md
```

## OpenChoreo Custom Resource Definitions (CRDs)

### Platform-Level Resources

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

### Project-Level Resources

7. **Project** (namespaced)
   - Groups related components together
   - References a DeploymentPipeline
   - Example: `demo-project-1`, `ecommerce-demo`

8. **Component** (namespaced)
   - Represents an application component
   - References a ComponentType and owner Project
   - Contains component parameters (replicas, port, etc.)

9. **Workload** (namespaced)
   - Defines the container image and configuration for a Component
   - Contains container specifications

10. **ComponentRelease** (namespaced)
    - Represents a versioned release of a component
    - Contains release metadata and version information

11. **ReleaseBinding** (namespaced)
    - Binds a ComponentRelease to an Environment
    - Links a specific release to a deployment target

### Important Labels & Annotations

**Labels:**
- `openchoreo.dev/organization: <org-name>` - Links resource to organization
- `openchoreo.dev/name: <resource-name>` - Resource identifier

**Annotations:**
- `openchoreo.dev/display-name: <name>` - Human-readable name
- `openchoreo.dev/description: <desc>` - Resource description

## Important Patterns & Conventions

### 1. Explicit Namespaces
- All namespaced resources MUST include `namespace: openchoreo` in their metadata
- This is required because the repository does not use Kustomize for namespace injection
- Exception: Organization is cluster-scoped and has no namespace

### 2. Resource Naming
- Files named after the resource: `http-service.yaml`, `staging-environment.yaml`
- Resource names match file names (without extension)
- Use lowercase with hyphens

### 3. Project Structure Convention
Each project follows this structure:
```
projects/<project-name>/
├── project.yaml                    # Project resource
└── components/
    └── <component-name>/
        ├── component.yaml          # Component resource
        ├── workload.yaml           # Workload resource
        ├── releases/               # ComponentRelease resources
        │   └── <release-name>.yaml
        └── release-bindings/       # ReleaseBinding resources
            └── <binding-name>.yaml
```

### 4. Flux Resources Location
- Flux resources are in `flux/` directory
- These are NOT managed by Flux itself (applied manually with `kubectl apply`)
- They configure Flux to watch and apply the rest of the repository

## Common Tasks

### Adding a New Platform Resource

1. Create YAML file in appropriate subdirectory under `platform/`
2. Include `namespace: openchoreo` in metadata
3. Commit and push - Flux will automatically discover and apply

### Adding a New Project

1. Create project directory: `projects/<project-name>/`
2. Create `project.yaml` with the Project resource
3. Create `components/` subdirectory for components
4. Commit and push - Flux will automatically discover and apply

### Adding a New Component

1. Create component directory: `projects/<project-name>/components/<component-name>/`
2. Create `component.yaml` with Component resource
3. Create `workload.yaml` with Workload resource
4. Create `releases/` and `release-bindings/` subdirectories
5. Add release and binding resources as needed
6. Commit and push - Flux will automatically discover and apply

### Adding a New Release

1. Create release file in `projects/<project-name>/components/<component-name>/releases/`
2. Use naming convention: `<component-name>-<date>-<version>.yaml`
3. Commit and push

### Adding a Release Binding

1. Create binding file in `projects/<project-name>/components/<component-name>/release-bindings/`
2. Use naming convention: `<component-name>-<environment>.yaml`
3. Reference the appropriate release and environment
4. Commit and push

### Modifying an Existing Resource

1. Edit the resource file directly
2. Commit and push - Flux will automatically apply the changes

### Force Flux to Sync Immediately

```bash
# Trigger reconciliation for all resources
kubectl annotate gitrepository -n flux-system openchoreo-gitops \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite

kubectl annotate kustomization -n flux-system openchoreo-namespace-org \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite

kubectl annotate kustomization -n flux-system openchoreo-platform \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite

kubectl annotate kustomization -n flux-system openchoreo-projects \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

## Important Notes & Caveats

### 1. Organization is Cluster-Scoped
- Organization resources do NOT have a namespace in their metadata
- They are cluster-wide resources
- Only one organization per cluster in this setup

### 2. No Kustomize Files
- This repository does NOT use `kustomization.yaml` files
- Flux discovers resources directly from directory contents
- All resources must be self-contained with explicit namespaces

### 3. Health Checks Removed
- Flux Kustomizations DO NOT include `healthChecks`
- OpenChoreo CRDs may not have status conditions that Flux can check
- Resources are considered applied when created, not when "ready"

### 4. Prune Enabled
- Flux Kustomizations have `prune: true`
- Deleting a resource from Git will delete it from the cluster
- Be careful when removing resources

### 5. Cross-Namespace References
- Some resources reference other namespaces (e.g., `openchoreo-control-plane`, `openchoreo-data-plane`)
- These are intentional and should NOT be changed to `openchoreo`
- Example: DataPlane references secrets in `openchoreo-control-plane` namespace
- Example: HTTPRoute references gateway in `openchoreo-data-plane` namespace

## Design Rationale

### Why Separate Organization Directory?
- Keeps namespace and organization resources isolated
- Clear separation of concerns
- Enables independent lifecycle management

### Why Three Kustomizations?
1. **namespace-org**: Creates namespace and organization first
2. **platform**: Creates platform resources that depend on namespace existing
3. **projects**: Creates project resources that depend on platform resources
4. Ensures proper ordering without race conditions

### Why No Kustomize Files?
- Simpler resource management - just add YAML files
- Flux handles directory-based discovery automatically
- Each resource is self-contained and explicit
- Easier to understand what gets deployed

### Why No Health Checks?
- OpenChoreo CRDs may not implement status conditions
- Flux would wait indefinitely or timeout
- Creation is sufficient for this use case
- Can be added later if CRDs support it

## Troubleshooting Hints

### Flux Not Applying Changes
- Check GitRepository sync status: `kubectl get gitrepository -n flux-system`
- Verify branch name is correct (`main`)
- Check Flux controller logs
- Ensure kustomizations don't have errors

### Resources Not Appearing
- Verify YAML files are valid
- Check that namespace is included in resource metadata
- Look at Flux Kustomization status for errors
- Run: `kubectl describe kustomization -n flux-system openchoreo-projects`

### Dependency Issues
- Ensure resources are in the correct directory for their phase
- Platform resources belong in `platform/`
- Project resources belong in `projects/`
- Namespace/org belong in `organization/`

### Common Issues

**Resources missing namespace:**
- Add `namespace: openchoreo` to the resource metadata
- Exception: Organization is cluster-scoped

**Release binding not working:**
- Verify the referenced release exists
- Check that the environment name is correct
- Ensure component name matches

## Verification Commands

```bash
# Check Flux resources
kubectl get gitrepository -n flux-system
kubectl get kustomization -n flux-system

# Check OpenChoreo platform resources
kubectl get organizations
kubectl get namespaces openchoreo
kubectl get componenttypes,traits,dataplanes,environments,deploymentpipelines -n openchoreo

# Check project resources
kubectl get projects,components,workloads -n openchoreo
kubectl get componentreleases,releasebindings -n openchoreo

# View Flux logs
kubectl logs -n flux-system deploy/source-controller --follow
kubectl logs -n flux-system deploy/kustomize-controller --follow
```

## Working with Claude

When asking Claude to help with this repository:

### Good Prompts
- "Add a new environment called 'qa' based on staging"
- "Create a new project with a component"
- "Add a release binding for staging environment"
- "Update the prod dataplane to use a different gateway host"
- "Create a new component type for a cron job"

### Provide Context
- Mention which resource type you're working with
- Share error messages from `kubectl describe`
- Indicate if you want to test locally first or push to Git

### Claude Should Remember
- Always add `namespace: openchoreo` to namespaced resources
- Organization is cluster-scoped (no namespace field)
- No Kustomize files - resources are discovered directly
- Use `kubectl` commands, not `flux` CLI commands in documentation
- Follow the project structure convention for new projects/components
- Templates in ComponentTypes use `${...}` expressions

## Related Documentation

- `README.md` - Main repository documentation for users
- `flux/README.md` - Detailed Flux setup and troubleshooting

## Version Information

- **Flux API Versions**:
  - GitRepository: `source.toolkit.fluxcd.io/v1`
  - Kustomization: `kustomize.toolkit.fluxcd.io/v1`
- **OpenChoreo API Version**: `openchoreo.dev/v1alpha1`