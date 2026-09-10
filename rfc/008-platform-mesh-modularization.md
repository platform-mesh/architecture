# RFC 008: A Modular Framework for Platform Mesh

| Status       | Proposed                            |
|--------------|-------------------------------------|
| Authors      | @perseus985, @aaronschweig          |
| Amended by   | @ntnn                               |
| Reviewers    | @BergCyrill, @perseus985, @mkambeck |
| Created      | 2026-04-01                          |
| Updated      | 2026-09-06                          |

## Summary

This RFC restructures Platform Mesh as a set of composable **modules**. Each
module delivers a capability (a value stream that contributes to the shared
product "Platform Mesh") through a defined interface.

Platform Mesh is shipped as a curated default composition of modules.

The default composition is the officially tested and supported product
of Platform Mesh. Modules can be composed individually. Admins can
assemble "their own" Platform Mesh (for example, UI without the
ReBAC authorization engine, falling back to Kubernetes RBAC).

The modules carry a single conviction: **in everything we do, we believe
in the individual extensibility of Platform Mesh.** This makes Platform
Mesh two things at once:

- **A framework**: a set of standards-based **interfaces** - interface,
  not the implementation, is the contract.

- **A product**: a curated, fully integrated default stack that implements
  those interfaces out of the box: **Keycloak, OpenFGA, and OpenMFP**,
  the supported "all-in-one" Platform Mesh.

