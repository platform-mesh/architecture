# ADR 018: Authentication Module Boundaries

| Status  | Proposed   |
|---------|------------|
| Date    | 2026-10-27 |

---
## Context and Problem Statement

As [RFC 008](../rfc/008-platform-mesh-modularization.md) describes, authentication mechanisms that form the authentication capabilities required for Platform Mesh's core functionalities are currently implemented through the current choice of packaging a dedicated keycloak instance packaged with Platform Mesh.
To enable administrators to operate Platform Mesh in a variety of scenarios a managed IdP is not always the desired solution to solve authentication, thus authorization needs to be restructured into a module that separates the API from its current (default) implementation.
This seperation and definition of API boundaries should be acomplished with open standards whenever possible and CRD's when additional Platform Mesh requirements need to be met by an implementation.

---
## Scope and Non-Goals
This ADR aims to discuss and ultimately decide on:
- what requirements the boundary for the authentication module/capability has to fulfill
- which components have to be modified and in wich way

The decision will be split into multiple smaller parts:
- Authentication-Protocol
- Multi-Tenancy & Multi-Provider
- IAM-Service Boundary
- Invites

---
## Decision Drivers
- **Replaceability**: Core Platform Mesh components interaction with authentication mechanisms should not be tied to a the API of a specific identity provider
- **Defining Core Concepts**: As Platform Mesh does more then just leverage basic OIDC to provide authentication(-near) functionality, these functionalities need to be either considered core Platform Mesh concepts or implementation details of the default managed KeyCloak implementation
- **Open Standards**: leveraging open standards whenever possible
- **Possible Compositions**: the possible composition variants the capability needs to accomodate

---
## Considered Options

### Authentication-Protocol
The clear contender is OpenID Connect (OIDC) as it is the currently used protocol and one of the default authentication strategies supported by KCP. As already mentioned in RFC 008 DCR(RFC 7591 Dynamic Client Registration) should also be part of the extended open standard, as it ensures the necessary functionality that the reconcilitation of authentication configurations within platform mesh requires to be feasible.
Even though there are alternatives for the basic authentication functionality of OpenID Connect (for example SAML 2.0) none of them are worth considering over OIDC + DCR)

### Multi-Tenancy & Multi-Provider
Providing each `org` workspace with its own dedicated/isolated IdP "tenant" is solved in the current implementation by creating seperate KeyCloak realms within the managed Keycloak Instance - one realm for each org. This is currently done by the security-operator, which reconciles `IdentityProviderConfiguration`'s as Keycloak Realms & the required DCR's and stores client secrets etc. in kubenetes secrets, referenced by the `IdentityProviderConfiguration`.
Even though this is a great solution as a default implementation, isolated IdP "tenant" isolation does not have to automatically mean multi-tenant IdP, as completely seperate IdP Instances can handle this requirement just as well.
Since reconciliation of the `IdentityProviderConfiguration` currently includes a lot of assumptions regarding tenant creation that is not configurable this isn't the full API contract that we are looking for, as it represents just the (as the name says) configuration part rather then the actual IdP itself.
*This builds upon the assumption that tenant isolation work organization workspaces is the desired authentication model*
*adopters could also think about operating multiple workspaces with the same oidc idp which would require platform mesh to define its own identity segmentation model or rely on authorization alone to ensure unauthorized access to org-seperated-workspaces resources*

To accomodate the requirement of isolation on an IdP-Level and ensure the feasibility of reconciliation of those isolated "tenants" Platform Mesh has to define its requirements on an API-Level.

1. Split reconcilitation of `IdentityProviderConfiguration` including realm creation into replaceable service -> `IdentityProviderConfiguration` becomes the boundary
    - tenant provisioning including DCR must be implemented by the alternative implementation
    - DCR is only a "soft" API defined by the `IdentityProviderConfiguration`'s `IdentityProviderClientConfig`-spec rather then the protocol/specification itself
    - DCR could be implemented in the custom idp-controller itself, no additional seperate adapter required

2. Split reconcilitation of IdP Configuration and Tenant provisioning -> OIDC + DCR & `IdentityProviderTenant` become the boundary
    - tenant provisioning is split from the current idp controller's reconciliation and may be replaced to enable idp provisioning for other idps
    - new CRD `IdentityProviderTenant` representing the the provisioned idp itself including necessary credentials and meta data for `IdentityProviderConfiguration` reconciliation & all regular OIDC auth flows
    - OIDC & DCR become the API contract the security-operator relies on to ensure proper authentication configuration
    - DCR has to be implemented explicitly by a custom adapter, when the idp does not support it out of the box

### IAM-Service Boundary
The IAM-Service serves as the central gateway for user & role management. It defines a generic User-Management API via GraphQL.
Even though replaceability of implementation details was kept in mind when implementing the GraphQL-API-Surface, currently from a service perspective it serves as both the api contract and the implementation as its interface is contained inside the service itself (to be specific the `ResolverService` Interface).
Also the IAM-Service bundles identity with authorization management which in turn blurres the line between the authentication and authorization capabilities.
This bundling requires a custom IAM-Service implementation including authorization even when just authentication is being replaced with a non-default solution.

1. To consider the current implementation as a RFC 008 compliant boundary we must assume:
    - the `ResolverService` interface is a satisfying api boundary thats independent from its implementation
    - implementing `ResolverService` and thus providing a (partial) custom implementation of the IAM-Service is considered a adapter
    - unambiguous affiliation of the IAM-Service to the authentication or authorization capability can/must not be made even though replaceability is part of its requirements

2. When considering the current implementation as non compliant:
    - the `ResolverService` interface is too ambiguous to be used as a clear boundary for neither authentication nor authorization
    - the IAM-Service's Resolver-API must be split to provide seperate api boundaries for authentication and authorization which IAM-Service relies on

