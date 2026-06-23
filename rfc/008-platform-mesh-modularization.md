# RFC 008: A Modular and Layered Framework for Platform Mesh

| Status       | Proposed                   |
|--------------|----------------------------|
| Authors      | @perseus985, @aaronschweig |
| Created      | 2026-04-01                 |
| Updated      | 2026-06-16                 |

## Summary

This RFC restructures Platform Mesh as a set of composable **modules**. Each
module delivers a capability (a value stream that contributes to the shared
product "Platform Mesh") through a defined interface. The modules are
organized into a **layered default composition**:

> **L1 Core → L2 Authentication → L3 Authorization → L4 Portal**

The layers are the officially tested and supported compositions of Platform
Mesh. Beyond the layers, modules can be composed individually; operators can
assemble "their own" Platform Mesh (for example, Portal without the ReBAC
authorization engine, falling back to Kubernetes RBAC), and each such
composition is classified by a support tier.

Underneath the layers runs a single conviction: **in everything we do, we
believe in the individual extensibility of Platform Mesh.** This makes Platform
Mesh two things at once:

- **A framework**: a set of standards-based **interfaces**, one per concern
  (identity, authorization, portal, deployment, providers, observability). The
  interface, not the implementation, is the contract.

- **A product**: a curated, fully integrated default stack that implements
  those interfaces out of the box: **Keycloak, OpenFGA, and OpenMFP**,
  the supported "all-in-one" Platform Mesh.

