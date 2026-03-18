# RFC 002: Provider Bootstrap and Lifecycle Operator

| Status  | Proposed                                                     |
|---------|--------------------------------------------------------------|
| Author  | @mjudeikis                                                   |
| Created | 2026-03-18                                                   |
| Updated | 2026-03-18                                                   |

## Summary

This RFC proposes a Kubernetes operator that automates the full lifecycle of Platform Mesh providers — from workspace bootstrap and kubeconfig generation to OCM-based deployment of provider controllers and portal UIs. Today, provider onboarding is a multi-step, manual process involving admin-level access and shell scripts. This operator replaces that with a declarative `Provider` custom resource.

## Motivation

- Eliminate manual `make init` / `kubectl ws` steps when onboarding a new provider.
- Standardize the bootstrap → deploy → teardown lifecycle across all providers.
- Enable GitOps workflows: a single `Provider` CR in a compute cluster is the desired-state declaration.
- Reduce blast radius by scoping admin kubeconfig usage to a short-lived init container.
- Leverage OCM as the single source of truth for provider artifact versions, aligning with the platform-wide OCM component model (see ADR 001).

## Out of Scope

- **Day 2 operations**: Monitoring, alerting, log aggregation, and operational runbooks for running providers are not covered by this RFC.
- **Provider updates and upgrades**: Automated rollout strategies (canary, blue-green), version compatibility checks, and coordinated upgrades across kcp resources and OCM component versions are deferred to a future RFC.

## Context and Problem Statement

Setting up a Platform Mesh provider today requires multiple manual phases (observed across `provider-quickstart`, `provider-kube-bind`, and `resource-broker`):

1. **Workspace creation** — An administrator creates the provider workspace hierarchy (`root:providers:<name>`) using an admin kubeconfig and `kubectl ws`.
2. **Bootstrap** — A CLI tool (`cmd/init`) applies kcp resources (APIResourceSchema, APIExport), provider resources (ProviderMetadata, ContentConfiguration, RBAC), and controller resources (ServiceAccount, token Secret, kubeconfig Secret).
3. **Kubeconfig generation** — The bootstrap tool polls for a ServiceAccount token, then synthesises a kubeconfig Secret pointing at the provider workspace, optionally rewriting the server URL through a front-proxy.
4. **Controller deployment** — A Helm chart is installed on the compute cluster, mounting the generated kubeconfig Secret and configuring the operator to watch an APIExportEndpointSlice.
5. **Portal deployment** — A second Helm chart deploys the provider's microfrontend UI (Angular/nginx, Luigi integration).

Each step depends on the previous one succeeding, admin credentials must be handled securely, and there is no reconciliation if any step drifts or fails partway.

### Problems

- **Operational toil**: Every new provider requires the same sequence executed correctly.
- **Security exposure**: Admin kubeconfig is used interactively and may linger on disk.
- **No drift detection**: If someone deletes the kubeconfig Secret or the deployment drifts, nothing self-heals.
- **No standardised deployment path**: Image tags, chart values, and kcp resource updates are done independently with no coordination.
- **Disconnected from OCM**: Provider deployments are managed outside the OCM component model, losing traceability between chart versions, image versions, and SBOMs.

## Proposal

### Custom Resource: `Provider`

Introduce a `Provider` CRD in the `providers.platform-mesh.io/v1alpha1` API group, deployed on the compute cluster where provider workloads run.

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: Provider
metadata:
  name: wildwest
  namespace: provider-cowboys
spec:
  # ── Bootstrap configuration ──
  bootstrap:
    # Admin kubeconfig used only by the init container.
    # The operator never mounts this into long-running pods.
    adminKubeconfigSecretRef:
      name: pm-admin-kubeconfig
      key: kubeconfig

    # Workspace path to create / bootstrap into.
    workspacePath: "root:providers:wildwest"

    # Optional: rewrite the server URL in the generated controller kubeconfig
    # so that in-cluster pods reach kcp through the front-proxy.
    hostOverride: "https://frontproxy-front-proxy.platform-mesh-system:6443"

    # Container image that runs the bootstrap logic (embeds kcp + provider YAML).
    image: ghcr.io/platform-mesh/provider-quickstart-init:v0.1.0

  # ── Controller deployment ──
  controller:
    ocm:
      # OCM root component name (as defined in ADR 001).
      # The operator resolves chart and image componentReferences from this.
      componentName: github.com/platform-mesh/wildwest-controller
      version: "0.1.0"
      # OCM registry to pull the component from.
      registry: ghcr.io/platform-mesh/ocm
      # Optional Helm value overrides applied on top of the chart
      # resolved from the OCM component.
      values:
        endpointSlice: wildwest.platform-mesh.io

  # ── Portal deployment (optional) ──
  portal:
    ocm:
      componentName: github.com/platform-mesh/wildwest-portal
      version: "0.1.0"
      registry: ghcr.io/platform-mesh/ocm
      values:
        ingress:
          host: wildwest.portal.example.com
