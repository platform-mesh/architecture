# ADR 016: UI Module service affilitaion

| Status  | Proposed   |
|---------|------------|
| Date    | 2026-10-13 |

## Context and Problem Statement
To archive modularization that is proposed in [RFC 008](../rfc/008-platform-mesh-modularization.md) the UI Module's Boundary must be clearly drawn by determining affiliation of each component.
Without a clear distinction for each component we run into the risk not aligning the module boundary with the actual goal of modularizing Platform-Mesh in a way that gives adopters the flexibility that a module based Platform brings to the table. Since the UI is defined to be replaceable and optional, the boundary needs to be defined at an API Level that an abitrary UI implementation can leverage to fetch and modify Data.

## Scope and Non-Goals
As this ADR aims to decide only about the Module association of services that could be part of the UI Module, other services will not be discussed. This should be done in separate ADRs for the Authentication- and Authorization-Module.

## Decision Drivers
- omittability without interrupting core functionality
- coupling to Authorization-Services especially when they're absent
- coupling to current Authentication Solution (and potential changes to `IdentityProviderConfiguration` due to Authentication Modularization)
- support of core functionality like ressource-/service-discovery
- direct coupling of current implementation to OpenMFP, Luigi or the microfrontend-UI-Pattern in general

## Considered Options
To evaluate the affiliation of a service, this sections considers module-association with possible options:

### Portal
The Portal should obviously be part of the UI module as it is deeply coupled with OpenMFP and just represents the default implementation of a solution to operate platform mesh via a graphical user interface

### IAM UI
The IAM UI is also deeply coupled with the microfrontend-pattern and just represents part of the current UI implementation for platform mesh.

### Marketplace UI
The Marketplace UI just as the IAM UI represents a microfrontend part of the current/default UI implementation.

### extension-manager-operator
The extension-manager-operator is partially coupled with the microfrontend-Pattern, more specifically Luigi as the microfrontend framework, as it primarily manages the `ContentConfiguration` CRD but also defines `ProviderMetaData`.
Even though the `ContentConfiguration` allows generic 'configurations' from a pure API perspective, the concrete validation implemented in the extension-manager-operator expects and validates these configuration as Luigi configurations.

#### extension-manager-operator as is
If looked at as a immutable service structure, the module affiliation of the extension-manager-operator would be decided based on the association of the `ContentConfiguration` as an implementation detail of the current/default UI implementation or a core platform-mesh functionality.
If it is the latter, the schema-validation and internal model currently implemented should be more generic and be configurable so the actual platform-mesh user may use it for their own portal implementation (without Luigi).
The operator also defines `ProviderMetaData` which essentially imposes the same question as the `ContentConfiguration` due to it being unclear if the concept of provider MetaData being stored/managed by platform-mesh's services is a core feature or an implementation detail of the current/defalt ui implementation.

#### extension-manager-operator split up
Since we've already established that `ContentConfiguration` is more or less coupled to the micro-frontend-pattern and currenly bound to be a luigi configuration the extension-manager-operator including the `ProviderMetaData` could be labeled as a UI specific concept, which it is not, as it is required for the Marketplace concept to work which is a core Platform Mesh concept (stated in RFC 008).
This creates the opportunity to decouple `ContentConfiguration` & `ProviderMetaData` by splitting the extension-manager-operator up into a part that is UI and a part that is core.
##### The UI part
In this case the reconciliation of `ContentConfiguration` CRs and the parts of the existing `ApiResourceSchema` as well as `ApiExport` are seperated from the `ProviderMetadata` and bundled in a seperate service called `content-configuration-controller-manager`.

##### The core part
The `ProviderMetadata` CRD and therefore its `ApiResourceSchema` and `ApiExport` are seperated and moved into their own virtual-workspace decoupled from the `content-coniguration-controller-manager` and the vw for `ContentConfiguration`s. Naming would also have to be changed to affiliate the `ProviderMetadataResource` (formerly `providermetadatas.ui.platform-mesh.io`) to the marketplace rather than the ui to `providermetadatas.marketplace.platform-mesh.io`.

### terminal-controller-manager
As the terminal-controller-managers primary use is watching `Terminal` CRD to provide pods to be used as remote terminals in the UI, it is clearly related to the UI, even though it is neither coupled to the actual UI implementation (only provides WebSockets-Endpoint) nor to the microfrontend pattern. The functionality is already bound to a API-Boundry in form of the `Terminal` CRD, even though it could also be interpreted as an implementation detail specific to the current/default implementation. Since the service is disabled by default and its primary use being demos of Platform Mesh the functionality and usage of the `Terminal`-CRD is **not** a core Platform Mesh principle/functionality.
Affiliating the terminal-controller-manager with the UI Module will allow a deployment without UI (API-only) to completely omit this service, as there is not need for remote-terminals in browsers via WebSockets without a Portal. On the other hand replacing the UI Module requires an implementation of a controller to manage the `Terminal` CRD or the implementation to provide a completly custom solution for remote terminal management/provision, if that feature is desired in the alternative UI implementation.

