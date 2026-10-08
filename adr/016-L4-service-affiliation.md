# ADR 016: L4 service affilitaion

| Status  | Proposed   |
|---------|------------|
| Date    | 2026-10-05 |

## Context and Problem Statement
To archive modularization that is proposed in [RFC 008](../rfc/008-platform-mesh-modularization.md) the L4 Boundary must be clearly drawn by determining layer affiliation of each component.
Without a clear distinction for each component we run into the risk not aligning the layer boundary with the actual goal of modularizing. Since the Portal is defined to be replaceable and optional, the boundary needs to be at the API Level that the Portal leverages to fetch and modify Data.

## Scope and Non-Goals
As this ADR aims to decide only about the Layer association of services that could be L4, other services will not be discussed. This should be done in separate ADRs for L3 and L2.

## Decision Drivers
- omittability without interrupting core functionality
- coupling to Authorization-Services especially when they're absent
- support of core functionality like ressource-/service-discovery
- direct coupling of current implementation to OpenMFP, Luigi or the microfrontend-UI-Pattern in general

## Considered Options
To evaluate the affiliation of a service, this sections considers layer-association with possible options:

### Portal
The Portal should obviously be part of L4 as it is deeply coupled von OpenMFP and just represents the default implementation of a solution to operate platform mesh via a graphical user interface

### IAM UI
The IAM UI is also deeply coupled with the microfrontend-pattern and just represents part of the current ui implementation for platform mesh.

### Marketplace UI
The Marketplace UI just as the IAM UI represents a microfrontend part of the current/default ui implementation.

### extension-manager-operator
The extension-manager-operator partially coupled with the microfrontend-Pattern, more specifically Luigi as the microfrontend framework, as it primarily manages the `ContentConfiguration` CRD but also defines `ProviderMetaData`.
Even though the `ContentConfiguration` allows generic 'configurations' from a pure API perspective, the concrete validation implemented in the extension-manager-operator expects and validates these configuration as Luigi configurations.
The Layer affiliation of the extension-manager-operator should be decided based on the association of the `ContentConfiguration` as an implementation detail of the current/default UI implementation or a core platform-mesh functionality.
If it is the latter, the schema-validation and internal model currently implemented should be more generic and be configurable so the actual platform-mesh user may use it for their own portal implementation (without Luigi).
The operator also defines `ProviderMetaData` which essentially imposes the same question as the `ContentConfiguration` due to it being unclear if the concept of provider MetaData being stored/managed by platform-mesh's services is a core feature or an implementation detail of the current/defalt ui implementation.

### terminal-controller-manager
As the terminal controller managers primary use is watching `Terminal` CRD to provide pods to be used as remote terminals in the UI, it is clearly related to L4, even though it is neither coupled to the actual UI implementation (only provides WebSockets-Endpoint) nor to the microfrontend pattern. The functionality is already bound to a API-Boundry in form of the `Terminal` CRD, even though it could also be interpreted as an implementation detail specific to the current/default implementation.

#### terminal-controller-manager as L4 service
Affiliating the terminal-controller-manager with L4 will allow a deployment without L4 (API-only) to completely omit this controller, as there is not need for remote-terminals in browsers via WebSockets without a Portal. On the other hand replacing L4 requires an implementation of a controller to manage the `Terminal` CRD or the implementation to provide a completly custom solution for remote terminal management/provision, if that feature is desired in the alternative UI implementation.

Pro's:
- no unecessary deployment of the terminal-controller-manager in api-only operation
- customizability of the `Terminal` Controller by the implementer in replacement scenarios

Con's:
- when replacing L4 you need to bring your own terminal controller (or compareable solution) if you want remote terminal functonality

#### terminal-controller-manager as L1 service
When considering terminal-controller-manager as a L1 service, it will always be present/deployed even when there is not imminent use for it when running without L4. When replacing L4, the platform-mesh build-in functionality will be available to use for a custom portal implementation where only the client-side implementation needs to be provided. This also ensures preservation of the CRD handling standard via KCP that platform-mesh provides out of the box.

Pro's:
- easy implementation of remote terminals in alternative portal implementations
- no CRD handling through a (potentially replaced) L4 service

Con's:
- fixed (*non-customizable*) implementation of the `Terminal` CRD handling
- terminal-controller-manager is always deployed even when no L4 is present

#### terminal-controller-manager as L1 service with option to disable
This variant comes with all the Pro's and Con's of the regular L1 service variant, except that the controller may not be deployed if not needed.

additional Pro's:
- terminal-controller-manager may be disabled when ommiting L4

additional Con's:
- deployment complexity rises, as the terminal-controller-manager may continue to be disabled when adding L4 at a later stage, which will cause issues
- it is not clear how the feature toggle should be handled
- running with parts of L1 disabled would rather create a new layer which is not desired by RFC 008

### kubernetes-graphql-gateway
The kubernetes-graphql-gateway is the essential way for any interaction with the ressources provided by service-providers via HTTP.
Since the RFC calls for replaceability rather than just optionality, the graphql-schema could be defining the strong API-Contract for other UI implementations and act as the boundary that seperates L4 services.

