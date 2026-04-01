# RFC 007: Component Modularity and Replaceability in Platform Mesh

| Status  | Proposed                    |
|---------|-----------------------------|
| Author  | @perseus985                 |
| Created | 2026-04-01                  |
| Updated | 2026-04-01                  |

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
  where a UI or other components aren't necessary.


The goal of this RFC is not to remove any existing component. It is to ensure that
each component can be **turned off independently** and **replaced with an
alternative** that conforms to a defined interface (e.g. OIDC or GraphQL) so that the Platform Mesh core
remains valuable regardless of the surrounding technology choices.

## Security Rationale

In security-hardened and compliance-heavy environments, fewer running components
means a smaller attack surface. This is a direct requirement under BSI Grundschutz
(SYS.1.1.A6).

- **OpenFGA** is called on every kcp API request via webhook. In deployments where
  ReBAC is not required, e.g. single-tenant, or environments with existing
  authorization infrastructure. This is unnecessary overhead and an additional
  availability dependency on the critical API path.
- **Keycloak** requires a database, ongoing CVE maintenance, and dedicated
  operational expertise. Organizations already running an enterprise IdP should not
  be required to operate a second one, even when it's only used to sync identities to it.
- **OpenMFP and the GraphQL Gateway** expose additional network endpoints. In
  non-interactive or API-only deployments, these surfaces are unnecessary.

Making components optional is therefore both an operational and a security
improvement.

## Proposal

### Core vs. Optional Components

We propose a conceptual separation between **core** and **optional** components:

**Core** (required for Platform Mesh to function):
- kcp
- Platform Mesh Operator
- Flux (helm-controller + source-controller = hard runtime dependency,
  the Operator creates HelmRelease resources and expects Flux to reconcile them)

**Optional** (can be disabled or replaced):
- security-operator (separate operator, not part of platform-mesh-operator itself)
- Keycloak (identity provider)
- OpenFGA (authorization engine)
- OpenMFP + GraphQL Gateway (portal layer)
- cert-manager (certificate management, see https://github.com/platform-mesh/platform-mesh-operator/issues/504)
- Any provider-specific operators that depend on Keycloak or OpenFGA

The key requirement is that disabling an optional component results in a
**degraded but functional** platform, not a broken one.

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

- Controllers that integrate with OpenFGA should check for it's presence before
  registering webhook handlers or making authorization calls. If OpenFGA is absent,
  authorization falls back to kcp-native RBAC.
- Controllers that provision Keycloak realms or clients should be skippable via
  configuration. If no identity provider is configured, the operator expects kcp to
  be configured with an external OIDC issuer directly.
- Provider-specific operators that depend on OpenFGA or Keycloak should clearly
  document this dependency and fail with a descriptive error, not a crash, if
  the dependency is absent.

### Packaging

We do not prescribe a specific packaging mechanism, that is an implementation
detail for the maintainers to decide. Options include:

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
- Changing the Platform Mesh API or CRD schemas
- Prescribing which alternative implementations are officially supported

## References

- cert-manager disabling: https://github.com/platform-mesh/platform-mesh-operator/issues/504
- Teleport OIDC documentation: https://goteleport.com/docs/reference/oidc/
- Backstage: https://backstage.io
- Platform Mesh Operator: https://github.com/platform-mesh/platform-mesh-operator
- kcp documentation: https://docs.kcp.io
- BSI SYS.1.1(General Server): https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/07_SYS_IT_Systeme/SYS_1_1_Allgemeiner_Server_Edition_2023.pdf?__blob=publicationFile&v=3