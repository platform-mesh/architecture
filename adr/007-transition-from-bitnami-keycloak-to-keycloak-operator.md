# ADR 007: Transition from Bitnami Keycloak Helm Chart to the Official Keycloak Operator

| Status  | Draft      |
|---------|------------|
| Date    | 2026-04-11 |

## Context and Problem Statement

Platform Mesh deploys Keycloak using the **Bitnami Keycloak Helm chart** as a subchart managed by the PlatformMesh operator. Bitnami has [ended free-tier security updates](https://github.com/bitnami/charts/issues/35164) (August 2025), which also led to the [proposed PostgreSQL migration](https://github.com/platform-mesh/architecture/pull/31) (ADR 006). Beyond licensing, the Bitnami chart wraps Keycloak in an opinionated layer that lags behind upstream releases, limits access to Keycloak-native configuration, and bundles an embedded PostgreSQL subchart we no longer need (ADR 006).

The Keycloak project provides an **official Kubernetes Operator** that manages Keycloak instances via a `Keycloak` CRD, giving direct access to upstream configuration without a third-party chart abstraction.

## Decision Drivers

* Reduce dependency on Bitnami charts (same driver as ADR 006).
* Direct access to upstream Keycloak configuration without Bitnami abstraction.
* Operator-managed lifecycle: rolling updates, health monitoring, and instance scaling via CRD.
* Consistent with the project's preference for operator-managed infrastructure (KCP Operator, CNPG for PostgreSQL, Keycloak Operator for identity).
* The transition must be **optional** — operators can point Platform Mesh at an external Keycloak instance.

## Considered Options

1. **Keep Bitnami Keycloak Helm chart** — status quo.
2. **Official Keycloak Operator** — Kubernetes operator with `Keycloak` and `KeycloakRealmImport` CRDs.

## Decision Outcome

Chosen option: **Official Keycloak Operator**, because it is maintained by the Keycloak project, provides CRD-based lifecycle management, tracks upstream releases directly, and reduces the dependency on Bitnami. Since no official Helm chart exists, a minimal local chart wraps the operator's published manifests (`keycloak-k8s-resources`) for Helm-based deployment.

### Consequences

* Good, because the dependency on Bitnami charts is further reduced.
* Good, because Keycloak configuration is managed via the upstream CRD without a third-party abstraction layer.
* Good, because the operator handles rolling updates, health checks, and instance scaling.
* Good, because realm imports can be managed declaratively via `KeycloakRealmImport` CRDs.
* Neutral, because Keycloak requires a pre-optimized image (`kc.sh build`) for production use — we will likely build and publish our own optimized Keycloak image rather than consuming the upstream image directly.
* Bad, because no official Helm chart exists — a local chart must be maintained to wrap the operator manifests.
* Bad, because advanced pod configuration (resources, imagePullPolicy) requires the `unsupported.podTemplate` escape hatch, which may change between operator versions.

## Deployment Options

| Approach | Pros | Cons |
|----------|------|------|
| **Local Helm chart** (chosen) | Parameterized, fits existing Helm/Flux/OCM workflow | Must track upstream manifest changes manually |
| **Raw kubectl apply / Kustomize** | Simplest, direct from upstream | No parameterization, not compatible with OCM-based release pipeline |
| **OLM / Operator Lifecycle Manager** | Automatic updates | Heavy dependency, not compatible with OCM-based release pipeline, primarily an OpenShift concept |

The local chart approach was chosen because it integrates with the existing Helm + FluxCD + OCM release pipeline while keeping the chart minimal and easy to update when the upstream operator manifests change.
