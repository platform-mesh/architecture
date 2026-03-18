# RFC 006: Provider Bootstrap Operator

| Status  | Proposed   |
|---------|------------|
| Author  | @mjudeikis |
| Created | 2026-03-18 |
| Updated | 2026-03-30 |

## Summary

A controller inside the existing `Platform-mesh` operator that automates provider bootstrap and lifecycle. It watches `Provider` and `ManagedProvider` CRDs in kcp, bootstraps workspaces, generates kubeconfigs, and optionally deploys provider workloads onto hosting clusters. Deployment is only for `ManagedProvider`, while `Provider` is bootstrap-only. This operator makes provider onboarding declarative and self-healing, replacing manual scripts and Helm installs. 

## Motivation

Provider onboarding today is manual (`make init`, shell scripts, manual Helm installs). This operator makes it declarative and self-healing.
In addition, we want to support providers as first-class citizens and exercise the model for building the platform itself.

## Proposal

### Two CRD Types

**`Provider`** — bootstrap-only. Creates the kcp workspace, applies API resources (ProviderMetadata, RBAC), generates a controller kubeconfig. The user deploys their own controller. 

**`ManagedProvider`** — extends `Provider` with OCM-based deployment of controller and portal charts onto a hosting cluster. This is for core providers, where we want to manage the full lifecycle. The operator handles bootstrapping and deployment, including updates and teardown.

Both live in `providers.platform-mesh.io/v1alpha1` and are stored in kcp. The operator runs on the hosting cluster and watches both kcp and the local cluster.

### Workspace and APIExport Layout

All providers live under `:root:system:providers`. This workspace exposes two APIExports:

- **`providers`** — exposes the `Provider` CRD. Bound recursively so any workspace in the platform can create `Provider` resources. This allows third-party users to register their own providers.
- **`managed-providers`** — exposes the `ManagedProvider` CRD. Bound recursively but intended for internal/platform use only. Keeps managed (core) providers separate from user-contributed ones.

Having two separate APIExports allows the platform to expose `Provider` broadly to platform users while keeping `ManagedProvider` restricted to platform operators. Each provider bootstrapped by the operator gets its own child workspace under `:root:system:providers:<provider-name>`.

```
:root
  └── :system
        └── :providers                          ← APIExports: providers, managed-providers
              ├── :wildwest                     ← bootstrapped provider workspace
              ├── :httpbin                      ← bootstrapped provider workspace
              └── ...
```

### Architecture

```
Operator (Platform-mesh operator, hosting cluster)
  ├── Watches kcp: Provider / ManagedProvider CRs
  ├── Watches hosting cluster: Jobs, Secrets, Deployments
  │
  ├── Provider reconciler:
  │     1. Create workspace hierarchy in kcp (root:system:providers:<provider-name>)
  │     2. Apply kcp resources (ProviderMetadata, RBAC, etc)
  │     3. Create ServiceAccount, generate controller kubeconfig Secret
  │
  └── ManagedProvider reconciler:
        1. Everything Provider does
        2. Resolve OCM component → extract Helm chart + image ref
        3. Install controller chart on hosting cluster
        4. Install portal chart on hosting cluster (optional)
```

### Authentication

Two supported modes:

1. **Service accounts** — admin kubeconfig Secret mounted with static token. Short-term solution or for testing. Long term solution requires kcp to support OIDC trust for workload identity federation.
2. **OIDC trust (keyless)** — Users OIDC federation configued in kcp for OIDC-based authentication.

Note: generated kubeconfigs must use the logical cluster name (e.g. `2cyb4oxml4sv8o3r`), not the human-readable workspace path, for obfuscation.

### Example: Provider (bootstrap-only)

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: Provider
metadata:
  name: wildwest
spec:
  workspacePath: "root:system:providers:wildwest" # optional, defaults to root:system:providers:<name>
  auth:
    adminKubeconfigSecretRef: # secret to be created 
      name: pm-admin-kubeconfig
      key: kubeconfig
  hostOverride: "https://frontproxy.platform-mesh-system:6443" # optional, host override for internal, external access.
```

### Example: ManagedProvider (bootstrap + deploy)

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ManagedProvider
metadata:
  name: wildwest
spec:
  workspacePath: "root:system:providers:wildwest"
  auth:
    adminKubeconfigSecretRef:
      name: pm-admin-kubeconfig
      key: kubeconfig
  hostOverride: "https://frontproxy.platform-mesh-system:6443"
  controller: # deployment spec for provider controller. Full struct TBD, but at minimum needs OCM component reference and values.
    ocm:
      componentName: github.com/platform-mesh/wildwest-controller
      version: "0.1.0"
      registry: ghcr.io/platform-mesh/ocm
      values:
        endpointSlice: wildwest.platform-mesh.io
  portal:
    ocm:
      componentName: github.com/platform-mesh/wildwest-portal
      version: "0.1.0"
      registry: ghcr.io/platform-mesh/ocm
```

### Reconciliation Phases

1. **Bootstrapping** — create workspace, apply kcp resources, generate kubeconfig Secret.
2. **Deploying** (ManagedProvider only) — resolve OCM components, install Helm charts.
3. **Ready** — continuous drift detection, re-bootstrap if kubeconfig deleted, re-deploy if chart drifts.

### Teardown

On CR deletion: uninstall Helm releases (if ManagedProvider), delete kubeconfig Secret, optionally clean up kcp workspace (`spec.cleanupOnDelete: true`).

### Portability

Existing providers (provider-quickstart, http-bin, resource-broker) should be portable to this model. Each provider ships its bootstrap resources as embedded manifests. The operator applies them generically — no provider-specific logic in the operator.

## Open Questions

1. Should OIDC keyless auth be the default long-term, with SA kubeconfigs as a stopgap?
2. Credential rotation strategy for long-lived SA tokens.
3. OCM value injection conventions (`image.repository`, `image.tag`, `kubeconfigSecret.name`).
