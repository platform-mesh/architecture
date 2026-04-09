# RFC 008: Component Modularity and Replaceability in Platform Mesh

| Status  | Proposed                    |
|---------|-----------------------------|
| Author  | @perseus985                 |
| Created | 2026-04-01                  |
| Updated | 2026-04-09                  |

## Summary

This RFC proposes that Platform Mesh components, specifically the identity provider
(Keycloak), the authorization engine (OpenFGA), and the portal layer (OpenMFP +
GraphQL Gateway) become independently optional and replaceable. Operators should
be able to disable any of these components and substitute an alternative
implementation without forking Platform Mesh or losing core functionality.

## Motivation

Platform Mesh today bundles a specific set of technology choices:

| Concern         | Current implementation    |
|-----------------|---------------------------|
| Identity / OIDC | Keycloak                  |
| Authorization   | OpenFGA (ReBAC)           |
| Portal / UI     | OpenMFP + GraphQL Gateway |
| GitOps          | Flux                      |

These choices are reasonable defaults. However, treating them as hard,
non-negotiable dependencies creates a significant adoption barrier for organizations
that:

- Already operate an enterprise identity provider (e.g. Active Directory Federation
  Services, Okta, Ping Identity etc.)
- Have existing authorization infrastructure they are required to use
- Want to use an alternative portal framework (e.g. Backstage or Headlamp)
- Deploy in environments where some components are operational overload for use cases
  where a UI or other components aren't necessary
- Do not need multiple organizations or nested accounts, and want to reduce
  complexity and resource consumption accordingly
- Require an API-only deployment without any portal components (kcp API only)

Modularity directly reduces the complexity and resource footprint of Platform Mesh
in scenarios where the full feature set is not needed.

The goal of this RFC is not to remove any existing component. It is to ensure that
each component can be **turned off independently** and **replaced with an
alternative** that conforms to a defined interface (e.g. OIDC or GraphQL) so that the Platform Mesh core
remains valuable regardless of the surrounding technology choices.

## Security Rationale

In security-hardened and compliance-heavy environments, fewer running components
means a smaller attack surface. This is a direct requirement under BSI Grundschutz
(SYS.1.1.A6).

- **OpenFGA** is called on every kcp API request via webhook. In deployments where
  ReBAC is not required, e.g. single-tenant or environments with existing
  authorization infrastructure, this is unnecessary overhead and an additional
  availability dependency on the critical API path.
- **Keycloak** requires a database, ongoing CVE maintenance, and dedicated
  operational expertise. Organizations already running an enterprise IdP should not
  be required to operate a second one, even when it's only used to sync identities to it.
- **OpenMFP and the GraphQL Gateway** expose additional network endpoints. In
  non-interactive or API-only deployments, these surfaces are unnecessary.

Making components optional is therefore both an operational and a security
improvement.

## Supported Configuration Variants

The following deployment configurations must be supported. Each variant removes
one or more optional components while the remaining stack stays fully functional:

| # | Configuration                         | Description                                      |
|---|---------------------------------------|--------------------------------------------------|
| 1 | w/o OpenFGA                           | No ReBAC; authorization via Kubernetes RBAC       |
| 2 | w/o OpenFGA + Keycloak                | No ReBAC, no managed IdP; external IdP (e.g. Dex) |
| 3 | w/o Keycloak                          | External IdP, OpenFGA still active                |
| 4 | w/o Portal                            | API-only, no UI components                        |
| 5 | w/o Portal + Keycloak                 | API-only with external IdP                        |
| 6 | w/o Portal + OpenFGA                  | API-only, no ReBAC                                |
| 7 | w/o Portal + OpenFGA + Keycloak       | Minimal: API-only, external IdP, Kubernetes RBAC  |

E2E tests must cover each of these configuration variants to ensure functionality
across releases.

### Full Deployment (all components)

```mermaid
graph TD
    User((User))

    subgraph "Layer 3 — Portal & Extensibility"
        Portal[Portal UI / IAM UI]
        GW[GraphQL Gateway]
        ExtMgr[Extension Manager]
    end

    subgraph "Layer 2 — Accounts, Authorization & Authentication"
        KC[Keycloak]
        FGA[OpenFGA]
        ReBACWH[ReBAC AuthZ Webhook]
        SecOp[security-operator]
        AccOp[account-operator]
    end

    subgraph "Layer 1 — Control Plane &#40;always required&#41;"
        KCP[kcp]
        PMOp[PM Operator]
        Flux[Flux]
    end

    User --> Portal
    Portal --> GW
    GW --> KCP
    KC -.->|OIDC| KCP
    ReBACWH -->|authz webhook| KCP
    ReBACWH --> FGA
    SecOp --> FGA
    AccOp --> FGA
    AccOp --> KCP
    ExtMgr --> KCP
    PMOp --> Flux
    Flux --> KCP
```

