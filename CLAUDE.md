# OpenShift GitOps Bootstrap Repository

This repository hosts configuration files to install and configure components on an OpenShift cluster using OpenShift GitOps (ArgoCD).

## Repository Structure

```
.
├── bootstrap/
│   └── application.yaml          # Bootstrap Application pointing to applications/
└── applications/                 # Application Helm charts and configurations
    └── <app-name>/              # Each application in its own directory
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

## GitOps Workflow

- **Bootstrap**: The cluster deployment mechanism creates the ArgoCD Application from `bootstrap/application.yaml`
- **Application Sync**: ArgoCD monitors the `applications/` folder and deploys all applications
- **Sync Policy**: Automated with prune and self-heal enabled
- **Environments**: Single environment per branch (dev/prod handled via git branches)

## Application Management Rules

### Creating New Applications

1. **Directory Structure**: Each application must have its own subdirectory under `applications/`
   - Store the complete Helm chart in the application directory
   - Include: `Chart.yaml`, `values.yaml`, and `templates/` folder

2. **Naming Conventions**: Names will be provided per application
   - Follow specific naming as directed for each new application
   - Maintain consistency within the application structure

3. **Validation Requirements** (CRITICAL):
   - **MUST** run `helm lint <app-path>` before completion
   - **MUST** run `helm template <app-name> <app-path>` to verify rendering
   - Fix all linting errors and rendering issues before marking work complete
   - Validate YAML syntax and Kubernetes resource schemas

### Modifying Existing Applications

1. Always use Read tool to examine current state before modifications
2. Maintain existing chart structure and patterns
3. Run validation after changes (helm lint + helm template)
4. Document significant changes in chart values or templates

## Prerequisites and Dependencies

- **Operators**: All required operators are deployed beforehand
- **Namespaces**: Target namespaces are pre-created
- **Deployment Order**: May be required for some applications; clarify if uncertain

## Quality Gates

Before marking any application work as complete:
- ✅ Helm chart lints without errors
- ✅ Helm template renders successfully
- ✅ All YAML is syntactically valid
- ✅ Required Kubernetes resource fields are present
- ✅ ArgoCD Application manifest follows bootstrap pattern

## Working with This Repository

1. **Read First**: Always read existing files before modifying
2. **Use Absolute Paths**: Follow repository root conventions
3. **Validate Always**: Never skip helm lint/template validation
4. **Follow Patterns**: Match existing ArgoCD and Helm patterns
5. **Ask When Uncertain**: Especially about naming or deployment order

## Repository Context

- Primary branch: `main`
- Bootstrap path: `bootstrap/application.yaml`
- Applications path: `applications/`
- GitOps tool: OpenShift GitOps (ArgoCD)
- Chart storage: In-repo (not external Helm repositories)