```

### OCM Component Resolution

The operator uses the OCM SDK to resolve provider artifacts from the OCM registry. Each provider's root component follows the structure defined in ADR 001:

```
root-component (github.com/platform-mesh/wildwest-controller)
  ├── componentRef: chart (github.com/platform-mesh/helm-charts/wildwest-controller)
  │     └── resource: chart (type: helmChart)
  └── componentRef: image (github.com/platform-mesh/images/wildwest-controller)
        ├── resource: image (type: ociImage)
        ├── resource: sbom-cyclonedx
        └── resource: sbom-spdx
```

The operator resolves the root component, then:

1. **Extracts the Helm chart** from the `chart` component reference's `helmChart` resource.
2. **Extracts the image reference** from the `image` component reference's `ociImage` resource.
3. Injects the OCM-resolved image reference as a Helm value, overriding the chart's default `appVersion`. This ensures OCM is the authoritative source for image versions.
4. Merges any user-supplied `values` from the `Provider` CR.
5. Installs the chart using the Helm SDK.

This means the `Provider` CR never specifies image tags directly — they are determined by the OCM component version. Users only specify the root component name and version.

### Operator Reconciliation Loop

The operator watches `Provider` resources and drives each through a phased lifecycle:

```
                ┌──────────────────┐
                │   Provider CR    │
                │   created/updated│
                └────────┬─────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  Phase: Bootstrapping│
              │                      │
              │  Run init container  │
              │  (Job with admin     │
              │   kubeconfig)        │
              └──────────┬───────────┘
                         │ Job succeeds
                         │ kubeconfig Secret exists
                         ▼
              ┌──────────────────────┐
              │  Phase: Deploying    │
              │                      │
              │  Resolve OCM         │
              │  components          │
              │                      │
              │  Install controller  │
              │  chart from OCM      │
              │                      │
              │  Install portal      │
              │  chart from OCM      │
              └──────────┬───────────┘
                         │ Deployments healthy
                         ▼
              ┌──────────────────────┐
              │  Phase: Ready        │
              │                      │
              │  Continuous          │
              │  reconciliation      │
              │  (drift detection)   │
              └──────────────────────┘
```

#### Phase 1: Bootstrapping

The operator creates a Kubernetes **Job** in the provider's namespace:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: wildwest-bootstrap
  namespace: provider-cowboys
  ownerReferences:
    - apiVersion: providers.platform-mesh.io/v1alpha1
      kind: Provider
      name: wildwest
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      initContainers:
        - name: bootstrap
          image: ghcr.io/platform-mesh/provider-quickstart-init:v0.1.0
          args:
            - --kcp-kubeconfig=/etc/kcp/admin/kubeconfig
            - --host-override=https://frontproxy-front-proxy.platform-mesh-system:6443
            - --workspace-path=root:providers:wildwest
          volumeMounts:
            - name: admin-kubeconfig
              mountPath: /etc/kcp/admin
              readOnly: true
            - name: controller-kubeconfig
              mountPath: /etc/kcp/output
          env:
            - name: OUTPUT_SECRET_NAME
              value: wildwest-controller-kubeconfig
            - name: OUTPUT_SECRET_NAMESPACE
              value: provider-cowboys
      containers:
        - name: done
          image: busybox
          command: ["true"]
      volumes:
        - name: admin-kubeconfig
          secret:
            secretName: pm-admin-kubeconfig
        - name: controller-kubeconfig
          emptyDir: {}
```

The init container:

1. Connects to kcp using the admin kubeconfig.
2. Creates the workspace hierarchy if it does not exist.
3. Applies embedded kcp resources (APIResourceSchema, APIExport).
4. Applies provider resources (ProviderMetadata, ContentConfiguration, RBAC).
5. Creates a ServiceAccount, waits for its token Secret.
6. Generates a controller kubeconfig Secret on the **compute cluster** (not in kcp) containing the workspace-scoped token.