#### gateway as L4 service
While possible, the graphql-layer could be omitted fully when operating API-only. This would then require direct kubernetes-api calls via `kubectl` and remove the active service discovery as well as automatic api-schema generation that the graphql-gateway provides.

Pro's:
- most flexible way for different ui implementations
- no GraphQL requirement
- completely omittable when API-only via `kubectl` is the desired operating-mode

Con's:
- no clear API-contract
- no explorable HTTP-API-Playground
- ressource-/service-discovery needs to be implemented

#### gateway as L4 service with separation of listener as L1
This is essentially the same variant as the 'gateway as L4 service' with the exeption of the listener being explicitly seperated from the gateway itself.
In this case we lose some flexiblity in the ui implementation and partially regain the GraphQL requirement, but at the same time simplify/implement ressource-/service-discovery out of the box. Additionally to preserve remote cluster access via the `ClusterAccess` CRD the current naming `api: gateway.platform-mesh.io/v1alpha1` would be inapropriate, when considering the listener as the consumer of this resource (even though it does not own its lifecycle).

#### gateway as L1 service
In this case the gateway (and the listener) is always included as its part of platform-mesh's core. This allows the gateway to clearly define a string API-Contract, that UI implementations (including the default implementation) can leverage and clearly seperates the UI implementation as L4 from platform-mesh's Contracts that hold in all possible modular-operating models.

Pro's:
- easiest way to provide a clear API-Boundary
- flexible interaction with data-structures and ressources in general due to composable GraphQL-queries
- explorable HTTP-API-Playground
- ressource-/service-discovery out of the box

Con's:
- fixed commitment to GraphQL as the HTTP-Query-Language
- API-only deployments still provide the full GraphQL-Functionality even when it is not required

#### gateway as L1 service with option to disable
This is essentially the same variant as the 'gateway as L1 service' the the exeption that the service, even though being considered a L1-/Core-service, can be disabled when not needed. For example in API-only operation where direct interaction with `kubectl` is acceptable/desired. This does make an API-only deployment potentially more lightweight when the GraphQL-API is not needed, but at the same time adds complexity to deployments, breaks the API-Contract that L4 would depend on if deployed later and defies the clear affiliation of the service to L1, which in turn could mean that the gateway must define its own layer, that can be omitted/replaced, which would not align with RFC 008. Additionally when fully disabled remote cluster access would need to fully work around the current centrally managed solution handled via the `ClusterAccess` CRD.

### virtual-workspaces
The decision for the virtual-workspaces service depends directly on the layer affiliation of the extension-manager-operator, as its only purpose is to compose data from the two CRD's `ContentConfiguration` and `ProviderMetadata` (and `ApiExports`'s, but this does not matter in the context of this decision).
If the extension-manager-operator's CRDs are not considered core platform-mesh functionalities the principle of the served `MarketplaceEntry`'s is not as well.

### iam-service
The IAM service essentially has the same scenarios as the kubernetes-graphql-gateway regarding it's api contract providing read- and write-functionality via GraphQL.
The key difference is the coupling of the iam-service to L3 and L2 as its primary function is to manage identity (Authentication) and access (Authorization) data.
For this reason in **none** of the possible scenarios should the iam-service be considered a L4 service. As the scope of this ADR does not include L3 and L2 the iam service should primarily be considered as a L1 service, though ADR's for L3 and L2 might affiliante the iam service as L3 or L2.

As the iam service is not part of L4, it's GraphQL-API contract can act as a clear boundary to L4 services, as it is comparable to the kubernetes-graphql-gateway's GraphQl-API contract.

## Decision Outcome
effectively ommited services in modular Setups without L4:
- Portal (as in the current OpenMFP implementation)
    - clearly this is the default implementation and thus should be omittable and replaceable
- IAM UI
    - clearly this is the default implementation and thus should be omittable and replaceable
- Marketplace UI
    - clearly this is the default implementation and thus should be omittable and replaceable

Services that need to be modified to be clearly affiliated with L4 or L1:
- extension-manager-operator
    - when considering the `ContentConfiguration` a core principle of platform-mesh, it needs to be generalized/configurable to be used by any kind of UI implementation
- virtual-workspaces
    - entirely dependend on the extension-manager-operator affiliation

UI-near Services that are **not** omitted and thus should be considered as L1-Services:
- terminal-controller-manager
    - affiliating this service with L1 states the provided functionality of remote-terminals in ephemeral pods via the `Terminal` CRD as a core platform-mesh functionality rather then a specific feature of the default portal implementation
- kubernetes-graphql-gateway
    - this should be a L1 service that fulfills the API-Contracts defined by the GraphQL-Schemas, it should not be able to be disabled so that the defined contract always holds
- iam-service
    - the iam-service is clearly not a L4 service thus should be considered as L1 in the scope of this ADR even though in following ADRs for L2 and L3 affiliation could change to L2 or L3


## Open Questions
To finalize this ADR a consensus needs to be found about what is considered a "core platform-mesh" (L1) functionality an what is not:
- `ContentConfiguration` and `ProviderMetaData`
- `Terminal`
- the marketplace principle in general (without specific focus on microfrontends to impement such a Marketplace)
