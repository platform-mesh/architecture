# ADR 006: Provider Bootstrap Operator

| Status  | Proposed   |
|---------|------------|
| Date    | 2026-04-08 |

> Full design in [RFC 006: Provider Bootstrap Operator](../rfc/006_provider-bootstrap-operator.md).

## Context

Provider onboarding is today entirely manual — shell scripts, `make init`, and hand-applied Helm installs. We need a declarative, self-healing model. The naive approach (a single CRD that handles both kcp-side bootstrap and runtime-side deployment) would allow a misconfigured kcp RBAC rule to trigger arbitrary workload deployments on the runtime cluster.

## Decision Drivers

* **Security boundary**: kcp-level resources must never be able to cause deployments into the runtime cluster. The threat model requires this to hold even if kcp RBAC is misconfigured.
* **Declarative lifecycle**: Provider onboarding must be expressible as Kubernetes resources — no shell scripts, no manual steps.
* **Self-healing**: The operator must continuously reconcile provider state and correct drift.
* **Portability**: Existing providers (provider-quickstart, http-bin, resource-broker) must be portable to this model with no provider-specific logic in the operator itself.
* **Separation of concerns**: kcp-side bootstrap (SA, RBAC, kubeconfig) and runtime-side orchestration (workspace creation, Helm deployment) should be independently reconcilable.

## Decision

Introduce two resources with two dedicated controllers inside the Platform-mesh operator:

- **`Provider`** (APIResourceSchema in kcp) — kcp-side bootstrap only: ServiceAccount, RBAC, and kubeconfig Secret within the provider workspace. No runtime-cluster effects.
- **`ManagedProvider`** (CRD on the runtime cluster) — orchestrates the full lifecycle: creates the kcp workspace, triggers `Provider` bootstrap, copies the kubeconfig Secret to the runtime namespace, and deploys provider workloads via OCM/Helm.

The security boundary is enforced structurally: a `Provider` resource in kcp is physically incapable of affecting the runtime cluster, regardless of kcp RBAC configuration.

## Considered Options

1. **Single runtime CRD** — one `ManagedProvider` resource handles everything. Simpler, but gives a kcp-level actor the ability to cause runtime-cluster deployments.
2. **Single kcp resource** — one `Provider` resource drives everything from kcp. Same security problem in reverse, and kcp resources cannot directly reach the runtime cluster anyway.
3. **Two resources on two layers** — chosen. Enforces the security boundary by construction.

## Consequences

- Good, because kcp RBAC misconfiguration cannot cause runtime-cluster deployments.
- Good, because provider onboarding becomes declarative and self-healing.
- Good, because the two controllers are independently reconcilable.
- Good, because the model is generic — any provider can be onboarded by creating a `ManagedProvider` CR, with no operator changes.
- Bad, because two resources increase the conceptual surface area for operators managing providers.
- Bad, because the `ManagedProvider` controller must coordinate across two clusters, increasing implementation complexity.