The operator watches the Job. On success it verifies the controller kubeconfig Secret exists and transitions to Phase 2. On failure it sets `status.phase: BootstrapFailed` with the Job's failure reason.

#### Phase 2: Deploying

The operator resolves OCM components and installs charts:

1. **Resolve OCM root component** — Pull the root component descriptor from the OCM registry.
2. **Follow component references** — Resolve the `chart` and `image` component references to obtain the Helm chart artifact and the OCI image reference.
3. **Controller chart** — Install the chart with the OCM-resolved image reference and the generated kubeconfig Secret name injected as Helm values (`kubeconfigSecret.name`, `image.repository`, `image.tag`).
4. **Portal chart** — Same resolution process, installed with user-supplied value overrides.

The operator uses the OCM Go SDK (`ocm.software/ocm`) for component resolution and the Helm Go SDK (`helm.sh/helm/v3`) for chart installation. It tracks the OCM component version, resolved chart digest, and deployment status in `Provider.status`.

#### Phase 3: Ready

Once all deployments have available replicas, the operator sets `status.phase: Ready`. It continues to reconcile on a configurable interval (default: 5 minutes) to detect and repair drift:

- If the kubeconfig Secret is deleted, re-run bootstrap Job.
- If a deployment is missing or has drifted, re-install from OCM.

### Provider Status

```yaml
status:
  phase: Ready  # Bootstrapping | BootstrapFailed | Deploying | DeployFailed | Ready | Deleting
  conditions:
    - type: Bootstrapped
      status: "True"
      lastTransitionTime: "2026-03-18T10:00:00Z"
      reason: JobSucceeded
      message: "Bootstrap job wildwest-bootstrap completed"
    - type: ControllerDeployed
      status: "True"
      lastTransitionTime: "2026-03-18T10:01:00Z"
      reason: OCMComponentDeployed
      message: "github.com/platform-mesh/wildwest-controller:0.1.0"
    - type: PortalDeployed
      status: "True"
      lastTransitionTime: "2026-03-18T10:01:05Z"
      reason: OCMComponentDeployed
      message: "github.com/platform-mesh/wildwest-portal:0.1.0"
  bootstrap:
    jobName: wildwest-bootstrap
    kubeconfigSecretName: wildwest-controller-kubeconfig
    workspacePath: "root:providers:wildwest"
  controller:
    ocmComponent: github.com/platform-mesh/wildwest-controller
    ocmVersion: "0.1.0"
    resolvedChartVersion: "0.1.0"
    resolvedImageRef: "ghcr.io/platform-mesh/wildwest-controller:v0.1.0"
  portal:
    ocmComponent: github.com/platform-mesh/wildwest-portal
    ocmVersion: "0.1.0"
    resolvedChartVersion: "0.1.0"
    resolvedImageRef: "ghcr.io/platform-mesh/wildwest-portal:v0.1.0"
```

### Teardown

When a `Provider` CR is deleted:

1. Finalizer blocks deletion.
2. Operator uninstalls deployed Helm releases (controller, portal).
3. Operator deletes the controller kubeconfig Secret.
4. Optionally (controlled by `spec.bootstrap.cleanupOnDelete: true`), the operator runs a cleanup Job that removes workspace resources from kcp.
5. Finalizer is removed, CR is garbage-collected.

### Security Considerations

| Concern | Mitigation |
|---------|-----------|
| Admin kubeconfig exposure | Mounted only in the short-lived bootstrap Job init container. Never mounted in long-running pods. |
| Admin kubeconfig at rest | Stored as a Kubernetes Secret. Cluster-level encryption-at-rest and RBAC apply. |
| Controller kubeconfig scope | Scoped to the provider workspace via ServiceAccount token. Cannot access other workspaces. |
| Operator RBAC | Requires `batch/v1 Jobs`, `v1 Secrets`, and Helm release management in the provider namespace. Does not require cluster-admin. |
| OCM registry access | Operator uses image pull secrets or workload identity to authenticate to the OCI registry hosting OCM components. |

### Init Container Contract

To keep the operator generic, the init container image must satisfy a simple contract:

