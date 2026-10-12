# AGENTS.md

## Project Structure

**Serpentera** is a Kubernetes cluster managed by FluxCD with GitOps. Apps are deployed via Helm charts through Flux kustomizations.

### Key Directories

- `apps/` - Application Helm chart configurations and values
- `clusters/prod/` - Production cluster kustomizations
- `clusters/test/` - Test cluster kustomizations
- `mise.toml` - Task definitions and tool pins (there is no Taskfile; `task` will not work)

## Essential Commands

### Task Runner

There is no `Taskfile.yml` in this repo - the `task` binary will not find tasks.
Tasks live in `mise.toml` and run via `mise run <task>`.

```bash
mise tasks                 # List available tasks
mise run flux-sync         # Reconcile the Git source
mise run bootstrap-flux    # Bootstrap FluxCD
mise run add-app <name>    # Scaffold a new app directory
mise run add-repo          # Create repo.yml for a chart
mise run add-ks <name>     # Create a cluster kustomization
mise run add-helmrelease   # Create a HelmRelease
mise run build-ks <name>   # Dry-run build a kustomization
mise run existing-claim    # Create a PVC manifest
mise run setup-sops        # Create the sops-age secret
```

Tool versions are pinned in `mise.toml` (flux2, kubectl, age, gh, k0sctl).
`mise install` provisions them.

### FluxCD Operations

```bash
flux reconcile source git flux-system  # Manual sync
flux check                             # Verify cluster health
```

## App Development Workflow

### Adding New Apps

**For apps with official Helm charts:**

1. `mise run add-app myapp` (flag: `-u <chart-url>` for an official chart)
2. Edit `apps/myapp/values-config.yml` with Helm values
3. `mise run add-ks myapp` to create the cluster kustomization
4. Add `- name: envoy-gateway` to its `dependsOn` if the app has a route
5. Commit and push - Flux will deploy automatically

**For apps without official Helm charts:**

1. `mise run add-app myapp` (omits URL to use bjw-s app-template default)
2. Edit `apps/myapp/values-config.yml` using bjw-s app-template structure (see `apps/mealie/` example)
3. Configure container image, service, **route**, and persistence in values-config.yml
4. `mise run add-ks myapp` to create cluster kustomization
5. Add `- name: envoy-gateway` to the new kustomization's `dependsOn`
6. Commit and push - Flux will deploy automatically

### Routing: Gateway API only

