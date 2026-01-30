# Shared GitHub Actions Workflows

This repository contains reusable workflows used by all Surfside Labs applications.

## Workflows

### `build-deploy.yml`

Automated build and deployment workflow for Kubernetes applications.

**Features:**
- Native ARM64 Docker builds (no emulation!)
- GitHub Container Registry (GHCR) image storage
- Automatic Kubernetes manifest generation from `.k8s/config.yaml`
- Deployment to K3s cluster via kubectl
- Dry-run mode for PR validation
- Docker layer caching for fast builds
- Health check validation

**Usage:**

In your application repository, create `.github/workflows/k8s-deploy.yml`:

```yaml
name: Build and Deploy to K3s

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: surfside-labs/.github/.github/workflows/build-deploy.yml@main
    with:
      app-name: my-app
      namespace: default
      dry-run: ${{ github.event_name == 'pull_request' }}
    secrets: inherit
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `app-name` | Yes | - | Application name (must match K8s resources) |
| `namespace` | No | `default` | Kubernetes namespace |
| `dry-run` | No | `false` | Dry run mode (builds but doesn't deploy) |
| `config-file` | No | `.k8s/config.yaml` | Path to K8s configuration file |

**Secrets:**

The workflow requires these organization secrets:

- `KUBECONFIG` - Base64-encoded kubeconfig for K3s cluster access
- `GITHUB_TOKEN` - Automatically provided by GitHub Actions

**Workflow Steps:**

1. **Checkout** - Clones repository
2. **Docker Build** - Builds ARM64 image with layer caching
3. **Push to GHCR** - Pushes image to GitHub Container Registry
4. **Generate Manifests** - Creates K8s manifests from config
5. **Create Secrets** - Sets up image pull secret
6. **Deploy** - Applies manifests to K3s cluster
7. **Status** - Shows deployment status and service info

**Dry Run Mode:**

When `dry-run: true` (automatically set for PRs):
- Builds Docker image
- Shows what would be deployed
- Does NOT push image or deploy to cluster
- Perfect for PR validation

**Architecture:**

```
┌─────────────────┐
│  Your App Repo  │
└────────┬────────┘
         │ triggers
         ▼
┌─────────────────────────┐
│  Shared Workflow        │
│  (this repository)      │
└────────┬────────────────┘
         │ runs on
         ▼
┌─────────────────────────┐
│  Self-Hosted Runner     │
│  (Raspberry Pi)         │
└────────┬────────────────┘
         │ deploys to
         ▼
┌─────────────────────────┐
│  K3s Cluster            │
│  (ARM64 native)         │
└─────────────────────────┘
```

## Configuration File Format

Your application must have a `.k8s/config.yaml` file:

```yaml
app:
  name: my-app
  namespace: default
  replicas: 1

container:
  port: 8080
  env:
    NODE_ENV: production
  healthCheck:
    enabled: true
    path: /health
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
    limits:
      memory: 1Gi
      cpu: 500m

service:
  type: LoadBalancer
  port: 80
  targetPort: 8080

storage:  # Optional
  - name: data
    size: 10Gi
    mountPath: /data

secrets:  # Optional
  - name: app-secrets
    optional: false
```

## Requirements

### Self-Hosted Runner

Must be installed on your Raspberry Pi with:
- Docker
- kubectl configured for K3s access
- Python 3.11+ with PyYAML and Jinja2
- ARM64 architecture

See `k8s-deployments` repository for runner setup instructions.

### Organization Secrets

Set up in GitHub organization settings:

1. **KUBECONFIG** - K3s cluster access
   ```bash
   kubectl config view --flatten --minify | base64 -w 0
   ```

2. Add to: `https://github.com/organizations/surfside-labs/settings/secrets/actions`

## Development

### Testing Workflow Changes

1. Create a test application
2. Use the workflow with `@your-branch` instead of `@main`
3. Trigger manually via workflow_dispatch
4. Verify behavior
5. Merge to main when ready

### Manifest Generation

The workflow includes a Python script that generates Kubernetes manifests from your config file. It supports:

- Deployments with custom resources and replicas
- Services (ClusterIP, NodePort, LoadBalancer)
- PersistentVolumeClaims
- Secret mounting
- Health checks (liveness/readiness probes)
- Multiple container ports
- Environment variables

## Troubleshooting

### Build Fails

**Issue:** Docker build fails

**Solution:** Check Dockerfile syntax and dependencies

### Deploy Fails

**Issue:** kubectl cannot connect

**Solution:** Verify KUBECONFIG secret is set correctly

**Issue:** Image pull error

**Solution:** Check repository is public or regcred is created

### Pod Not Starting

**Issue:** CrashLoopBackOff

**Solution:** Check application logs:
```bash
kubectl logs -n <namespace> -l app=<app-name>
```

## Contributing

Improvements to shared workflows are welcome! Please:

1. Test thoroughly with multiple applications
2. Maintain backwards compatibility
3. Update documentation
4. Consider security implications

## License

MIT
