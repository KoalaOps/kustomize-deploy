# Kustomize Deploy

High-level deployment action that handles both GitOps (ArgoCD) and direct kubectl deployments with automatic detection.

## Features

- 🔍 **Auto-detects GitOps** - Checks for ArgoCD management
- 📝 **Updates manifests** - Sets image tags, labels, and environment variables
- 🚀 **Dual mode** - GitOps commit or kubectl apply
- ⏳ **Wait for rollout** - Monitors deployment status
- 🎯 **Namespace management** - Creates namespaces as needed

## Usage

```yaml
- name: Deploy to Kubernetes
  uses: KoalaOps/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/production
    service_name: backend
    image: myregistry.io/backend:v1.2.3
    environment: production
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `working_directory` | Working directory for operations | ❌ | '.' |
| `overlay_dir` | Path to kustomize overlay directory | ✅ | - |
| `service_name` | Service name | ✅ | - |
| `image` | Full image with tag | ✅ | - |
| `environment` | Environment name | ✅ | - |
| `actor` | User deploying | ❌ | `${{ github.actor }}` |
| `run_id` | Run ID for tracking | ❌ | `${{ github.run_id }}` |
| `detect_gitops` | Auto-detect GitOps mode from manifests | ❌ | `true` |
| `force_mode` | Force deployment mode (gitops, kubectl, or auto) | ❌ | `auto` |
| `commit_message` | Commit message for GitOps | ❌ | auto |
| `create_namespace` | Create namespace if it does not exist | ❌ | `true` |
| `wait_timeout` | Timeout for waiting on deployments (seconds) | ❌ | `120` |
| `env_patches` | Environment file patches (JSON format) | ❌ | - |

## Outputs

| Output | Description |
|--------|-------------|
| `mode` | Deployment mode used (gitops or kubectl) |
| `namespace` | Kubernetes namespace |
| `deployment` | Primary deployment name |
| `managed_by` | Value of managed-by label if found |

## How It Works

### 1. Update Phase
- Uses kustomize-edit to set image tag
- Updates version label
- Adds deployment metadata (last-deployed-by, deployment-id, etc.)
- Patches environment variables in config files (if env_patches provided)

### 2. Inspection Phase
- Uses kustomize-inspect to extract namespace and workloads
- Gets primary deployment name
- Validates kustomization builds successfully

### 3. Detection Phase
```yaml
# Checks for ArgoCD management
app.kubernetes.io/managed-by: argocd
```

### 4. Deploy Phase

**GitOps Mode:**
- Commits changes to git
- Pushes to trigger ArgoCD sync

**Kubectl Mode:**
- Applies manifests to cluster
- Waits for rollout to complete

## Examples

### Auto-detect mode (recommended)
```yaml
- name: Deploy service
  uses: KoalaOps/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/${{ inputs.environment }}
    service_name: ${{ inputs.service }}
    image: ${{ inputs.registry }}/${{ inputs.service }}:${{ inputs.tag }}
    environment: ${{ inputs.environment }}
```

### Force GitOps mode
```yaml
- name: Deploy via GitOps
  uses: KoalaOps/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/production
    service_name: backend
    image: myregistry.io/backend:v1.2.3
    environment: production
    force_mode: gitops
    commit_message: "Deploy backend v1.2.3 to production"
```

### Force kubectl mode
```yaml
- name: Direct deployment
  uses: KoalaOps/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/staging
    service_name: frontend
    image: myregistry.io/frontend:latest
    environment: staging
    force_mode: kubectl
    wait_timeout: 300
```

### With namespace creation
```yaml
- name: Deploy to new namespace
  uses: KoalaOps/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/feature
    service_name: api
    image: myregistry.io/api:feature-123
    environment: feature-123
    create_namespace: true
```

### With environment variable patches
```yaml
- name: Deploy with env patches
  uses: KoalaOps/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/production
    service_name: backend
    image: myregistry.io/backend:v1.2.3
    environment: production
    env_patches: |
      {
        "container.env": {
          "SENTRY_RELEASE": "v1.2.3",
          "BUILD_ID": "${{ github.run_id }}"
        }
      }
```

## Prerequisites

### For GitOps mode
- Write access to the repository
- ArgoCD configured to watch the repository

### For kubectl mode
- Authenticated kubectl context
- Appropriate RBAC permissions
- Cluster access (use cloud-login action first)

## Error Handling

The action will fail if:
- Overlay directory doesn't exist
- Kustomization.yaml is invalid
- Git push fails (GitOps mode)
- Kubectl apply fails (kubectl mode)
- Deployment doesn't become ready within timeout

## Notes

- Always use with cloud-login action for kubectl mode
- GitOps mode requires repository write permissions
- Image format must be `registry/repository:tag`
- Supports both Deployment and Rollout resources