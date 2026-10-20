# RFC 006: Provider Bootstrap Operator

| Status  | Proposed   |
|---------|------------|
| Author  | @mjudeikis |
| Created | 2026-03-18 |
| Updated | 2026-05-26 |

## Summary

Two controllers inside the Platform-mesh operator that automate provider bootstrap and lifecycle. `Provider` is a kcp-level resource that handles kcp-side bootstrap only (SA, RBAC, APIExport, kubeconfig). `ManagedProvider` is a runtime-cluster CRD that orchestrates the full lifecycle — creating the workspace, triggering Provider, copying secrets, and deploying workloads. This separation ensures that a kcp-level resource can never cause deployments into the runtime cluster.

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

`:root:providers:system` exposes the `providers` APIExport with recursive binding. Provider workspaces live directly under `:root:providers:<name>`.

```
:root
  └── :providers
        ├── :system                            ← system Provider resources + their dependants
        ├── :httpbin-234badf00d                ← bootstrapped provider workspace
        ├── :kube-bind-5badc0ffee              ← bootstrapped provider workspace (originating from a ManagedProvider)
        ├── :wildwest-0deadbeef1               ← bootstrapped provider workspace
        └── ...
```

### Architecture

```
Platform-mesh operator (runs on runtime/hosting cluster)
  │
  ├── Provider controller (watches kcp via VirtualWorkspace)
  │     Triggered by: Provider resource in a kcp workspace
  │     Actions (kcp-side only):
  │       1. Create provider workspace root:providers:<Provider.Name>-<Suffix>, where <Suffix> must be unique to that Provider
  │       2. Create admin ServiceAccount in provider workspace
  │       3. Generate kubeconfig Secret in target workspace
  │
  └── ManagedProvider controller (watches runtime cluster)
        Triggered by: ManagedProvider CR on runtime cluster
        Actions:
          1. Wait for Provider status.phase == Ready
          2. Copy kubeconfig Secret from Provider's workspace to runtime cluster
          3. Deploy OCM components inside runtimeDeployments into runtime cluster
```

### Authentication

Two supported modes:

1. **Service accounts** — admin kubeconfig Secret mounted with static token. Short-term solution or for testing. Long term solution requires kcp to support OIDC trust for workload identity federation.
2. **OIDC trust (keyless)** — Users OIDC federation configured in kcp for OIDC-based authentication.

Note: generated kubeconfigs must use the logical cluster name (e.g. `2cyb4oxml4sv8o3r`), not the human-readable workspace path, for obfuscation.

### Example: Provider (kcp-level, bootstrap-only)

Created in any workspace that has bound the `providers` APIExport.

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: Provider
metadata:
  name: wildwest
spec:
  providerKubeconfigSecret:
    name: my-secret-name
    namespace: my-secret-namespace
    key: kubeconfig
  hostOverride: "https://frontproxy.platform-mesh-system:8443" # optional
status:
  phase: Ready
```

### Example: ManagedProvider (runtime-level, full lifecycle)

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ManagedProvider
metadata:
  name: wildwest
  namespace: platform-mesh-system
spec:
  # Optional. Defaults to root:providers:system
  # Where to create the Provider resource.
  provider:
    path: root:orgs:org-a:bob
    name: wildwest

  # Optional. Defaults to <ManagedProvider.Namespace>/provider-<ManagedProvider.Name>-kubeconfig
  # Override destination of the admin kubeconfig to provider's workspace (e.g. root:orgs:org-a:bob).
  # This is stored inside the runtime cluster (runtimeKubeconfigSecret).
  providerKubeconfigSecret:
    name: my-secret-name
    namespace: my-secret-namespace
    key: kubeconfig # Optional. The service controller may need a different key. Defaults to "kubeconfig".

  # Optional.
  # Deploy the resources from runtimeDeployments into
  # a different cluster pointed to by the referrenced kubeconfig.
  runtimeKubeconfigSecret:
    name: my-service-cluster-kubeconfig
    key: kubeconfig

  runtimeDeployments:
  - ocm: # We may provide different component sources in the future. For now it's OCM->OCIRepository->HelmRelease.
      componentName: github.com/platform-mesh/wildwest-controller
      version: "0.1.0"
      registry: ghcr.io/platform-mesh/ocm
      values:
        # We expect the service controller to have its own workspace bootstrapping phase.
        # This service controller is then responsible for populating that workspace with all resources
        # necessary to provide the service (APIResourceSchemas, APIExports, ...)
        provider:
          bootstrap:
            kubeconfigRef:
              name: my-secret-name
              namespace: my-secret-namespace
  - ocm:
      componentName: github.com/platform-mesh/wildwest-portal
      version: "0.1.0"
      registry: ghcr.io/platform-mesh/ocm
```

### ManagedProvider Reconciliation Flow

```
ManagedProvider CR created (runtime cluster)
  │
  ▼
1. Create Provider resource at specified location
  │
  ▼
2. Wait for Provider status.phase == Ready
   (Provider controller creates workspace, SA, RBAC, kubeconfig in kcp)
  │
  ▼
3. Copy kubeconfig Secret from kcp workspace → runtime namespace
  │
  ▼
4. Deploy runtimeDeployments components
  │
  ▼
5. Ready — continuous drift detection
```

### Teardown

**ManagedProvider deletion**: uninstall runtime Helm releases, delete runtime kubeconfig Secret, optionally delete kcp workspace (`spec.cleanupOnDelete: true`).

**Provider deletion**: clean up SA, RBAC, APIExport, and kubeconfig Secret within the kcp workspace.

### Portability

Existing providers (provider-quickstart, http-bin, resource-broker) should be portable to this model. Each provider ships its bootstrap resources as embedded manifests. The operator applies them generically — no provider-specific logic in the operator.

## Open Questions

1. Should OIDC keyless auth be the default long-term, with SA kubeconfigs as a stopgap?
2. Credential rotation strategy for long-lived SA tokens.