The default stack is what most admins run, and it stays first-class. But
where an admin already runs their own system (their own Keycloak, Zitadel,
or Entra ID instead of the bundled IdP; Headlamp or no portal at all instead of
OpenMFP; their own GitOps engine), the promise is: **use the interface, and
where your system does not already speak it, write an adapter.**
The modules are tested defaults, the interfaces are how you make
Platform Mesh yours (see [Interfaces](#interfaces)).

## Motivation

Platform Mesh today bundles a specific set of technology choices:

| Concern         | Current implementation    |
|-----------------|---------------------------|
| Identity / OIDC | Keycloak                  |
| Authorization   | OpenFGA (ReBAC)           |
| Portal / UI     | OpenMFP + GraphQL Gateway |

kcp — the Core control plane — is the non-swappable substrate beneath all of
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

1. **Dependencies become explicit.** Each module declares its
   dependencies explicitly. The API boundary rules out accidental
   coupling with direct calls into a modules implementation.
2. **The support promise stays bounded.** The *tested* set stays small:
   the modules test against the APIs of their dependencies, not the
   dependencies, the default composition is tested end-to-end.
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

## The Model: Modules and Composition

**Vocabulary.** Four words, used precisely (full definitions in the [Glossary](#glossary)):

- a **capability** is one unit of value (identity, authorization, a portal, …) —
  also called a **module**; the two are the same thing in this RFC.
- each module exposes its capability through an **interface**: its contract, an
  open standard where one exists.
- the **framework** is the set of all those interfaces; the **product**
  (Platform Mesh) is the default set of modules that implement them.
- a **boundary**: The API contract a module defines, preferably KRM or
  an open standard.

### Modules

A **module** is the unit of composition, one capability. Each module:

- exposes its capability through a defined interface (preferably
  a KRM-based API where possible, an open standard where one exists),
- declares which other modules it depends on, and
- specifies its behavior when an optional dependency is absent.

Platform Mesh ships a default implementation for every module, but the
interface, not the implementation, is the contract.

### Modules and Boundaries

Modules declare explicit dependencies and are tested against the APIs of
its dependencies, not the dependencies themselves, to ensure the
boundary is not crossed or relies on implementation-specific behaviour.

The initial boundaries established in Platform Mesh are Core,
Authentication, Authorization and UI.

E.g. based on the initial boundaries the **provider-bootstrap
mechanism** will be in core. Similar to the **security-operator**, which
activates subroutines depending on the deployed modules.

Platform Mesh does not deploy a **GitOps engine**, supports several
engines and has no preference.

### Initial Boundaries

- **Core**: The non-negotiable floor of Platform Mesh components.
- **Authentication**: OIDC
- **Authorization**: Kubernetes authorization, other authorizers can be
  mounted in via a authorization webhoook.
- **UI**: The interactive UI, can be omitted for API-only use.

### Default Composition

The default composition of modules is the coherent set of operators,
services and components Platform Mesh currently is composed of.

The core composed of kcp and the pure KRM operators like
accounts-operator and security-operator.

The authentication implemented by Keycloak and supporting components.

The authorization implemented by OpenFGA and supporting components.

The UI composed Platform Mesh UI components like Portal, Marketplace UI, IAM UI.

The default composition is covered by the full end-to-end test suite.

## Interfaces

> *In everything we do, we believe in the individual extensibility of Platform Mesh.*

Modularity (the previous sections) answers *"which capabilities run?"*
Extensibility answers *"whose implementation fills each slot?"* They are the
same mechanism seen from two sides: every module exposes its capability through
a defined **interface**: an open standard wherever one exists, and an admin
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
OpenMFP) is one supported set of implementations of them; an admin who
wants a different implementation targets the interface and supplies an
[adapter](#adapters) where their system does not already speak it.

| Interface | Boundary | Open standard / protocol | Default implementation | Bring your own (via adapter) |
|---|---|---|---|---|
| **Identity** | Authentication | OIDC + RFC 7591 DCR | Keycloak (managed mode) | DCR-capable IdPs (Okta, Auth0, PingFederate) directly; non-DCR IdPs (Zitadel, Entra ID) via a vendor adapter |
| **Authorization** | Authorization | Kubernetes authorization webhook (`SubjectAccessReview`) | OpenFGA, ReBAC engine | Any engine behind the webhook; managed Kubernetes RBAC fallback. Relationship-model portability is the open part |
| **Portal** | UI | **GraphQL** over Platform Mesh resources (served by `kubernetes-graphql-gateway`); `ContentConfiguration` + microfrontend = OpenMFP composition, not a standard ([RFC 001](001-api-providers-and-ui-discovery.md)) | OpenMFP | Headlamp (plugin/adapter), custom portal |
| **Provider / Marketplace** | Core → UI | `APIExport` / `APIBinding` + provider bootstrap (always); `ProviderMetadata` / `ContentConfiguration` for the marketplace UI (UI, optional) ([RFC 004](004_core-platform-extendability.md), [RFC 006](006_provider-bootstrap-operator.md)) | Platform Mesh core providers | Third-party providers contributed to the marketplace |
| **Observability** | cross-cutting | OTLP (OpenTelemetry export) | Platform Mesh emits OTLP; no bundled backend | Your Prometheus / Grafana / Datadog / SIEM / audit pipeline |
| **GitOps Engine** | - | `pm-operator` deploy contract  | none bundled; Flux or ArgoCD (admin chooses) | plain Helm, any deployment tooling |

The table is the canonical map, and a **snapshot of the current state**. It
is the set of interfaces Platform Mesh exposes today, not a closed list; new
capabilities are added over time per
[Adding a New Capability](#adding-a-new-capability). The two interfaces this
RFC introduces get a note; the four that already exist point to their contract:

- **Identity**: OIDC + RFC 7591 DCR (DCR is what makes managed multi-org
  practical). Managed/federated modes and the provisioning contract are in the
  [Authentication contract](#authentication).
- **Observability**: components emit **OTLP** ([RFC
  002](002-runtime-lifecycle-v2.md)); no bundled backend, so
  a compliance-bound operator routes telemetry into their own pipeline.
- **Authorization** → the [Authorization contract](#authorization): the webhook is the
  interface, OpenFGA the swappable engine.
- **Portal** → the [UI contract](#ui): the open standard is **GraphQL**
  (via the gateway); OpenMFP is the default, alternative portals plug in via an
  adapter.
- **Provider / Marketplace** → the Core provider bootstrap mechanism
  ([RFC 004](004_core-platform-extendability.md), [RFC 006](006_provider-bootstrap-operator.md)).
- **GitOps Engine** → the [Core contract](#core): deployment is an
  interface, not a capability; Platform Mesh bundles no engine and supports
  several (Flux, ArgoCD, plain Helm), chosen by the admin.

### Adapters

An **adapter** is a small component that implements a Platform Mesh interface on
behalf of an external system that does not already speak it natively; the
platform depends on the *interface*, and the adapter absorbs the *vendor
specifics*. Identity is the worked example: a **standards adapter** for
DCR-capable IdPs vs. a **vendor-API adapter** for IdPs without DCR (the
[Authentication interface](#authentication)). Writing an adapter is real engineering,
not a configuration toggle; the project ships and tests the defaults and
documents each interface so the ecosystem (community) can build the rest.

## Adding a New Capability

The capabilities described in this RFC, the initial modules and the
[interface catalog](#interfaces), are a **snapshot of the current state**, not
a closed set. The whole point of the model is that Platform Mesh can grow
without renegotiating the product: a new capability enters the framework by
following the same contract every existing module already obeys.

1. **Define the interface.** A capability *is* its interface, not its implementation.
   Adopt an open standard where one exists, otherwise specify
   the contract explicitly, with KRM resources as the intent surface.
2. **Declare dependencies and absent-dependency behavior.** State which
   existing modules the capability depends on and how it degrades or falls
   back when an optional dependency is absent (as authz falls back to Kubernetes
   RBAC). This keeps the dependency direction one-way and preserves composability.
3. **Ship a default implementation.** Provide one curated, integrated default
   so the capability is first-class out of the box, the *product* half of the
   framework/product pairing.
4. **Classify its support.** Place the capability in the relevant
   [support tier](#support-tiers) so the test and maintenance promise is
   explicit from day one.

New capabilities are expected to arrive as their own **providers** in
the Platform Mesh ecosystem, as an **extensions** to existing modules or
as their own module if distinct enough in the functionality.

## Per-Module Capability Contracts

Each module is summarized to its **capability** and its **interface**.

### Core

The smallest Platform Mesh: the account model on kcp, API-first, lowest
footprint. Default modules: kcp, `pm-operator`, `account-operator`,
`security-operator` (baseline), the provider-bootstrap mechanism, and the
multi-cluster substrate (`multicluster-runtime`, `api-syncagent`, `kube-bind`).

**Interface:** the kcp API (workspaces, `APIExport` / `APIBinding`, usable via
kubectl) and an externally configured OIDC issuer.

**Contract, managed state & migrations:** external OIDC issuer & scopes, _to be detailed._

### Authentication

Managed, multi-tenant identity. Default: Keycloak (one realm per organization,
RFC 7591 DCR), driven by the `security-operator` IDP subroutine.

**Interface:** OIDC + RFC 7591 Dynamic Client Registration. Keycloak is the
default implementation, not the contract. DCR-capable IdPs swap directly,
others via a vendor-API adapter.

**Contract, federation modes (managed / federated), provisioning, managed
state & migrations:** identity extension, _to be detailed._

### Authorization

Fine-grained, relationship-based (ReBAC) authorization for every kcp API
request; falls back to managed Kubernetes RBAC when absent. Default engine:
OpenFGA, behind the Kubernetes authorization webhook.

**Interface:** the Kubernetes authorization webhook (`SubjectAccessReview`).
ReBAC is the model; OpenFGA is only the swappable engine. Authorization intent
is expressed as KRM resources and projected into the engine by a single
reconciler (IAM Service).

### UI

The interactive experience: UI, the GraphQL API surface, extensions, and the
marketplace. Default: OpenMFP (Portal / IAM / Marketplace UIs), the
`kubernetes-graphql-gateway`, `virtual-workspaces`, and
`extension-manager-operator`.

**Interface:** **GraphQL** over the Platform Mesh resources (served by
`kubernetes-graphql-gateway`), what any portal consumes. `ContentConfiguration`
+ the microfrontend contract are the optional, OpenMFP-specific composition
  model, not an open standard.

The UI must not make assumptions on a specific authn/authz provider and
work with the common kubernetes APIs.

## Packaging and Configuration

Platform Mesh is delivered as an **OCM (Open Component Model) artifact**: the
administrator configures the **PlatformMesh custom resource** and the
**`pm-operator`** reconciles it into the desired set of modules. The operator
owns that reconciliation and is **not tied to a specific packaging or GitOps
tool** — it does not require Helm or any particular deployment engine. Platform
Mesh bundles no GitOps engine and supports several (Flux, ArgoCD, plain Helm),
chosen by the admin. The PlatformMesh resource is the single configuration
surface:
module toggles, configuration, and runtime mutability (adding or removing
modules after install, with install-time selection as the minimum first step).

## Support Tiers

The Platform Mesh project supports the default composition and validates
it with end-to-end tests.

The modules maintained by the Platform Mesh project are tested within
their own boundaries and against the boundaries of the modules it
requires.

A documented and validated composition is a "headless" composition without UI.

**Implementations** (which software fills a module slot):

| Tier | Definition | Examples |
|------|-----------|----------|
| **Tier 1**: Default | The default software for a module slot, covered by the full E2E test suite | Keycloak, OpenFGA, OpenMFP |
| **Tier 2**: Validated | Actively tested in CI, documented configuration | Dex (as external OIDC issuer) |
| **Tier 3**: Community | Known to work or community-contributed (typically via an adapter), not part of CI | Okta, Auth0, Zitadel, Entra ID, ADFS, Headlamp |

The **GitOps engine is not tiered**: the deployment tool (Flux, ArgoCD, plain
Helm, …) is the admin's choice, not a module implementation.

## Open Questions / TODOs

- **Minimum deployment** — Core (kcp + the account model) is the floor that makes
  a deployment "Platform Mesh"; confirm the irreducible minimum and what
  distinguishes it from plain kcp.
- **User stories** — derive the concrete implementation user stories from the
  modules and their interfaces (the work backlog).
- **Breaking changes for clean interfaces** — providing single, clean interfaces
  may require breaking changes to current APIs/CRDs.

## Non-Goals

- Removing Keycloak, OpenFGA, or OpenMFP as defaults. They remain the
  out-of-the-box experience (the full default composition). The
  framing here makes them *the default implementations of interfaces*, not the
  only ones; it does not demote them.
- Shipping or maintaining an adapter for every alternative system. The project
  ships and tests the default implementations and documents each interface;
  alternative implementations (Zitadel, Entra ID, Headlamp, …) are
  first-class wherever a conforming adapter is maintained, but guaranteeing one
  for every system is explicitly out of scope; that is what keeps the support
  surface bounded.
- Prescribing the administrator's deployment tooling. Platform Mesh defines
  what the `pm-operator` reconciles, not how its manifests reach the
  cluster.
- Modularizing everything up front - modules are created based on
  stakeholder demand.

## Glossary

- **Module / capability**: one unit of value (identity, authorization, a
  portal, …), delivered by its components and exposed through one interface. The
  two words are used interchangeably in this RFC.
- **Interface**: the contract a capability exposes, an open standard where one
  exists (OIDC + DCR, the Kubernetes authorization webhook, GraphQL, OTLP, …).
  The interface, not the implementation, is what the rest of the platform
  depends on.
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
- **Boundary**: The API contract between modules
