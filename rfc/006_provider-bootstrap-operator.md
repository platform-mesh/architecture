# RFC 006: Provider Bootstrap Operator

| Status  | Proposed   |
|---------|------------|
| Author  | @mjudeikis |
| Created | 2026-03-18 |
| Updated | 2026-03-30 |

## Summary

Two controllers inside the Platform-mesh operator that automate provider bootstrap and lifecycle. `ProviderSetup` is a kcp-level resource that handles kcp-side bootstrap only (SA, RBAC, APIExport, kubeconfig). `ManagedProvider` is a runtime-cluster CRD that orchestrates the full lifecycle — creating the workspace, triggering ProviderSetup, copying secrets, and deploying workloads. This separation ensures that a kcp-level resource can never cause deployments into the runtime cluster.

## Motivation

Provider onboarding today is manual (`make init`, shell scripts, manual Helm installs). This operator makes it declarative and self-healing.
In addition, we want to support providers as first-class citizens and exercise the model for building the platform itself.

## Security Rationale

It is problematic to offer a resource in kcp that can cause deployments into the runtime cluster. Misconfigured RBAC in kcp could allow deploying harmful code on the platform-mesh runtime. By splitting the resources:

- **kcp-level resources** (`Provider`) can only affect kcp — creating workspaces, RBAC, secrets within kcp.
- **Runtime-level resources** (`ManagedProvider`) control what gets deployed. Only operators with access to the runtime cluster can create these.

## Proposal

### Two Resources, Two Layers

**`Provider`** (APIResourceSchema in kcp) — kcp-level only. A controller in the Platform-mesh operator watches via VirtualWorkspace (`providers.platform-mesh.io`). It creates ServiceAccount, RBAC, and a kubeconfig Secret inside the provider workspace. No runtime-side effects. APIExport and providers bootstrap is out of scope for now.

**`ManagedProvider`** (CRD on runtime cluster) — orchestrates the full lifecycle from the runtime side. A controller in the Platform-mesh operator handles creation, deployment, and teardown.

### Workspace and APIExport Layout

All providers live under `:root:system:providers`. This workspace exposes the `providers` APIExport, bound recursively so any workspace can create `Provider` resources.

```
:root
  └── :system
        └── :providers                          ← APIExport: providers (exposes Provider)
              ├── :wildwest                     ← bootstrapped provider workspace
              ├── :httpbin                      ← bootstrapped provider workspace
              └── ...
```

### Architecture

```
Platform-mesh operator (runs on runtime/hosting cluster)
  │
  ├── Provider controller (watches kcp via VirtualWorkspace)
  │     Triggered by: Provider resource in a kcp workspace
  │     Actions (kcp-side only):
  │       1. Create ServiceAccount in provider workspace
  │       2. Create RBAC (roles, bindings)
  │       3. Generate kubeconfig Secret in provider workspace
  │     Note: requires permission claims with constraints
  │
  └── ManagedProvider controller (watches runtime cluster)
        Triggered by: ManagedProvider CR on runtime cluster
        Actions:
          1. Create provider workspace in kcp with correct type
          2. Create a Provider resource in the new workspace
          3. Wait for Provider status.phase == Ready
          4. Copy kubeconfig Secret from provider workspace to runtime namespace
          5. Resolve OCM component → extract Helm chart + image ref
          6. Deploy controller to target cluster (may be a different cluster)
          7. Deploy portal (optional)
```

### Authentication

Two supported modes:

1. **Service accounts** — admin kubeconfig Secret mounted with static token. Short-term solution or for testing. Long term solution requires kcp to support OIDC trust for workload identity federation.
2. **OIDC trust (keyless)** — Users OIDC federation configured in kcp for OIDC-based authentication.

Note: generated kubeconfigs must use the logical cluster name (e.g. `2cyb4oxml4sv8o3r`), not the human-readable workspace path, for obfuscation.

### Example: ProviderSetup (kcp-level, bootstrap-only)

Created in any workspace that has bound the `providers` APIExport.

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: Provider
metadata:
  name: wildwest
spec:
  hostOverride: "https://frontproxy.platform-mesh-system:6443" # optional
status:
  phase: Ready
  kubeconfigSecretRef:
    name: wildwest-kubeconfig  # Secret created in this workspace
```

### Example: ManagedProvider (runtime-level, full lifecycle)

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ManagedProvider
metadata:
  name: wildwest
  namespace: platform-mesh-system
spec:
  workspacePath: "root:system:providers:wildwest" # optional, defaults to root:system:providers:<name>
  controller:
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

### ManagedProvider Reconciliation Flow

```
ManagedProvider CR created (runtime cluster)
  │
  ▼
1. Create provider workspace in kcp (:root:system:providers:<name>)
  │
  ▼
2. Create Provider resource in the new workspace
  │
  ▼
3. Wait for Provider status.phase == Ready
   (Provider controller creates SA, RBAC, kubeconfig in kcp)
  │
  ▼
4. Copy kubeconfig Secret from kcp workspace → runtime namespace
  │
  ▼
5. Resolve OCM components, deploy controller + portal
  │
  ▼
6. Ready — continuous drift detection
```

### Teardown

**ManagedProvider deletion**: uninstall Helm releases, delete runtime kubeconfig Secret, optionally delete kcp workspace (`spec.cleanupOnDelete: true`).

**ProviderSetup deletion**: clean up SA, RBAC, APIExport, and kubeconfig Secret within the kcp workspace.

### Portability

Existing providers (provider-quickstart, http-bin, resource-broker) should be portable to this model. Each provider ships its bootstrap resources as embedded manifests. The operator applies them generically — no provider-specific logic in the operator.

## Open Questions

1. Should OIDC keyless auth be the default long-term, with SA kubeconfigs as a stopgap?
2. Credential rotation strategy for long-lived SA tokens.
3. OCM value injection conventions (`image.repository`, `image.tag`, `kubeconfigSecret.name`).
4. Exact permission claims and constraints needed for the ProviderSetup controller's VirtualWorkspace access.
5. Should the ManagedProvider support deploying to a cluster different from where the operator runs?