| Input | Mechanism | Description |
|-------|-----------|-------------|
| Admin kubeconfig | Volume mount at `/etc/kcp/admin/kubeconfig` | Used to authenticate to kcp |
| Workspace path | `--workspace-path` flag | Target workspace to bootstrap |
| Host override | `--host-override` flag (optional) | Rewrite server URL for in-cluster access |
| Output secret name | `OUTPUT_SECRET_NAME` env var | Name of the kubeconfig Secret to create on the compute cluster |
| Output secret namespace | `OUTPUT_SECRET_NAMESPACE` env var | Namespace for the output Secret |

| Output | Mechanism | Description |
|--------|-----------|-------------|
| Controller kubeconfig Secret | Created on compute cluster | Contains workspace-scoped kubeconfig |
| Exit code 0 | Process exit | Signals success to the Job controller |

This contract allows each provider to ship its own init image with embedded bootstrap resources, while the operator remains provider-agnostic.

## Alternatives Considered

### Alternative 1: Shell scripts / Makefile (current state)

The existing `make init` + manual Helm install approach.

- **Pros**: Simple, no new operator to maintain.
- **Cons**: No self-healing, no drift detection, manual toil scales linearly with provider count, admin kubeconfig management is ad-hoc.

### Alternative 2: GitOps with FluxCD / ArgoCD

Use Flux HelmReleases and Kustomizations to manage provider deployments.

- **Pros**: Well-understood GitOps tooling, broad ecosystem.
- **Cons**: Bootstrap phase (workspace creation, kubeconfig generation) cannot be modelled as a HelmRelease — it requires imperative kcp API calls. Would need a separate mechanism (CronJob, Tekton pipeline) for bootstrap, splitting the lifecycle across two systems. No native OCM integration — would require custom tooling to resolve OCM components to Helm chart references.

### Alternative 3: Unified operator without init container

Embed all bootstrap logic directly in the operator rather than delegating to an init container.

- **Pros**: Single binary, simpler Job management.
- **Cons**: The operator would need to import every provider's bootstrap resources or use a plugin system. Init containers keep the operator generic and allow providers to evolve their bootstrap independently.

### Alternative 4: Direct Helm chart references (without OCM)

Reference Helm charts directly by OCI URL and version in the `Provider` CR, bypassing OCM.

- **Pros**: Simpler operator — no OCM SDK dependency, no component resolution step.
- **Cons**: Loses the version traceability that OCM provides (which chart version goes with which image version). Image tags must be specified separately, risking version skew. No SBOM association. Diverges from the platform-wide OCM component model established in ADR 001.

## Implementation Notes

- The operator should be built with `controller-runtime`, use the OCM Go SDK (`ocm.software/ocm`) for component resolution, and the Helm Go SDK (`helm.sh/helm/v3`) for chart installation.
- Bootstrap Job names should be deterministic (e.g. `<provider-name>-bootstrap`) to enable idempotent re-runs.
- The operator should watch for kubeconfig Secret deletion using an `Owns(&corev1.Secret{})` predicate.
- The `resource-broker` operator CRD (`Broker`) provides a useful reference for deployment spec patterns (image, replicas, resources, security context).
- OCM component resolution should be cached to avoid redundant registry pulls on every reconciliation loop.
- The operator should support configurable registry credentials via `imagePullSecrets` or service account annotations for workload identity.

## Open Questions

1. **Workspace cleanup on delete**: Should the operator remove the kcp workspace and its resources when a Provider CR is deleted, or leave them as orphans for manual cleanup? The `cleanupOnDelete` flag makes this opt-in, but the default behaviour needs agreement.
2. **Multi-cluster bootstrap**: If a provider needs to bootstrap into multiple kcp shards, should the `Provider` CR support a list of workspace targets, or should users create one `Provider` CR per shard?
3. **Credential rotation**: The generated controller kubeconfig uses a long-lived ServiceAccount token. Should the operator periodically rotate this token and restart the controller, or should we move to short-lived certificate-based credentials (as in `kcp-operator`)?
4. **OCM value injection convention**: Which Helm values should the operator auto-inject from OCM resolution (e.g. `image.repository`, `image.tag`, `kubeconfigSecret.name`) versus require the user to specify? A convention-over-configuration approach reduces boilerplate but may be surprising.
5. **OCM registry authentication**: Should the operator use a dedicated Secret for OCM registry credentials, reuse the namespace's default image pull secrets, or rely on workload identity?