The default stack is what most operators run, and it stays first-class. But
where an operator already runs their own system (their own Keycloak, Zitadel,
or Entra ID instead of the bundled IdP; Headlamp or no portal at all instead of
OpenMFP; their own GitOps engine), the promise
is: **use the interface, and where your system does not already speak it, write
an adapter.** The layers are the tested defaults; the interfaces are how you
make Platform Mesh yours (see [Interfaces](#interfaces)).

## Motivation

Platform Mesh today bundles a specific set of technology choices:

| Concern         | Current implementation    |
|-----------------|---------------------------|
| Identity / OIDC | Keycloak                  |
| Authorization   | OpenFGA (ReBAC)           |
| Portal / UI     | OpenMFP + GraphQL Gateway |

kcp — the L1 Core control plane — is the non-swappable substrate beneath all of
these, so it is not a "choice" in this table. These choices are reasonable
defaults. However, treating them as hard,
non-negotiable dependencies creates a significant adoption barrier for
organizations that:

- Already operate an enterprise identity provider (e.g. Active Directory
  Federation Services, Okta, Ping Identity)
- Have existing authorization infrastructure they are required to use
- Want to use an alternative portal framework (e.g. Headlamp)
- Want an API-only or reduced-footprint deployment, where a UI or other
  components are unnecessary overhead
- Do not need multiple organizations or nested accounts, and want to reduce
  complexity and resource consumption accordingly

A modular model addresses these barriers, and it does more than that:

1. **Dependency direction becomes explicit.** Outer layers depend on inner
   ones; inner layers never depend on outer ones. This rules out, by
   construction, the kind of accidental coupling that makes a component hard
   to disable or replace.
2. **The support promise stays bounded.** The *tested* set stays small: the N
   layered shapes are first-class, and any further compositions are explicitly
   named and tier-classified, not the implicitly-promised 2^N matrix that arises
   from treating every component as independently optional.
3. **The ecosystem can contribute capabilities.** With defined module
   interfaces, providers and extensions become capabilities that third
   parties can build and contribute to the shared product, the foundation
   for a marketplace of capabilities on top of Platform Mesh.

### Security Rationale

In security-hardened and compliance-heavy environments, fewer running
components means a smaller attack surface:

- **OpenFGA** is called on every kcp API request via webhook. In deployments
  where ReBAC is not required, this is unnecessary overhead and an additional
  availability dependency on the critical API path.
- **Keycloak** requires a database, ongoing CVE maintenance, and dedicated
  operational expertise. Organizations already running an enterprise IdP
  should not be required to operate a second one.
- **OpenMFP and the GraphQL Gateway** expose additional network endpoints.
  In non-interactive or API-only deployments, these surfaces are unnecessary.

Making modules omittable is therefore both an operational and a security
improvement.

## The Model: Modules, Layers, Composition

**Vocabulary.** Four words, used precisely (full definitions in the
[Glossary](#glossary)):

- a **capability** is one unit of value (identity, authorization, a portal, …) —
  also called a **module**; the two are the same thing in this RFC.
- each module exposes its capability through an **interface**: its contract, an
  open standard where one exists.
- a **layer** (**L1–L4**; the *L* is for *layer*) is a tested *bundle* of modules.
- the **framework** is the set of all those interfaces; the **product**
  (Platform Mesh) is the default set of modules that implement them.

### Modules

A **module** is the unit of composition, one capability. Each module:

- exposes its capability through a defined interface (an open standard where
  one exists),
- declares which other modules it depends on, and
- specifies its behavior when an optional dependency is absent.

Platform Mesh ships a default implementation for every module, but the
interface, not the implementation, is the contract.

### Layers

The **layers** are the precomposed, officially tested module sets. They are
strictly ordered: an outer layer depends on the layers beneath it, never the
other way around.

| Layer | Capability added | Default modules |
|-------|------------------|-----------------|
| **L1 Core** | Workspace and account lifecycle on top of kcp, API-first access, provider bootstrap | kcp, `pm-operator`, `account-operator`, `security-operator` (baseline), provider bootstrap mechanism |
| **L2 Authentication** | Managed, multi-tenant identity for platform users | Keycloak, `security-operator` (IDP automation) |
| **L3 Authorization** | Fine-grained, relationship-based access control | OpenFGA, ReBAC authorization webhook, `iam-service`, `security-operator` (FGA automation) |
| **L4 Portal** | Interactive UI, GraphQL API surface, extensions, marketplace | OpenMFP Portal, GraphQL Gateway (`kubernetes-graphql-gateway`), `virtual-workspaces`, `extension-manager-operator`, Portal UI / IAM UI / Marketplace UI |

Two placement notes (rationale to be captured in ADRs): the **provider-bootstrap
mechanism** belongs to L1, and the **GitOps engine is not a layer**: it is a
deployment interface — Platform Mesh bundles no engine and supports several
(Flux, ArgoCD, plain Helm). The **`security-operator`** is one
component whose subroutines activate per deployed layer: baseline RBAC in L1,
identity automation in L2, OpenFGA automation in L3.

### Supported Compositions

The **layered shapes** are the first-class, fully E2E-tested compositions
(Tier 1). Beyond them, **module compositions** assemble layers
non-contiguously; they are supported, explicitly named, and classified by
[support tier](#support-tiers) rather than implicitly promised. The seven
configuration variants of the original RFC 008 all remain reachable; they map
onto these shapes, shown in the *Former variant* column.

| Shape / composition | Layers | Description | Tier | Former variant |
|---|---|---|---|---|
| **S1** | L1 | Minimal, API-only; externally configured OIDC issuer; Kubernetes RBAC | Tier 1 | 7 (w/o Portal + OpenFGA + Keycloak) |
| **S2** | L1+L2 | API-only with managed multi-tenant IdP | Tier 1 | 6 (w/o Portal + OpenFGA) |
| **S3** | L1+L2+L3 | API-only with managed IdP and ReBAC authorization | Tier 1 | 4 (w/o Portal) |
| **S4** | L1+L2+L3+L4 | Full product, the out-of-the-box experience | Tier 1 | n/a |
| L1+L2+L4 | L1+L2+L4 | Portal with managed IdP, no ReBAC; authorization falls back to Kubernetes RBAC | Tier 2 | 1 (w/o OpenFGA) |
| L1+L3+L4 | L1+L3+L4 | Portal with external IdP and ReBAC | Tier 2 | 3 (w/o Keycloak) |
| L1+L4 | L1+L4 | Portal with external IdP, Kubernetes RBAC | Tier 2 | 2 (w/o OpenFGA + Keycloak) |
| L1+L3 | L1+L3 | External IdP with ReBAC authorization | Tier 2 | 5 (w/o Portal + Keycloak) |

Tier 1 shapes are covered by the full E2E test suite on every release; Tier 2
compositions are actively tested in CI with documented configuration.

A composition can omit a layer in the middle only because every module
defines its behavior when an optional dependency is absent (e.g. L3 absent →
RBAC fallback; L2 absent → externally configured OIDC issuer from L1). What
is *not* possible is omitting a capability that an outer module strictly
requires: the Portal's value depends on knowing who the user is, so some
form of identity, managed (L2) or external (L1's OIDC issuer configuration),
is always required beneath L4. **L1 is the floor:** L2–L4 are omittable, but a
deployment without L1 (the kcp control plane and account model) is plain kcp,
not Platform Mesh.

## Optional vs. Replaceable

*Optional* and *replaceable* are different promises from Platform Mesh to its
administrators, with different maintenance costs for the project:

- **Optional:** the module can be omitted. The platform still works, but the
  capability that module provides is unavailable, and modules that strictly
  depend on it are also unavailable or degraded. Administrators accept the
  trade-off knowingly. Platform Mesh promises: *"It runs without this. Some
  features will be missing or degraded."*

- **Replaceable:** the module can be swapped for a conforming alternative
  implementation, and the administrator gets the **same capability with no
  functional drawback**. Dependent modules continue to work unchanged.
  Platform Mesh promises: *"Both lines of usage are supported as
  first-class."*

Replaceability is significantly more expensive than optionality: every
replaceable module multiplies the validated combinations the project must
test, document, and maintain. Aligning replaceability with **open standards**
keeps the support surface finite. The standard differs in kind per concern:
for authentication the interface is OIDC + RFC 7591 DCR, and any conformant,
DCR-capable issuer can fill the slot directly; for authorization the interface
is the Kubernetes authorization webhook (`SubjectAccessReview`), itself a
standard, while the *relationship model* behind it (ReBAC) has no cross-vendor
standard yet, so OpenFGA is treated as a swappable **engine** behind that
webhook rather than a drop-in-replaceable standard.

| Layer | Optional? | Replaceable? | Notes |
|-------|-----------|--------------|-------|
| L1 Core | No | No | L1 is the minimum that defines a Platform Mesh deployment. |
| L2 Authentication | Partially, see below | Yes, via OIDC + DCR | DCR-capable IdPs swap directly; non-DCR IdPs (Zitadel, Entra ID) via a vendor-API adapter. |
| L3 Authorization | Yes, falls back to managed Kubernetes RBAC | Engine-swappable via the authz webhook | The `SubjectAccessReview` interface is defined; OpenFGA is the swappable ReBAC engine. A portable relationship-model standard is the open part. |
| L4 Portal | Yes | Yes, via the Portal interface | Alternative portals (Headlamp, custom) consume the same APIs via an adapter; not drop-in identical to OpenMFP. |

> [!NOTE]
> **L2 optionality.** Some form of identity is always required: kcp cannot
> answer "who is this request from?" without it. What is optional is the
> **managed, multi-tenant authentication experience** that Platform Mesh
> provides by default (Keycloak, per-organization realms, Dynamic Client
> Registration, federated invites). An administrator may drop that layer and
> bring their own identity arrangement instead. For this to be a real choice,
> the PlatformMesh resource must expose enough configuration to wire a
> non-managed authentication setup end-to-end (static OIDC client
> configuration, issuer URL, scopes, audience) without requiring the managed
> components to be present (see the [L1 contract](#l1-core)).

## Interfaces

> *In everything we do, we believe in the individual extensibility of
> Platform Mesh.*

Modularity (the previous sections) answers *"which capabilities run?"*
Extensibility answers *"whose implementation fills each slot?"* They are the
same mechanism seen from two sides: every module exposes its capability through
a defined **interface**: an open standard wherever one exists, and an operator
may connect their own system to that interface instead of using the default.

This section is the **map of extension points**: the surface an operator
targets to bring their own implementation. Each is identified by its **open
standard** (OIDC + DCR for identity, the Kubernetes authorization webhook for
authorization, GraphQL for the portal, OTLP for observability), not by a
Platform-Mesh-specific component name; the precise contract per interface lives
in its ADR. *Capability / API extension*, adding your own APIs, is a separate
mechanism (providers wired to kcp,
[RFC 004](004_core-platform-extendability.md) /
[RFC 006](006_provider-bootstrap-operator.md)); the interfaces here are the
platform's infrastructure contracts. The default stack (Keycloak, OpenFGA,
OpenMFP) is one supported set of implementations of them; an operator who
wants a different implementation targets the interface and supplies an
[adapter](#adapters) where their system does not already speak it.

| Interface | Layer | Open standard / protocol | Default implementation | Bring your own (via adapter) |
|---|---|---|---|---|
| **Identity** | L1 / L2 | OIDC + RFC 7591 DCR | Keycloak (managed mode) | DCR-capable IdPs (Okta, Auth0, PingFederate) directly; non-DCR IdPs (Zitadel, Entra ID) via a vendor adapter |
| **Authorization** | L3 | Kubernetes authorization webhook (`SubjectAccessReview`) | OpenFGA, ReBAC engine | Any engine behind the webhook; managed Kubernetes RBAC fallback. Relationship-model portability is the open part |
| **Portal** | L4 | **GraphQL** over Platform Mesh resources (served by `kubernetes-graphql-gateway`); `ContentConfiguration` + microfrontend = OpenMFP composition, not a standard ([RFC 001](001-api-providers-and-ui-discovery.md)) | OpenMFP | Headlamp (plugin/adapter), custom portal |
| **Provider / Marketplace** | L1 (core) → L4 (UI) | `APIExport` / `APIBinding` + provider bootstrap (always, L1); `ProviderMetadata` / `ContentConfiguration` for the marketplace UI (L4, optional) ([RFC 004](004_core-platform-extendability.md), [RFC 006](006_provider-bootstrap-operator.md)) | Platform Mesh core providers | Third-party providers contributed to the marketplace |
| **Observability** | cross-cutting | OTLP (OpenTelemetry export) | Platform Mesh emits OTLP; no bundled backend | Your Prometheus / Grafana / Datadog / SIEM / audit pipeline |
| **GitOps Engine** | deployment | `pm-operator` deploy contract  | none bundled; Flux or ArgoCD (operator chooses) | plain Helm, any deployment tooling |

The table is the canonical map, and a **snapshot of the current state**. It
is the set of interfaces Platform Mesh exposes today, not a closed list; new
capabilities are added over time per
[Adding a New Capability](#adding-a-new-capability). The two interfaces this
RFC introduces get a note; the four that already exist point to their contract:

- **Identity**: OIDC + RFC 7591 DCR (DCR is what makes managed multi-org
  practical). Managed/federated modes and the provisioning contract are in the
  [L2 contract](#l2-authentication).
- **Observability** (cross-cutting, all layers incl. core): components emit
  **OTLP** ([RFC 002](002-runtime-lifecycle-v2.md)); no bundled backend, so a
  compliance-bound operator routes telemetry into their own pipeline.
- **Authorization** → the [L3 contract](#l3-authorization): the webhook is the
  interface, OpenFGA the swappable engine.
- **Portal** → the [L4 contract](#l4-portal): the open standard is **GraphQL**
  (via the gateway); OpenMFP is the default, alternative portals plug in via an
  adapter.
- **Provider / Marketplace** → the L1 provider bootstrap mechanism
  ([RFC 004](004_core-platform-extendability.md), [RFC 006](006_provider-bootstrap-operator.md)).
- **GitOps Engine** → the [L1 contract](#l1-core): deployment is an
  interface, not a capability; Platform Mesh bundles no engine and supports
  several (Flux, ArgoCD, plain Helm), chosen by the operator.

### Adapters

An **adapter** is a small component that implements a Platform Mesh interface on
behalf of an external system that does not already speak it natively; the
platform depends on the *interface*, and the adapter absorbs the *vendor
specifics*. Identity is the worked example: a **standards adapter** for
DCR-capable IdPs vs. a **vendor-API adapter** for IdPs without DCR (the
[L2 interface](#l2-authentication)). Writing an adapter is real engineering,
not a configuration toggle; the project ships and tests the defaults and
documents each interface so the ecosystem (community) can build the rest.

## Adding a New Capability

The capabilities described in this RFC, the L1–L4 layers and the
[interface catalog](#interfaces), are a **snapshot of the current state**, not
a closed set. The whole point of the model is that Platform Mesh can grow
without renegotiating the product: a new capability enters the framework by
following the same contract every existing module already obeys.

1. **Define the interface.** A capability *is* its interface, not its
   implementation. Adopt an open standard where one exists; otherwise specify
   the contract explicitly, with KRM resources as the intent surface. 
2. **Declare dependencies and absent-dependency behavior.** State which
   existing modules the capability depends on and how it degrades or falls
   back when an optional dependency is absent (as L3 falls back to Kubernetes
   RBAC). This keeps the dependency direction one-way and preserves
   composability.
3. **Ship a default implementation.** Provide one curated, integrated default
   so the capability is first-class out of the box, the *product* half of the
   framework/product pairing.
4. **Classify its support.** Place the capability and any composition it
   introduces in the relevant [support tier](#support-tiers) so the test and
   maintenance promise is explicit from day one.

Most new capabilities are expected to arrive as **providers / extensions** on
top of the existing layers: domain capabilities contributed through the kcp
provider-bootstrap mechanism
(`APIExport` / `APIBinding`, [RFC 004](004_core-platform-extendability.md) /
[RFC 006](006_provider-bootstrap-operator.md)) and surfaced in the marketplace.
Adding a **new layer** to the L1–L4 ordering is a heavier change: it alters the
tested default composition and the support matrix, so it requires its own ADR
and an update to the [Supported Compositions](#supported-compositions) table.

## Per-Module Capability Contracts

Each layer is summarized to its **capability** and its **interface**.

### L1 Core

The smallest Platform Mesh: the account model on kcp, API-first, lowest
footprint. Default modules: kcp, `pm-operator`, `account-operator`,
`security-operator` (baseline), the provider-bootstrap mechanism, and the
multi-cluster substrate (`multicluster-runtime`, `api-syncagent`, `kube-bind`).

**Interface:** the kcp API (workspaces, `APIExport` / `APIBinding`, usable via
kubectl) and an externally configured OIDC issuer.

**Contract, managed state & migrations:** external OIDC issuer & scopes, _to be detailed._

### L2 Authentication

Managed, multi-tenant identity. Default: Keycloak (one realm per organization,
RFC 7591 DCR), driven by the `security-operator` IDP subroutine.

**Interface:** OIDC + RFC 7591 Dynamic Client Registration. Keycloak is the
default implementation, not the contract. DCR-capable IdPs swap directly,
others via a vendor-API adapter.

**Contract, federation modes (managed / federated), provisioning, managed
state & migrations:** identity extension, _to be detailed._

### L3 Authorization

Fine-grained, relationship-based (ReBAC) authorization for every kcp API
request; falls back to managed Kubernetes RBAC when absent. Default engine:
OpenFGA, behind the Kubernetes authorization webhook.

**Interface:** the Kubernetes authorization webhook (`SubjectAccessReview`).
ReBAC is the model; OpenFGA is only the swappable engine. Authorization intent
is expressed as KRM resources and projected into the engine by a single
reconciler (IAM Service).

### L4 Portal

The interactive experience: UI, the GraphQL API surface, extensions, and the
marketplace. Default: OpenMFP (Portal / IAM / Marketplace UIs), the
`kubernetes-graphql-gateway`, `virtual-workspaces`, and
`extension-manager-operator`.

**Interface:** **GraphQL** over the Platform Mesh resources (served by
`kubernetes-graphql-gateway`), what any portal consumes. `ContentConfiguration`
+ the microfrontend contract are the optional, OpenMFP-specific composition
  model, not an open standard. The portal MUST function with L3 absent.

## Packaging and Configuration

Platform Mesh is delivered as an **OCM (Open Component Model) artifact**: the
administrator configures the **PlatformMesh custom resource** and the
**`pm-operator`** reconciles it into the desired set of modules. The operator
owns that reconciliation and is **not tied to a specific packaging or GitOps
tool** — it does not require Helm or any particular deployment engine. Platform
Mesh bundles no GitOps engine and supports several (Flux, ArgoCD, plain Helm),
chosen by the operator. The PlatformMesh resource is the single configuration
surface:
module toggles, configuration, and runtime mutability (adding or removing
modules after install, with install-time selection as the minimum first step).

## Support Tiers

Composition support and implementation support are graded separately:

**Compositions** (which modules are deployed):

| Tier | Definition | Compositions |
|------|-----------|--------------|
| **Tier 1**: Layered shapes | Covered by the full E2E test suite on every release | S1 (L1), S2 (L1+L2), S3 (L1+L2+L3), S4 (full) |
| **Tier 2**: Validated compositions | Actively tested in CI, documented configuration | L1+L2+L4, L1+L3, L1+L3+L4, L1+L4 |
| **Tier 3**: Community compositions | Known to work or community-contributed, not part of CI | everything else |

**Implementations** (which software fills a module slot):

| Tier | Definition | Examples |
|------|-----------|----------|
| **Tier 1**: Default | The default software for a module slot, covered by the full E2E test suite | Keycloak, OpenFGA, OpenMFP |
| **Tier 2**: Validated | Actively tested in CI, documented configuration | Dex (as external OIDC issuer) |
| **Tier 3**: Community | Known to work or community-contributed (typically via an adapter), not part of CI | Okta, Auth0, Zitadel, Entra ID, ADFS, Headlamp |

The **GitOps engine is not tiered**: the deployment tool (Flux, ArgoCD, plain
Helm, …) is the operator's choice, not a module implementation.

## Open Questions / TODOs

- **Layer membership** — define precisely which components belong to which
  layer, and what is a layer member vs. an optional module deployed on top.
- **Minimum deployment** — L1 (kcp + the account model) is the floor that makes
  a deployment "Platform Mesh"; confirm the irreducible minimum and what
  distinguishes it from plain kcp.
- **User stories** — derive the concrete implementation user stories from the
  layers and their interfaces (the work backlog).
- **Replace a layer, not just omit it** — when an optional layer is absent, an
  operator may want to plug in their own implementation rather than the built-in
  fallback (e.g. their own authorization instead of the Kubernetes RBAC
  fallback). Define how a layer is *replaced*, not only omitted.
- **Breaking changes for clean interfaces** — providing single, clean interfaces
  may require breaking changes to current APIs/CRDs, which is in tension with the
  additive-only [Non-Goal](#non-goals); scope and sequence them.

## Non-Goals

- Removing Keycloak, OpenFGA, or OpenMFP as defaults. They remain the
  out-of-the-box experience (composition S4 with Tier-1 implementations). The
  framing here makes them *the default implementations of interfaces*, not the
  only ones; it does not demote them.
- Shipping or maintaining an adapter for every alternative system. The project
  ships and tests the default implementations and documents each interface;
  alternative implementations (Zitadel, Entra ID, Headlamp, …) are
  first-class wherever a conforming adapter is maintained, but guaranteeing one
  for every system is explicitly out of scope; that is what keeps the support
  surface bounded.
- Breaking changes to existing Platform Mesh API or CRD schemas. Additive
  changes to the PlatformMesh resource for module toggles and OIDC
  configuration are in scope.
- Prescribing the administrator's deployment tooling. Platform Mesh defines
  what the `pm-operator` reconciles, not how its manifests reach the
  cluster.

## Glossary

- **Module / capability**: one unit of value (identity, authorization, a
  portal, …), delivered by its components and exposed through one interface. The
  two words are used interchangeably in this RFC.
- **Interface**: the contract a capability exposes, an open standard where one
  exists (OIDC + DCR, the Kubernetes authorization webhook, GraphQL, OTLP, …).
  The interface, not the implementation, is what the rest of the platform
  depends on.
- **Layer (L1–L4)**: a tested bundle of modules (the *L* is for *layer*); the
  officially supported compositions.
- **Framework / product**: the *framework* is the full set of interfaces; the
  *product* (Platform Mesh) is the default set of modules that implement them
  (Keycloak, OpenFGA, OpenMFP).
- **Adapter**: a component that implements an interface on behalf of an
  external system that does not already speak it (e.g. a non-DCR IdP).
- **DCR (Dynamic Client Registration, RFC 7591)**: a standard endpoint that lets
  the platform register an OIDC client per organization automatically, instead
  of static, hand-configured clients; what makes managed multi-org practical.
- **Provider / provider-bootstrap mechanism**: a capability / API contributed
  via kcp `APIExport` / `APIBinding` and the bootstrap flow of
  [RFC 004](004_core-platform-extendability.md) /
  [RFC 006](006_provider-bootstrap-operator.md); the mechanism for *capability
  extension*, distinct from the infrastructure interfaces above.
- **KRM**: the Kubernetes Resource Model; kcp is Platform Mesh's KRM control
  plane. Intent is expressed as resources and reconciled into effects.
- **kcp**: the control-plane substrate Platform Mesh is built on; see
  [docs.kcp.io](https://docs.kcp.io).
- **Organization / account / workspace**: the account model: organizations and
  accounts map to kcp workspaces.
