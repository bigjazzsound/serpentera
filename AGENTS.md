# AGENTS.md

## Project Structure

**Serpentera** is a Kubernetes cluster managed by FluxCD with GitOps. Apps are deployed via Helm charts through Flux kustomizations.

### Key Directories

- `apps/` - Application Helm chart configurations and values
- `clusters/prod/` - Production cluster kustomizations
- `clusters/test/` - Test cluster kustomizations
- `.sops.yaml` - SOPS encryption config for secrets

## Essential Commands

### Task Runner

Use `task` (Taskfile.yml) for most operations, NOT `mise` tasks:

```bash
task                    # Show all available tasks
task bootstrap          # Install required tools (kubectl, flux, etc.)
task bootstrap-flux     # Bootstrap FluxCD (requires CLUSTER var)
task sync              # Sync Flux with GitHub
task add-app           # Add new application
task add-infra         # Add infrastructure component
task add-ks            # Add kustomization
task reset-helmrelease # Reset failed Helm release
```

### Required Variables

Many tasks require environment variables:

- `CLUSTER=prod` or `CLUSTER=test`
- `BRANCH=main` (or current branch)
- `NAME=<app-name>`

Example: `CLUSTER=prod NAME=jellyfin task add-ks`

### FluxCD Operations

```bash
flux reconcile source git flux-system  # Manual sync
flux check                             # Verify cluster health
```

## App Development Workflow

### Adding New Apps

**For apps with official Helm charts:**

1. `task add-app NAME=myapp URL=<helm-repo-url>`
2. Edit `apps/myapp/values-config.yml` with Helm values
3. `task add-ks NAME=myapp ENV=prod` to create cluster kustomization
4. Commit and push - Flux will deploy automatically

**For apps without official Helm charts:**

1. `task add-app NAME=myapp` (omit URL to use bjw-s app-template default)
2. Edit `apps/myapp/values-config.yml` using bjw-s app-template structure (see `apps/mealie/` example)
3. Configure container image, service, ingress, and persistence in values-config.yml
4. `task add-ks NAME=myapp ENV=prod` to create cluster kustomization
5. Commit and push - Flux will deploy automatically

### bjw-s App Template Usage

When no official Helm chart exists, use the bjw-s app-template (`oci://ghcr.io/bjw-s-labs/helm/app-template`). Key configuration sections in `values-config.yml`:

- `controllers.<name>.containers.app` - Container image and environment
- `service.app` - Service configuration
- `ingress.app` - Ingress/routing configuration
- `persistence.<name>` - Storage configuration
- Schema validation available at: `https://raw.githubusercontent.com/bjw-s/helm-charts/main/charts/other/app-template/schemas/helmrelease-helm-v2.schema.json`

### App Structure (Standard Pattern)

Each app follows this structure:

```
apps/<name>/
├── kustomization.yml     # Kustomize config
├── kustomizeconfig.yml   # Kustomize name reference config
├── ns.yml               # Namespace
├── repo.yml             # Helm repository (HelmRepository or OCIRepository)
├── release.yml          # HelmRelease
├── values-config.yml    # Helm chart values
└── secrets.sops.yml     # Encrypted secrets (optional)
```

**Example - mealie app (bjw-s app-template):**

- `kustomization.yml` - Includes `secrets.sops.yml` in resources list
- `repo.yml` - OCIRepository pointing to `oci://ghcr.io/bjw-s-labs/helm/app-template`
- `release.yml` - Uses `chartRef` instead of `chart.spec` for OCI sources
- `secrets.sops.yml` - SOPS-encrypted secrets referenced via `envFrom` in values-config.yml

## SOPS/Secrets Management

### Encryption Setup

- Age key stored in `age.agekey` (not committed)
- SOPS config in `.sops.yaml` encrypts `*.sops.yml` files
- Secret setup: `task setup-sops` (creates sops-age secret in flux-system namespace)

### Working with Secrets

- Encrypt: `sops -e secrets.yaml > secrets.sops.yml`
- Edit encrypted: `sops secrets.sops.yml`
- All `*.sops.yml` files auto-decrypt during Flux deployment

## Architecture Notes

### Cluster Environments

- **prod**: Production cluster at `serpentera.cjf.lan` (multi-node)
- **test**: Test cluster (fewer apps, used for testing)

### FluxCD Structure

- Source: GitHub repository `bigjazzsound/serpentera`
- Flux watches `clusters/<env>/` for kustomizations
- Each `.yml` in `clusters/<env>/` defines a Flux kustomization pointing to `apps/<name>/`

### Dependencies

Apps can depend on infrastructure via `dependsOn` in cluster kustomizations (e.g., most apps depend on `ingress-nginx`).

## Common Issues

### Failed Helm Releases

Use `task reset-helmrelease NAME=<app>` to reset stuck deployments.

### SOPS Decryption Failures

Ensure `sops-age` secret exists in `flux-system` namespace: `task setup-sops`

### Bootstrap Requirements

- GitHub CLI (`gh`) authenticated
- `kubectl` configured for target cluster
- Age key available in `age.agekey`

## Tool Versions

- FluxCD: Managed by mise.toml (currently 2.7.5)
- Kubernetes: k0s v1.34.1+k0s.1 (see k0sctl.yaml)
- Renovate: Automated dependency updates via GitHub Actions

## File Patterns to Avoid

- Never commit `age.agekey`
- Never commit unencrypted secrets
- Use `*.sops.yml` suffix for encrypted files only