Pro's:
- no unecessary deployment of the terminal-controller-manager in api-only operation and UI deployments that do not need it
- customizability of the `Terminal` Controller by the implementer in replacement scenarios

Con's:
- when replacing the UI you need to bring your own terminal controller (or compareable solution) if you want remote terminal functonality

### kubernetes-graphql-gateway
The kubernetes-graphql-gateway is the essential way for the interaction with the ressources provided by service-providers via HTTP by the current UI implementation.
It primarily serves the purpose of wrapping the KRM-Api for the UI to consume providing GraphQL conviniences.
Therefore it does not define any kind of API contract to interact with Platform Mesh ressources itself but rather proxies the KRM-API Contract in a more convenient way (at least from a UI implementation perspective).

#### gateway as UI service
The GraphQL-layer can be omitted fully without losing any core Platform Mesh functionality.
Adopters that implement their own UI Module can but also must implement their own composable Web-API solution (independent from the gateway implemenatation & GraphQL itself) when its needed.

Pro's:
- most flexible way for different ui implementations
- no GraphQL requirement
- completely omittable when API-only and interaction with the KRM-API directly is the desired operating-mode

Con's:
- no explorable HTTP-API-Playground out-of-the-box
- adopters with a BYO UI solution need to implement their own gateway-like solution if such a service is required

#### gateway as a Core service
In this case the gateway (and the listener) is always included as it is part of platform-mesh's core. This would suggest the gateway's generated GraphQl-Schemas are the API-Contract that UI implementations (including the default implementation) could leverage.

Pro's:
- easiest way for a BYO UI to use the API
- flexible interaction with data-structures and ressources in general due to composable GraphQL-queries
- explorable HTTP-API-Playground out-of-the-box

Con's:
- fixed commitment to GraphQL as the HTTP-Query-Language
- API-only deployments still provide the full GraphQL-Functionality even when it is not required (with all drawbacks that a additional not needed service brings)

### virtual-workspaces
The decision on how to affiliate the virtual-workspaces service depends directly on the module affiliation of the extension-manager-operator, as its only purpose is to compose data from the two CRD's `ContentConfiguration` and `ProviderMetadata` (and `ApiExports`'s, but this does not matter in the context of this decision).
If the extension-manager-operator's CRDs are not considered core platform-mesh functionalities the principle of the served `MarketplaceEntry`'s is not as well, since it relies on the `ProviderMetadata` as it is directly part of it's spec.

Since we've already established the idea of decoupling `ProviderMetadata` & `ContentConfiguration` and moving the `ProviderMetadata` into the Marketplace namespace, the two virtual workspaces hosted by the virtual-workspaces service should also be split into two seperate services: one hosting the vw for `MarketplaceEntry` being part of Platform Meshs core module and one for the `ContentConfiguration` being part of the UI module.

### iam-service
The IAM service essentially has the same scenarios as the kubernetes-graphql-gateway regarding it's api contract providing read- and write-functionality via GraphQL.
The key difference is the coupling of the iam-service to L3 and L2 as its primary function is to manage identity (Authentication) and access (Authorization) data.
For this reason in **none** of the possible scenarios should the iam-service be considered a L4 service. As the scope of this ADR does not include L3 and L2 the iam service should primarily be considered as a L1 service, though ADR's for L3 and L2 might affiliante the iam service as L3 or L2.

As the iam service is not part of L4, it's GraphQL-API contract can act as a clear boundary to L4 services, as it is comparable to the kubernetes-graphql-gateway's GraphQl-API contract.

## Decision Outcome
Services that comprise the UI Module:
- Portal (as in the current OpenMFP implementation)
    - clearly this is the default implementation and thus should be omittable and replaceable
- IAM UI
    - clearly this is the default implementation and thus should be omittable and replaceable
- Marketplace UI
    - clearly this is the default implementation and thus should be omittable and replaceable
- terminal-controller-manager
    - the provided functionality of remote-terminals in ephemeral pods via the `Terminal` CRD is a feature of the default portal implementation and **not** a core Platform Mesh principle/functionality
- kubernetes-graphql-gateway
    - due to the gateway not defining the API-Contracts itself but rather "wrapping it for convenience" & GraphQL-commitment and not being required (or undesired) in API-only deployment scenarios the gateway and its listener are a specific implementation detail of the current UI implementation

Services that need to be modified to be clearly affiliated with L4 or L1:
- extension-manager-operator
    - when considering the `ContentConfiguration` a core principle of platform-mesh, it needs to be generalized/configurable to be used by any kind of UI implementation
- virtual-workspaces
    - entirely dependend on the extension-manager-operator affiliation

UI-near Services that are **not** omitted and thus should be considered as L1-Services:
- iam-service
    - the iam-service is clearly not a L4 service thus should be considered as L1 in the scope of this ADR even though in following ADRs for L2 and L3 affiliation could change to L2 or L3


## Open Questions
To finalize this ADR a consensus needs to be found about what is considered a "core platform-mesh" (L1) functionality an what is not:
- `ContentConfiguration` and `ProviderMetaData`
- `Terminal`
- the marketplace principle in general (without specific focus on microfrontends to impement such a Marketplace)