**There is no ingress controller in this cluster.** ingress-nginx was removed in favour of
[Envoy Gateway](https://gateway.envoyproxy.io/) (see Architecture Notes). All external traffic
goes through a single `Gateway` named `external`. Do not add `ingress:` blocks or `Ingress`
resources - they will not be served.

Declare routes with `route:` in `values-config.yml`:

```yaml
route:
  app:
    parentRefs:
      - name: external
        namespace: gateway-system
        sectionName: https
    hostnames:
      - myapp.cjf.sh
    rules:
      - matches:
          - path:
              type: PathPrefix
              value: /
        backendRefs:
          - identifier: app
            port: http
```

Points to watch:

- `parentRefs[].sectionName` must be `https`. Use the `http` section only for redirect routes.
- `backendRefs[].identifier` references a `service.<id>` key in the same values file; `port` is a
  *named* port from that service, resolved to a number at render time.
- `route:` has **no `tls:` block**. TLS terminates once on the Gateway using the cluster-wide
  `*.cjf.sh` wildcard (`cert-manager/cjf-sh-tls`), so per-route TLS config is unnecessary.
- A chart that uses a *different* `route` schema (e.g. immich's `server.route`) must follow that
  chart's own conventions - see the app-specific notes below.

Apps on upstream charts that expose no `route` key (jellyfin, grafana) use a standalone
`httproute.yml` added to their `kustomization.yml` instead.

### bjw-s App Template Usage

When no official Helm chart exists, use the bjw-s app-template (`oci://ghcr.io/bjw-s-labs/helm/app-template`),
currently pinned at **5.3.0**. Key configuration sections in `values-config.yml`:

- `controllers.<name>.containers.app` - Container image and environment
- `service.app` - Service configuration
- `route.<name>` - HTTPRoute configuration (replaces `ingress.`)
- `persistence.<name>` - Storage configuration
- Schema validation available at: `https://raw.githubusercontent.com/bjw-s-labs/helm-charts/main/charts/other/app-template/schemas/helmrelease-helm-v2.schema.json`

Note the GitHub org is `bjw-s-labs`, not `bjw-s` (the latter's docs URLs 404).

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
- Secret setup: `mise run setup-sops` (creates sops-age secret in flux-system namespace)

### Working with Secrets

- Encrypt: `sops -e secrets.yaml > secrets.sops.yml`
- Edit encrypted: `sops secrets.sops.yml`
- All `*.sops.yml` files auto-decrypt during Flux deployment

## Architecture Notes

### Ingress: Envoy Gateway

External traffic is served by [Envoy Gateway](https://gateway.envoyproxy.io/), not an ingress
controller. Installed via the `apps/envoy-gateway/` app.

The app is deliberately split into two Flux Kustomizations, and **this is load-bearing**:

- `clusters/<env>/envoy-gateway.yml` → `./apps/envoy-gateway/controller` - the HelmRelease, which
  installs the Gateway API CRDs
- `clusters/<env>/gateway.yml` → `./apps/envoy-gateway/gateway` - GatewayClass, Gateway, policies;
  `dependsOn: envoy-gateway`

Pointing a single Kustomization at the parent `apps/envoy-gateway/` directory causes a bootstrap
deadlock: Flux server-side dry-runs the entire object set before applying anything, so `GatewayClass`
fails to resolve, nothing is applied, and the HelmRelease that installs the CRDs is never created.
`apps/envoy-gateway/` intentionally has no kustomization.yml so this cannot be reintroduced by
accident.

Key resources:

- `Gateway external` in namespace `gateway-system`, listeners `http` (80) and `https` (443)
- TLS terminates on the Gateway via `cert-manager/cjf-sh-tls` (wildcard `*.cjf.sh`). The Secret is
  cross-namespace, so a `ReferenceGrant` in `cert-manager` authorises it
- `apps/envoy-gateway/gateway/httproute-https-redirect.yml` - catch-all 308 to HTTPS on port 80,
  restoring the `ssl-redirect` behaviour ingress-nginx used to provide
- `EnvoyProxy external` - replica count and resources for the data plane

Gotchas learned the hard way:

- **`logging.level` keys must be real Envoy logger components.** `logging: {level: {envoy: info}}`
  renders `--component-log-level envoy:info`, and Envoy 1.39 rejects it during argv parsing with
  `error: invalid component specified 'envoy'`, exiting 1 instantly with no usable logs. Use
  `default:` (plus optionally registered components like `upstream`/`hcm`).
- **`extensionApis.enableBackend` is opt-in and defaults to false.** Routes whose `backendRef` is a
  `Backend` get a 500 direct response ("Backend is disabled in Envoy Gateway configuration") until
  it is set in `controller/values-config.yml`. Enabling it widens the API surface - anyone who can
  create `Backend` objects can point routes at arbitrary FQDNs.
- **Gateway API CRD bootstrap is order-sensitive** (see the two-Kustomization split above).

### DNS

All `cjf.sh` records are answered by blocky (`apps/blocky/values-config.yml`) from a single wildcard
mapping under `customDNS.mapping`, currently `cjf.sh: 192.168.200.16` (the Envoy Gateway LoadBalancer).
To change the gateway's address, edit that one line; blocky reloads and clients pick it up within
`caching.maxTime` (5m). Do **not** add per-app DNS records unless an app needs to bypass the gateway -
blocky does longest-match, so a specific hostname overrides the wildcard.

### Cluster Environments

- **prod**: Production cluster at `serpentera.cjf.lan` (multi-node)
- **test**: Test cluster (fewer apps, used for testing)

### Cluster Networking

- CNI is **kube-router**, with kube-proxy running. Multus is installed for secondary networks
  (`multus-lan`/`iot`/`dmz` NetworkAttachmentDefinitions, macvlan on `eno1.10`)
- MetalLB (L2 mode) provides LoadBalancer IPs from pools in `apps/metallb-pool/ipaddresspools/`
  (`k8s` = 192.168.200.11-192.168.203.254, `lan` = 10.0.0.200-250)
- The Gateway's `192.168.200.16` is **not pinned** via `spec.addresses`. If stability is needed,
  add `spec.addresses` to the Gateway so MetalLB cannot reassign it

### Apps with non-standard routing

- **omada-controller**: its management port 8043 speaks HTTPS with a self-signed cert
  (`CN=localhost`, SAN `DNS:Omada`, expires 2028-05-27). Replaced ingress-nginx's
  `backend-protocol: HTTPS` + `ssl-verify: false` with a `Backend` resource in
  `apps/omada-controller/backend.yml` using `tls.insecureSkipVerify: true`. The upstream
  `BackendTLSPolicy` has no `insecureSkipVerify` field, hence the `Backend` approach.

  **Caveat:** it uses a pod FQDN endpoint rather than a Service selector, so it bypasses
  Kubernetes Service-level policy and readiness.

- **immich**: uses the upstream immich chart with its own bjw-s `common` dependency, so its route
  schema is `server.route.main`, not top-level `route:`. It needs `extensionApis` unchanged, and
  **`backendRefs` must use `identifier:` with no `port:`** - `_route.tpl` only resolves a *string*
  port through the service lookup, so a literal `name:`/`port:` pair fails to render. See
  https://github.com/immich-app/immich-charts/issues/165

- **stash**: was installed directly via `helm install` outside Flux and was reverse engineered into
  `apps/stash/`. Chart was 5.0.1, moved to 5.3.0 to match the rest of the repo.

### Testing changes

`helm template` **must** be given API discovery when rendering routes, otherwise the bjw-s common
library falls back to `gateway.networking.k8s.io/v1alpha2` and you will chase a phantom diff:

```bash
helm template <release> oci://ghcr.io/bjw-s-labs/helm/app-template --version 5.3.0 \
  -n <ns> -f apps/<app>/values-config.yml \
  --api-versions "gateway.networking.k8s.io/v1/HTTPRoute"
```

Flux renders against a live cluster so it always sees the real CRDs; only offline/CI rendering is
affected. Verify route coverage by diffing rendered HTTPRoute hostnames against live Ingress
hostnames before removing anything.

### FluxCD Structure

- Source: GitHub repository `bigjazzsound/serpentera`
- Flux watches `clusters/<env>/` for kustomizations
- Each `.yml` in `clusters/<env>/` defines a Flux kustomization pointing to `apps/<name>/`

### Dependencies

Apps can depend on infrastructure via `dependsOn` in cluster kustomizations. **Every app with a
route must depend on `envoy-gateway`**, otherwise its HTTPRoute can be applied before the CRDs
exist and silently fail. The `gateway` Kustomization (data plane objects) additionally depends on
`certs` for the wildcard certificate.

## Common Issues

### Failed Helm Releases

Reset a stuck release by annotating it, e.g.:
`kubectl annotate helmrelease <name> -n flux-system fluxcd.io/force -='yes' --overwrite`

### SOPS Decryption Failures

Ensure `sops-age` secret exists in `flux-system` namespace: `mise run setup-sops`

### Bootstrap Requirements

- GitHub CLI (`gh`) authenticated
- `kubectl` configured for target cluster
- Age key available in `age.agekey`

### Gateway not becoming Programmed

`kubectl get gateway -n gateway-system external -o yaml` and read `status.conditions`. The
`Programmed=False (NoResources)` message "Envoy replicas unavailable" means the Envoy data plane
pods are not becoming ready - check the `envoy-gateway-system-external-*` deployment.

Reading those pod logs may require going around `kubectl logs`, which can fail with
`invalid component specified 'envoy'` when containerd has already garbage-collected the crashed
container. The kubelet keeps the logs on the node filesystem:

```bash
kubectl get --raw "/api/v1/nodes/<node>/proxy/logs/pods/<ns>_<pod>/<container>/0.log"
```

## Tool Versions

- FluxCD: Managed by `mise.toml` (flux2 2.9.1)
- kubectl: Managed by `mise.toml` (1.34.1)
- Kubernetes: k0s **v1.36.2+k0s** on nodes, containerd 2.3.2 (no k0sctl.yaml in this repo -
  cluster provisioning lives elsewhere)
- Envoy Gateway: pinned in `apps/envoy-gateway/controller/repo.yml` (v1.9.2, Envoy image distroless-v1.39.1)
- Renovate: Automated dependency updates via GitHub Actions

## File Patterns to Avoid

- Never commit `age.agekey`
- Never commit unencrypted secrets
- Use `*.sops.yml` suffix for encrypted files only
