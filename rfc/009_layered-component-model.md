# RFC 009: Layered Component Model for Platform Mesh

| Status     | Proposed                                                          |
|------------|-------------------------------------------------------------------|
| Author     | @aaronschweig                                                     |
| Created    | 2026-06-09                                                        |
| Updated    | 2026-06-09                                                        |
| Supersedes | [RFC 008](https://github.com/platform-mesh/architecture/pull/28)  |

This RFC supersedes [RFC 008](https://github.com/platform-mesh/architecture/pull/28). It proposes a strictly layered component model where each layer **adds** a capability, together with a sharper distinction between *optional* and *replaceable* components. The seven configuration matrix in RFC 008 captures the same design space, but a layered framing makes the dependencies and the support promise more legible.


## A Layered, Additive Model

Describe Platform Mesh as a stack of layers where each layer **adds a capability** on top of the one below it:

| Layer | Capability added | Components (illustrative) |
|-------|------------------|---------------------------|
| L1 — Core | Workspace and account lifecycle on top of kcp | `pm-operator`, `account-operator`, kcp, GitOps engine (Flux or ArgoCD) |
| L2 — Authentication | Federated identity for platform users | OIDC issuer (Keycloak by default, or any other OIDC provider) |
| L3 — Authorization | Fine-grained access control | OpenFGA + ReBAC webhook |
| L4 — Portal & Extensibility | Interactive UI, GraphQL API surface, extensions | OpenMFP, GraphQL Gateway, Extension Manager |

The layers are strictly ordered: an outer layer depends on every inner layer it sits on. A deployment may stop at any layer, but it cannot skip a layer in the middle and still claim the capability of the outer one. For example, a Portal deployment without an authentication layer is not a supported shape; the Portal's value depends on knowing who the user is.

This framing has two consequences worth stating directly:

1. **Dependency direction is explicit.** Outer layers depend on inner ones; inner layers never depend on outer ones. This rules out, by construction, the kind of accidental coupling that makes a component hard to disable.
2. **The supported configuration space is bounded.** A stack of N layers yields N supported deployment shapes (L1, L1+L2, L1+L2+L3, …), not the 2^N combinatorial matrix that arises from treating every component as independently optional.

> [!IMPORTANT]
> **Define the L1 capability contract.** L1 alone must be a "working Platform Mesh": it must be possible to deploy and operate the platform with only the core layer present. This requires an explicit list of (a) the capabilities L1 *does* provide on its own, and (b) the capabilities that are deliberately unavailable until L2 / L3 / L4 are added. Without this contract, "stop at L1" is not a credible offer.

> [!NOTE]
> **L2 — OIDC is the interface, Keycloak is an implementation.** Anything in Platform Mesh that today calls into Keycloak via a Keycloak-specific path *and* has an OIDC equivalent should be using the OIDC standard instead. Keycloak is the default OIDC provider shipped with Platform Mesh; it is not a fixed dependency. Any conformant OIDC provider must be a viable substitute without code changes in the rest of the stack.

> [!IMPORTANT]
> **L3 has no equivalent open standard.** For authentication, OIDC abstracts away the provider. For authorization there is no comparable industry standard to point at. The intent is to use the Kubernetes authorization primitives (the authz webhook protocol, RBAC resources) where possible, but capabilities differ meaningfully between a ReBAC system like OpenFGA and a static system like plain Kubernetes RBAC. "Replaceable" at L3 therefore cannot mean "drop-in equivalent" the way it can at L2. The promise here needs further thought.

## Optional vs. Replaceable

RFC 008 uses *optional* and *replaceable* somewhat interchangeably. The distinction matters because it expresses a different **promise** from Platform Mesh to its administrators, and a different **maintenance cost** for the project:

- **Optional:** the layer can be omitted. The platform still works, but the capability that layer provides is unavailable, and outer layers that depend on it are also unavailable. Administrators accept the trade-off knowingly. Platform Mesh promises: *"It runs without this. Some features will be missing or degraded."*

- **Replaceable:** the layer can be swapped for a conforming alternative implementation, and the administrator gets the **same capability with no functional drawback**. Outer layers continue to work unchanged. Platform Mesh promises: *"Both lines of usage are supported as first-class."*

Replaceability is significantly more expensive than optionality. Every replaceable component multiplies the validated combinations the project must test, document, and maintain. A combinatorial matrix of replaceable layers could grow unmaintainable rather quickly.

The proposal is therefore to be deliberate about which layers are which:

| Layer | Optional? | Replaceable? | Notes |
|-------|-----------|--------------|-------|
| L1 — Core | No | No | L1 is the minimum that defines a Platform Mesh deployment. |
| L2 — Authentication | Partially — see below | Yes, via OIDC | OIDC is a stable open standard; replacement is bounded. |
| L3 — Authorization | Yes (falls back to RBAC) | Open question — see L3 note above | Replaceability requires a defined authz interface. |
| L4 — Portal | Yes | Replaceable only at lower layer-support tiers | Replacement portals (Backstage, custom) are not drop-in equivalents. |

> [!NOTE]
> **L2 optionality.** Some form of identity is always required — kcp cannot answer "who is this request from?" without it. What is optional is the **managed, multi-tenant authentication experience** that Platform Mesh provides on top by default (Keycloak, per-organization realms, Dynamic Client Registration, federated invites). An administrator may drop that layer and bring their own identity arrangement instead. For this to be a real choice, the PlatformMesh resource must expose enough configuration to wire a non-managed authentication setup end-to-end (static OIDC client config, issuer URL, scopes, audience) without requiring the managed components to be present.

Aligning replaceability with **open standards** keeps the support surface finite. For L2 this is straightforward: OIDC is the interface, and any conformant issuer is replaceable.

## Per-Layer Detail

This section gets concrete about what each layer ships, what the user gets when only that layer (and the layers below it) are deployed.

### Layer 1: Core

The smallest possible Platform Mesh: a managed Platform Mesh instance with the account model, API-first, the lowest possible footprint and the lowest possible set of capabilities.

**Components**

- `pm-operator`: installation and lifecycle of the Platform Mesh instance and all of its components.
- `account-operator`: the managed account model inside kcp; provisions account workspaces and maintains the account hierarchy.
- `kcp`

**Capabilities the layer provides**

- A managed Platform Mesh instance, deployable and operable on its own.
- The account model: organizations and accounts, mapped to kcp workspaces.
- API-first access to the platform via `APIExport`s, `APIBinding`s, etc, and the kcp workspace APIs are usable directly via kubectl
- Single-tenant identity for the instance via an **externally-configured OIDC issuer**. The administrator points the `PlatformMesh` resource at an OIDC provider they already operate

### Layer 2: Authentication

L2 adds a managed multi-tenant authentication experience on top of L1. The administrator no longer brings their own IdP, instead Platform Mesh ships and runs one, and exposes its configuration as kcp resources inside the respective workspace.

**Components**

- `Keycloak:` the IdP shipped at L2. Picked specifically because the managed multi-tenant experience (per-organization realms, fully isolated issuers). There might be other IdPs out there that support similiar feature and adapters can be contributed by the community, but official support is only done for Keycloak at the moment.
- `security-operator` (IDP-automation responsibilities): provisions per-organization Keycloak realms, registers OIDC clients via RFC 7591 Dynamic Client Registration, and reconciles the per-workspace configurations resources into Keycloak.

**Capabilities the layer provides**

- A managed multi-tenant IdP, one Keycloak realm per organization, isolated from every other organization's identity domain.
- Automated OIDC client registration via RFC 7591, so clients are provisioned declaratively.
- A KRM-based configuration surface for identity. Anything an organization needs to configure about its IdP — upstream federation, client metadata, allowed audiences, is exposed as a resource inside the organization's workspace. End users never log in to Keycloak directly.

> [!IMPORTANT]
> **Gaps that block standalone L2 today.** The "everything is a resource" promise above is not yet fully delivered. There are configuration knobs that still require Keycloak admin access. As part of making L2 a viable standalone layer, these gaps need to be identified and closed.

**Replaceability**

L2 has two sub-capabilities with different replaceability stories:

- **Per-organization isolation (per-realm IdPs).** Not replaceable today. If an administrator wants this guarantee, they take Keycloak or provide a community contributed adapter for a different IdP.
- **OIDC client registration via RFC 7591.** Replaceable. RFC 7591 is an open standard, supported by multiple OIDC providers, and any conformant one can fill this slot.

An administrator who is willing to accept the drawback of losing per-org isolation can swap Keycloak for another RFC-7591-supporting provider and still keep dynamic client registration. They get a partial L2: managed client lifecycle, but a single shared IdP rather than per-organization isolation.

### Layer 3: Authorization

L3 adds fine-grained, relationship-based authorization on top of L1+L2. Once an identity is established by L2, L3 decides what that identity is allowed to do — at the granularity of individual resources and along the account hierarchy.

**Components**

- `iam-service`: assigns roles to users on specific resource contexts via a UI.
- `security-operator` (OpenFGA-automation responsibilities): establishes account entities in the Authorization Backend (OpenFGA).
- `OpenFGA`: the relationship tuple store.
- ReBAC webhook: the Kubernetes authorization webhook that is part of the Authorization Chain of kubernetes. It translates `SubjectAccessReviews` into queries against OpenFGA and returns allow/deny decisions.

**Capabilities the layer provides**

- Fine-grained, relationship-based authorization for every kcp API request. Permissions are evaluated against tuples in OpenFGA, expressing relationships between users, roles, and resources.
- Inheritance along the account hierarchy. Permissions granted at an organization or parent account propagate down to child accounts and the resources within them, following the relationships established in OpenFGA.

**Replaceability**

L3's replaceability story is deliberately left open in this RFC. There is no equivalent open standard for authorization the way OIDC fills that role for L2 (see the [L3 callout](#a-layered-additive-model) above).

> [!IMPORTANT]
> The scope of this RFC is to establish the layered approach as an alternative to RFC 008's pick-and-choose matrix; settling the L3 replacement interface is a separate, larger decision.

## Diagrams Showing What Each Layer Affects
**TODO — add per-layer diagrams.**