### Invites
Invites are currently a necessary means to populate newly created (empty) keycloak realms with an initial user.
Invites could be part of of the authentication API boundary if the invitation process is considered a core platform mesh concept.
This leaves 3 options with some additional necessary considerations:

1. Invites as core platform mesh concept
    - BYO IDP deployments need their own controller that handles reconciliation of `Invite` CRD's
    - Invites currently include authorization configuration that needs to be split from the invite process itself as it would be considered part of the authentication capability
2. Invites are a implementation detail of the default (managed) Keycloak implementation
    - BYO IDP deployments would need to build their own invite logic or add some kind of initial user creation in the idp-reconciliation
    - if no Invite mechanism is required due to other means of populating users into provisioned IdP tenants the feature may be omitted
3. Invites are an optional concept that can be reconciled by a controller implementation if needed
    - the default implementation would need to change the same way as when `Invite` is authentication only but a core concept
    - invites may be omited when not required
    - higher configuration complexity

---
## Decision Outcomes

### Authentication-Protocol
The clear contender for defining how Authentication between a user and Platform Mesh should be handled is OIDC as the defacto open-standard including DCR at least as the protocol for Client Registration. So that adopters with IdPs without DCR have a clean API to implement their adapter for.

### Multi-Tenancy & Multi-Provider
Chosen Option: **2** Split reconcilitation of IdP Configuration and Tenant provisioning
The default implementation essentially remains as is but tenant creation (or in the default implementations case realm creation) is split from the security operator as it is a authentication capability.
The API for tenant creation becomes a dedicated CRD that preceeds/is a requirement for the reconciliation of `IdentityProviderConfiguration`.

#### Authentication Contract
The `IdentityProviderConfiguration` does not sufficiently describe the configuration of an IdP without the assumption that it is a keycloak realm but the spec of the `IdentityProviderConfiguration` is actually quite fitting for the OIDC- & DCR-part and generic enough that platform mesh's security-operator can reconcile it as long as the required protocol is implemented properly.

For successful generic reconciliation we need two essential pieces of information that is currently directly coupled to keycloak:
1. credentials to send authenticated DCR-Requests (currently directly configured through the security-operator's env)
2. the hostname of the idp to access the endpoints (currently derived from the workspace name, keycloak base url from security-operator's env and the hardcoded `/realms/`-Scheme)

The new CRD `IdentityProviderTenant` needs to therefore trigger the creation of a idp "tenant" at reconciliation-time and (similiar to the `IdentityProviderConfiguration`) provide the two required information pieces back:
1. the credential to authenticate DRC-Requests - this should be a kubernetes secret accessible to the security operator
2. the public metadata in general (for example  Authority) about the idp that is required for OIDC auth-flows additionally to the client information contained in `IdentityProviderConfiguration`

Implementation of a custom adapter in form of a custom `idp controller` that reconciles `IdentityProviderTenant` allows provisioning of the IdP go beyond keycloak realm creation (for example creating a dedicated Dex instance or create a zitadel organization).

> IdP provisioning would have to be more customizable to allow for selection of potential IdP variations, especially when multi-provider (*not multi-instance*) is the desired operating model.
> Since provisioning the IdP is part of the initial provisioning of a new org that starts at the orglogicalcluster controller, at this point it has to already be decided on which IdP is going to be used for the org.
> As extensibility of of the kcp-workspace resource itself is not suitable for such configuration this leaves 2 solutions for the admin that operates such a custom setup:
>
> 1. implement a decision making process as a custom mechanism, that precedes the actual org provisioning which is then resolved by the custom `idp controller`
> 2. implement a decision making process within the `idp controller` that decides at reconsiliation-time how what to provision

Both cases are tricky to implement but supporting complex setups like this out-of-the-box is not in the scope of the Modularization.

### IAM-Service Boundary
Chosen Option: **2.** Split IAM-Service API-Boundary between authentication and authorization

#### Authorization Contract
Seperating the API boundary from the implementation when not possible/feasible through open standards should be done with CRD's.
The boundary of authorization already requires decoupling from OpenFGA with generic CRD's to project authorization-engine specifics onto generic platform mesh resources.
Aligning authentication with this principle, users, as the resource stored within the IdP that IAM-Service manages, requires decoupling. This can be achived by projecting User-Objects onto a custom CRD that is reconciled by a controller (where by default implements user management in the keycloak realm associated with the org's workspace) that can be replaced by a custom adapter when bringing your own IdP(-Solution/-Implementation).
*This assumes that user management is done primarily done via IAM-Service/KRM & and the controller*
*Also at this point the IAM-Service itself no longer defines the API-Contract thus it could be associated with the UI capability comparable to the kubernetes-graph-gateway as it is then omittable in API only deployments*

### Invites
Chosen Option: **3** Invites as an optional core platform mesh concept
The split of Invites from Authentication concepts (OpenFGA to be specific) will be necessery so that `Invite` can clearly be affiliated with authentication.
Keeping the Invite functionality with as a CRD makes the implementation of a controller a choice that can be made by the adopter if initial population of the IdP is desired. The implementation can also be done in a way that fits the specific use case.
Unreconciled `Invite` CRDs are tolerated.

---
## Consequences

- Platform Mesh's authentication boundary is defined by OIDC + DCR, the `IdentityProviderTenant` and `Invite` CRDs as well as a universal `User` CRD to decouple tenant- and user-managment from their respective reconciliation (that differ for different IDP setups)
- Removing direct calls to keycloak within the security-operator and IAM-Service
- authentication configuration no longer assumes a out-of-the-box multi-tenant capable IdP as a requirement
