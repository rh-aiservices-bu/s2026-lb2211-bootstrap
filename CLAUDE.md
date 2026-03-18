# OpenShift GitOps Bootstrap Repository

This repository uses OpenShift GitOps (ArgoCD) with an **App-of-Apps pattern** using Helm templating to deploy applications across environments.

## Repository Architecture

### Structure Overview

```
bootstrap/                         # Bootstrap Helm chart
├── Chart.yaml                     # Bootstrap chart metadata
├── values.yaml                    # Default values (git.targetRevision: main)
└── templates/
    └── application.yaml           # Templated Bootstrap Application

applications/                      # App-of-Apps Helm chart
├── Chart.yaml                     # Helm metadata for applications chart
├── values.yaml                    # Default values (targetRevision: HEAD)
├── templates/                     # ArgoCD Application templates
│   ├── <app>-app.yaml            # Each generates an ArgoCD Application
│   └── ...
└── <app-name>/                    # Individual application Helm charts
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        └── *.yaml                 # Kubernetes resources
```

### How It Works

1. **CI/CD** renders `bootstrap/` Helm chart with environment-specific `git.targetRevision`
2. **Bootstrap Application** deploys the `applications/` Helm chart from specified branch
3. The `applications/` Helm chart renders ArgoCD Application manifests from
   `applications/templates/`
4. Each Application deploys its own Helm chart from `applications/<app-name>/`
5. All applications automatically use the correct `targetRevision` via templating

## GitOps Workflow

- **Bootstrap**: CI/CD renders `bootstrap/` Helm chart with `git.targetRevision` override
- **Application Discovery**: ArgoCD renders Application manifests from `applications/templates/`
- **Deployment Order**: Controlled via `sync-wave` annotations (0, 1, 2, ...)
- **Auto-Sync**: All applications have automated sync with prune and self-heal enabled
- **Branch Strategy**: CI/CD controls which branch via Helm value overrides

## Application Management Rules

### Creating New Applications

1. **Create Application Helm Chart** under `applications/<app-name>/`:
   - **MUST** include: `Chart.yaml`, `values.yaml`, `templates/` directory
   - Store all Kubernetes resources in `templates/`
   - Keep application-specific configuration in `values.yaml`

2. **Create Application Template** in `applications/templates/<app-name>-app.yaml`:
   - Copy existing template as reference (e.g., `tooling-app.yaml`)
   - Set appropriate `sync-wave` annotation for deployment order
   - Use templated values: `{{ .Values.git.targetRevision }}`, `{{ .Values.argocd.namespace }}`
   - Set correct `destination.namespace` for where resources will be deployed
   - Path should be: `applications/<app-name>`

3. **Naming Conventions**:
   - Use lowercase with hyphens (e.g., `image-puller`, `tooling`)
   - Application template files: `<app-name>-app.yaml`
   - Helm chart directories: `applications/<app-name>/`

4. **Validation Requirements** (CRITICAL):
   - **MUST** run `helm lint applications/<app-name>/` for the application chart
   - **MUST** run `helm template <app-name> applications/<app-name>/` to verify resource rendering
   - **MUST** run `helm lint applications/` for the meta-chart
   - **MUST** run `helm template bootstrap applications/` to verify Application manifest generation
   - Fix all linting errors and rendering issues before marking work complete

5. **Deployment Order** (sync waves):
   - Use annotation: `argocd.argoproj.io/sync-wave: "<number>"`
   - Lower numbers deploy first (0, 1, 2, ...)
   - Leave gaps for future insertions (0, 5, 10 instead of 0, 1, 2)
   - Current waves:
     - Wave 0: `image-puller` (pre-pull images)
     - Wave 1: `tooling` (imagestreams)
     - Wave 2+: Available for new apps

### Modifying Existing Applications

1. **Read First**: Always use Read tool to examine current state before modifications
2. **Application Charts**: Modify `applications/<app-name>/` Helm chart directly
3. **Application Templates**: Only modify `applications/templates/<app-name>-app.yaml` if changing ArgoCD Application spec
4. **Do NOT Modify**: `git.targetRevision` in templates - this is managed via bootstrap
5. **Run Validation**: After changes, run helm lint/template on both the app chart and meta-chart
6. **Test Rendering**: Verify with `helm template bootstrap applications/` to see final Application manifests