### Variant 2: w/o OpenFGA + Keycloak (external IdP, K8s RBAC)

```mermaid
graph TD
    User((User))

    subgraph "Layer 3 — Portal & Extensibility"
        Portal[Portal UI / IAM UI]
        GW[GraphQL Gateway]
        ExtMgr[Extension Manager]
    end

    subgraph "Layer 2 — Accounts"
        SecOp[security-operator]
        AccOp[account-operator]
    end

    subgraph "Layer 1 — Control Plane &#40;always required&#41;"
        KCP[kcp]
        PMOp[PM Operator]
        Flux[Flux]
    end

    ExtIdP[External IdP
e.g. Dex]

    User --> Portal
    Portal --> GW
    GW --> KCP
    ExtIdP -.->|OIDC| KCP
    SecOp -->|K8s RBAC| KCP
    AccOp -->|K8s RBAC| KCP
    ExtMgr --> KCP
    PMOp --> Flux
    Flux --> KCP

    style ExtIdP stroke-dasharray: 5 5
```

### Variant 7: w/o Portal + OpenFGA + Keycloak (minimal, API-only)

```mermaid
graph TD
    User((User / API Client))

    subgraph "Layer 2 — Accounts"
        SecOp[security-operator]
        AccOp[account-operator]
    end

    subgraph "Layer 1 — Control Plane &#40;always required&#41;"
        KCP[kcp]
        PMOp[PM Operator]
        Flux[Flux]
    end

    ExtIdP[External IdP
e.g. Dex]

    User -->|kcp API| KCP
    ExtIdP -.->|OIDC| KCP
    SecOp -->|K8s RBAC| KCP
    AccOp -->|K8s RBAC| KCP
    PMOp --> Flux
    Flux --> KCP

    style ExtIdP stroke-dasharray: 5 5
```

## Proposal

### Core vs. Optional Components

We propose a layered model for Platform Mesh components, inspired by the
platform's own architectural thinking. Each layer depends on the layers inside
it, but outer layers can be omitted without breaking inner ones.

**Layer 1 — Control Planes (innermost, always required):**
- KCP (KCP-API-Server, LogicalCluster-Manager, VirtualWorkspaces)
- Platform Mesh Operator (pm-operator)
- Flux (hard runtime dependency: the Operator creates HelmRelease resources
  and expects Flux controllers to reconcile them) (ArgoCD in Future)

**Layer 2 — Accounts, Authorization, Authentication (optional as a whole and per component):**
- account-operator
- Keycloak / iam-service (identity provider)
- OpenFGA (authorization engine)
- rebac-authorization-webhook

**Layer 3 — Portal, Extensibility, API Compositions (optional):**
- OpenMFP Portal + GraphQL Gateway (kubernetes-graphql-gateway)
- extension-manager-operator
- Portal UI, IAM UI
- Marketplace-UI

**Outer layer — Provider-specific components (fully optional):**
- resource-broker

The key requirement is that disabling an outer layer leaves inner layers
fully functional. A deployment with only Layer 1 is minimal but operational.
A deployment with Layers 1 and 2 but without Layer 3 is headless but
multi-tenant capable.

### Single-Organization and Flat Account Mode

When deployed without OpenFGA or in resource-constrained environments, Platform
Mesh should support a **single-organization mode**:

- A predefined organization is provisioned at deployment time
- Creation of additional organizations is disabled
- Account depth is limited to 1 (flat hierarchy, no accounts-in-accounts
  WorkspaceType configuration)

This reduces complexity and resource requirements for scenarios that do not need
multiple organizations or nested account structures, such as API-only deployments
or single-tenant environments.

> **Note:** The account-depth restriction currently only applies at the portal
> level via a feature flag. This RFC requires it to be enforced at the API level
> as well.

### RBAC Fallback When OpenFGA Is Disabled

When OpenFGA is not active, Platform Mesh must manage Kubernetes RBAC resources
as a fallback for roles and permissions. Today the `security-operator` and
`account-operator` maintain the authorization model and tuples in OpenFGA.
Analogously, these operators must create and reconcile the equivalent Kubernetes
RBAC resources (Roles, RoleBindings, ClusterRoles, ClusterRoleBindings) when
OpenFGA is absent.

### Static OIDC Client Configuration

When using an external IdP instead of Keycloak, operators must be able to define
static OIDC client information (client ID, client secret, issuer URL) directly in
the PlatformMesh resource. This avoids the need for dynamic client registration
which external IdPs may not support.

### OIDC Scope Parameterization

OpenMFP and other components that perform OIDC flows must allow the OIDC scopes
to be configured via the PlatformMesh resource. Different IdPs require different
scopes:

