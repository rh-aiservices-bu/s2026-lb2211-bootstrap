# OpenShift GitOps Bootstrap Repository

This repository uses OpenShift GitOps (ArgoCD) to deploy and manage applications on OpenShift clusters using the **App-of-Apps pattern** with Helm templating.

## How It Works

### Architecture

```text
bootstrap/application.yaml (Bootstrap App)
    ↓ syncs
applications/ (Helm chart)
    ↓ generates
Application manifests
    ↓ each deploys
Individual Helm charts (image-puller/, tooling/, etc.)
```

1. **Bootstrap Application** (`bootstrap/application.yaml`) is applied to your cluster
2. It syncs the `applications/` Helm chart
3. The Helm chart renders ArgoCD Application manifests (from `applications/templates/`)
4. Each Application deploys its own Helm chart with actual resources

### Repository Structure

```text
.
├── bootstrap/                         # Bootstrap Helm chart
│   ├── Chart.yaml                     # Bootstrap chart metadata
│   ├── values.yaml                    # Default values (targetRevision: main)
│   └── templates/
│       └── application.yaml           # Templated Bootstrap Application
└── applications/                      # App-of-Apps Helm chart
    ├── Chart.yaml                     # Helm chart metadata
    ├── values.yaml                    # Default values (targetRevision: HEAD)
    ├── templates/                     # ArgoCD Application templates
    │   ├── image-puller-app.yaml     # Generates image-puller Application
    │   └── tooling-app.yaml          # Generates tooling Application
    ├── image-puller/                  # Helm chart for image puller DaemonSet
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   └── templates/
    │       └── daemonset.yaml
    └── tooling/                       # Helm chart for custom workbench imagestream
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            └── imagestream.yaml
```

## Working with Environments (dev/main)

The repository uses **CI/CD Helm value overrides** to deploy the same bootstrap chart to
different environments with different `targetRevision` values.

### Bootstrap Helm Chart

The `bootstrap/` directory is a Helm chart that CI/CD renders with environment-specific
values:

```yaml
# bootstrap/values.yaml (default)
git:
  repoURL: https://github.com/rh-aiservices-bu/s2026-lb2211-bootstrap
  targetRevision: main  # Default - override via CI/CD
```

### CI/CD Value Overrides

Your deployment automation overrides `git.targetRevision` when rendering the bootstrap:

**Example for dev environment:**

```yaml
ocp4_workload_gitops_bootstrap_helm_values:
  git:
    targetRevision: dev
```

**Example for production environment:**

```yaml
ocp4_workload_gitops_bootstrap_helm_values:
  git:
    targetRevision: main
```

**Example for feature branch testing:**

```yaml
ocp4_workload_gitops_bootstrap_helm_values:
  git:
    targetRevision: feat/my-feature
```

### Deployment Process

CI/CD renders the bootstrap chart with overrides, then applies it:

```bash
# CI/CD renders with environment-specific values
helm template bootstrap bootstrap/ \
  --set git.targetRevision=dev \
  | oc apply -f -
```

The `targetRevision` value propagates to **all** Application manifests automatically.

### Merging Between Branches (dev → main)

PRs merge cleanly with no conflicts:

- ✅ Single bootstrap chart in repository
- ✅ No environment-specific files to conflict
- ✅ CI/CD controls environment via value overrides
- ✅ Clean merges every time

## Adding a New Application

### Step 1: Create the Helm Chart

Create a new directory under `applications/` with your Helm chart:

```bash
applications/
└── my-new-app/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        └── your-resources.yaml
```

### Step 2: Create the Application Template

Add a new file in `applications/templates/`:

```yaml
# applications/templates/my-new-app.yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-new-app
  namespace: {{ .Values.argocd.namespace }}
  annotations:
    argocd.argoproj.io/sync-wave: "2"  # Deployment order
spec:
  project: {{ .Values.argocd.project }}
  source:
    repoURL: {{ .Values.git.repoURL }}
    targetRevision: {{ .Values.git.targetRevision }}  # Auto-managed!
    path: applications/my-new-app
    helm: {}
  destination:
    namespace: my-namespace
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: {{ .Values.syncPolicy.automated.prune }}
      selfHeal: {{ .Values.syncPolicy.automated.selfHeal }}
    syncOptions:
      {{- range .Values.syncPolicy.syncOptions }}
      - {{ . }}
      {{- end }}
```

### Step 3: Validate

```bash
# Lint the new application Helm chart
helm lint applications/my-new-app

# Test template rendering
helm template my-new-app applications/my-new-app

# Lint the main applications chart
helm lint applications/

# Verify Application manifests render correctly
helm template bootstrap applications/
```

### Step 4: Commit and Push

```bash
git add applications/
git commit -m "Add my-new-app application"
git push
```

ArgoCD will automatically detect and deploy your new application!

## Deployment Order (Sync Waves)

Applications deploy in order based on their sync wave annotation:

- **Wave 0**: `image-puller` - Pre-pulls images to all nodes
- **Wave 1**: `tooling` - Deploys custom workbench imagestream
- **Wave 2+**: Your applications

Lower numbers deploy first. Applications in the same wave deploy in parallel.

## Quick Start

### Initial Deployment

1. Render and apply the bootstrap chart with your environment's `targetRevision`:

   ```bash
   # For dev environment
   helm template bootstrap bootstrap/ \
     --set git.targetRevision=dev \
     | oc apply -f -

   # OR for production environment
   helm template bootstrap bootstrap/ \
     --set git.targetRevision=main \
     | oc apply -f -

   # OR for testing a feature branch
   helm template bootstrap bootstrap/ \
     --set git.targetRevision=feat/my-feature \
     | oc apply -f -
   ```

2. ArgoCD will automatically:
   - Sync the `applications/` Helm chart from the specified branch
   - Generate all Application manifests with the correct `targetRevision`
   - Deploy all applications in order (sync waves: 0, 1, 2...)

### Viewing Applications

```bash
# List all ArgoCD applications
oc get applications -n openshift-gitops

# Check sync status
argocd app list

# View specific application
argocd app get image-puller
```

### Making Changes

1. Modify the Helm chart in `applications/<app-name>/`
2. Validate with `helm lint` and `helm template`
3. Commit and push
4. ArgoCD auto-syncs changes (or use `argocd app sync <app-name>`)

## Troubleshooting

### Application Not Deploying

```bash
# Check Application status
oc describe application <app-name> -n openshift-gitops

# View sync errors
argocd app sync <app-name> --dry-run
```

### Helm Template Errors

```bash
# Test rendering locally
helm template <app-name> applications/<app-name>/

# Check for syntax errors
helm lint applications/<app-name>/
```

### Wrong Branch/Revision

Verify the bootstrap Application has the correct `targetRevision`:

```bash
# Check current bootstrap Application
oc get application bootstrap -n openshift-gitops -o yaml | grep targetRevision

# Re-apply with correct value if needed
helm template bootstrap bootstrap/ \
  --set git.targetRevision=<your-branch> \
  | oc apply -f -
```

## Best Practices

- ✅ Always validate with `helm lint` and `helm template` before committing
- ✅ Use sync waves for deployment order dependencies
- ✅ Keep application Helm charts simple and focused
- ✅ Test changes in dev branch before merging to main
- ✅ Use meaningful sync wave numbers (gaps of 1-5 for future insertions)
- ✅ Document any special deployment requirements in the app's values.yaml