### CI/CD Helm Value Overrides

The repository uses **CI/CD Helm value overrides** to deploy different environments:

**Bootstrap Helm Chart:**

- Single templated chart in `bootstrap/` directory
- Default `git.targetRevision: main` in `bootstrap/values.yaml`
- CI/CD overrides this value per environment

**Configuration Rules:**

- **NEVER** hardcode `targetRevision` in Application templates (`applications/templates/*.yaml`)
- **ALWAYS** use: `targetRevision: {{ .Values.git.targetRevision }}` in templates
- **Environment Selection**: CI/CD overrides `git.targetRevision` when rendering bootstrap
- **Single Source**: One bootstrap chart for all environments

**Example CI/CD Override Pattern:**

```yaml
ocp4_workload_gitops_bootstrap_helm_values:
  git:
    targetRevision: dev  # or 'main', or 'feat/branch-name'
```

### Merging Between Branches

When creating PRs from dev→main:

- **No Conflicts**: Single bootstrap chart, no environment-specific files
- **Clean Merges**: All changes merge normally
- **No Setup Required**: No special git configuration needed
- **CI/CD Controlled**: Environment selection handled by deployment automation

**Important for AI Assistants:**

- Do NOT modify `bootstrap/` files unless specifically requested
- Do NOT hardcode `targetRevision` anywhere - always use template variables
- When asked about environment configuration, refer to CI/CD value override approach
- Bootstrap chart should work for all environments via value overrides

## Prerequisites and Dependencies

- **Operators**: All required operators must be deployed beforehand
- **Namespaces**: Target namespaces must be pre-created (or use `syncOptions: CreateNamespace=true`)
- **Deployment Order**: Use sync waves for applications with dependencies
- **Image Availability**: Use `image-puller` DaemonSet (wave 0) for critical images

## Quality Gates

Before marking any application work as complete:

- ✅ Application Helm chart lints without errors: `helm lint applications/<app-name>/`
- ✅ Application resources render correctly: `helm template <app-name> applications/<app-name>/`
- ✅ Meta-chart lints without errors: `helm lint applications/`
- ✅ Application manifests render correctly: `helm template bootstrap applications/`
- ✅ All YAML is syntactically valid
- ✅ Required Kubernetes resource fields are present
- ✅ Sync wave annotation is set appropriately
- ✅ `targetRevision` uses template variable, not hardcoded value
- ✅ Destination namespace is correct

## Working with This Repository

1. **Read First**: Always read existing files before modifying
2. **Use Absolute Paths**: Follow repository root conventions
3. **Validate Always**: Never skip helm lint/template validation (both levels)
4. **Follow Patterns**: Match existing ArgoCD Application templates and Helm chart patterns
5. **Two-Level Helm**: Remember there are TWO Helm charts:
   - Meta-chart (`applications/`) generates Application manifests
   - App charts (`applications/<app-name>/`) generate actual Kubernetes resources
6. **Ask When Uncertain**: Especially about naming, deployment order, or sync waves

## Repository Context

- **Primary branch**: `main`
- **Dev branch**: `dev`
- **Bootstrap chart**: `bootstrap/` (Helm chart with templated Application)
- **Applications meta-chart**: `applications/` (Chart.yaml, values.yaml, templates/)
- **Individual apps**: `applications/<app-name>/` (each is a Helm chart)
- **GitOps tool**: OpenShift GitOps (ArgoCD)
- **Chart storage**: In-repo (not external Helm repositories)
- **Pattern**: App-of-Apps with Helm templating
- **Environment Strategy**: CI/CD Helm value overrides for `git.targetRevision`

## Common Operations

### Adding an Application

1. Create Helm chart: `applications/my-app/`
2. Create Application template: `applications/templates/my-app-app.yaml`
3. Validate both: `helm lint applications/my-app/` and `helm template bootstrap applications/`
4. Commit and push - ArgoCD auto-deploys

### Deploying to Different Environments

CI/CD renders bootstrap with environment-specific values:

```bash
# Dev environment
helm template bootstrap bootstrap/ \
  --set git.targetRevision=dev | oc apply -f -

# Production environment
helm template bootstrap bootstrap/ \
  --set git.targetRevision=main | oc apply -f -
```

### Viewing Generated Applications

```bash
helm template bootstrap applications/
```

This shows what ArgoCD Application manifests will be created.