- **Dex** requires the `offline_access` scope to issue refresh tokens
- Additional scopes like `email`, `profile`, or `groups` may be necessary
  depending on the IdP and user account setup

This parameterization must be exposed in the PlatformMesh custom resource.

### Runtime Mutability of Components

It must be possible to add or remove optional components after the initial
deployment. For example, adding the portal layer to an initially API-only
deployment, including all dependent components, should be a supported operation.

> **Open question:** Whether OpenFGA can be added or removed post-deployment
> requires further investigation due to the authorization state migration
> involved.

### Defined Interfaces, Not Specific Implementations

For each optional component, Platform Mesh should define an interface that any
conforming implementation can satisfy:

**Identity / OIDC interface:**
- Provide a standards-compliant OIDC discovery endpoint
- Issue JWTs that kcp can validate
- Examples: Keycloak (default), Teleport, Dex, Okta, Azure AD, ADFS

**Authorization interface:**
- Provide a webhook endpoint compatible with the Kubernetes authorization webhook
  protocol, or integrate with kcp's native RBAC
- Examples: OpenFGA (default), Teleport access policies, OPA, or native RBAC
  (no external authorization engine)

**Portal interface:**
- Consume the Platform Mesh API (kcp workspaces, provider APIs) and present them
  to users
- Examples: OpenMFP (default), Backstage via a Platform Mesh plugin, custom portal

**GitOps interface:**
- Reconcile manifests from a source repository into kcp workspaces
- Examples: Flux (default), ArgoCD (https://github.com/platform-mesh/backlog/issues/184)

Defining these interfaces, even informally as documentation, would make it
possible for the community to build and maintain alternative integrations without
modifying Platform Mesh itself.

### Operator Behavior Without Optional Components

The Platform Mesh Operator must handle absent optional components gracefully:

- Controllers that integrate with OpenFGA should check for its presence before
  registering webhook handlers or making authorization calls. If OpenFGA is absent,
  the operators must manage Kubernetes RBAC resources instead (see
  [RBAC Fallback](#rbac-fallback-when-openfga-is-disabled)).
- Controllers that provision Keycloak realms or clients should be skippable via
  configuration. If no identity provider is configured, the operator expects kcp to
  be configured with an external OIDC issuer directly.
- Provider-specific operators that depend on OpenFGA or Keycloak should clearly
  document this dependency and fail with a descriptive error, not a crash, if
  the dependency is absent.

### Packaging and Delivery

The Platform Mesh must be delivered as an **OCM (Open Component Model) artifact**
that can be deployed via the PlatformMesh-Operator. The parameterization of
individual components (enabled/disabled, configuration overrides) must be
expressible through the PlatformMesh custom resource.

Within the OCM component, the actual toggle mechanism is an implementation detail.
Options include:

- **Helm values flags** (`keycloak.enabled: false`) low effort, existing pattern
- **Kustomize components** composable overlays, GitOps-native

What matters is that the outcome is the same: any optional component can be
excluded from a deployment without causing others to fail.

## Open Questions

1. **Dependency audit**  Which Platform Mesh Operator controllers have hard
   runtime dependencies on Keycloak and OpenFGA?  Is there an existing dependency
   map, or would this need to be produced as a first step?

2. **Interface definition**  Is there appetite for formally defining the OIDC,
   authorization, and portal interfaces that alternative implementations must
   satisfy? Even informal documentation would be a useful starting point.

3. **OpenFGA as optional**  The OpenFGA authorization webhook is currently called
   on every kcp API request.  What would RBAC-only mode look like?  Is this a
   configuration change or a code change?  Do we want to provide default RBAC or is it a consumer problem?

4. **Roadmap**  Is component replaceability on the Platform Mesh roadmap?  Are
   there existing architectural decisions that affect feasibility?

## Non-Goals

- Removing Keycloak, OpenFGA, or OpenMFP as defaults, they remain the
  out-of-the-box experience
- Breaking changes to existing Platform Mesh API or CRD schemas (additive changes
  to the PlatformMesh resource for component toggles and OIDC configuration are
  in scope)
- Prescribing which alternative implementations are officially supported

## References

- cert-manager disabling: https://github.com/platform-mesh/platform-mesh-operator/issues/504
- Teleport OIDC documentation: https://goteleport.com/docs/reference/oidc/
- Backstage: https://backstage.io
- Platform Mesh Operator: https://github.com/platform-mesh/platform-mesh-operator
- kcp documentation: https://docs.kcp.io
- BSI SYS.1.1(General Server): https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/07_SYS_IT_Systeme/SYS_1_1_Allgemeiner_Server_Edition_2023.pdf?__blob=publicationFile&v=3