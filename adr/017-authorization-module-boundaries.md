# ADR 017: Authorization Module Boundaries

| Status | Proposed   |
| ------ | ---------- |
| Date   | 2026-10-25 |

## Context and Problem Statement

As described in [RFC 008](../rfc/008-platform-mesh-modularization.md), Platform Mesh should support any authorization engine behind the webhook or deployments without a ReBAC authorization engine at all. Deployments without an authorization module would fall back to the Kubernetes RBAC. OpenFGA will remain a default but swappable module.

Currently, OpenFGA coupling can be found outside the authorization webhook. For example:

- Security operator manages account relationship tuples.
- IAM writes role assignments directly into OpenFGA, also from invitation workflows.
- Search calls FGA to filter results or list accessible accounts.

Today's state would not allow to replace authorization just by replacing the authZ webhook.

The decision is where to establish the authorization boundary which allows core components to work with each supported backend.

## Decision Drivers

- **Replaceability:** Core components should not be tied to a specific an authorization engine and be provider-agnostic.
- **Clear ownership:** Define who manages permissions for each authorization implementation. One reconciler per projected permission.
- **Rely on base concepts:** Kubernetes RBAC, kcp, `SubjectAccessReview` and KRM resources.
- **Supported Platform Mesh compositions**: OpenFGA stays a default engine, alternative authorization module can be plugged in. Support deployments without an authorization engine that rely on native RBAC.

## Considered Options

1.  **Abstract away only the `SubjectAccessReview` webhook:** permission checks become independent of the engine, but it is not enough to decouple IAM and search and handle authorization state management.
2.  **Define a contract for authorization intent, permission checks and queries:** IAM owns reconciliation through backend adapters.
3.  **Define the contract that requires a separate operator per authorization backend:** each module owns its reconciliation component, adding deployments and moving reconciliation out of IAM.

## Decision Outcome

Chosen option: **2. Define a contract for authorization intent, permission checks and queries.** As described [in RFC 008](../rfc/008-platform-mesh-modularization.md#authorization), the approach includes both the `SubjectAccessReview` webhook and the shared KRM authorization intent projected into the engine by IAM Service through backend adapters. The RBAC adapter can use native resources directly if they can express required intent.

The shared permission-check interface describes the authenticated subjects without engine-specific mechanisms such as relationship and inheritance model in OpenFGA. Authorization-engine implementations support such functionality through their own authorization model. Platform Mesh core does not emulate ReBAC in native RBAC.

### kcp RBAC

To achieve this, the modularization will rely on the base concepts of [the RBAC chain in kcp](https://docs.kcp.io/kcp/main/concepts/authorization/authorizers/). Roles and bindings grant access within a workspace. `APIExport` and `APIBinding` make APIs available across workspaces. An API provider can additionally define a [`MaximalPermissionPolicy`](https://docs.kcp.io/kcp/main/concepts/apis/exporting-apis/#maximal-permission-policy) to limit access to exported resources.

Authorization modules are integrated through the `SubjectAccessReview` webhook. The integration must specify authorizer order and `Allow`, `Deny` and `NoOpinion` behavior. If a backend is temporarily unavailable it must not switch to native RBAC or allow failed checks.

kcp evaluates maximal permission policies (MPP) within its RBAC chain. The complication is that the RBAC chain and the external webhook are separate authorizers -- kcp API server has a built-in RBAC chain, including MPP checks, but it separately calls Platform Mesh’s authorization webhook over HTTPS. That means we cannot assume that every authorization path includes MPP checks. A backend grant is not allowed to authorize the request prohibited by target-workspace access restrictions or to give access to an exported API that an MPP prohibits.

### Authorization Contract

| Part                  | Responsibility                                                                                                                                                                                   |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Permission checks     | Evaluate an authenticated subject, groups, account/workspace scope and requested operation. Use `SubjectAccessReview` for the kcp webhook and an engine-neutral interface for service checks.    |
| Authorization intent  | Describe policies and assignments as shared KRM resources. Define their scope, revocation rules, ownership, creation and update.                                                                 |
| Authorization queries | Provide the role catalog and assignments needed by IAM. Provide accessible accounts and resource checks needed by search. Define the expected results and access restrictions for these queries. |

`SubjectAccessReview` can only answer a permission request, it cannot store assignments or list accessible resources. Direct calls to a webhook check only that authorizer, so service checks must preserve the same applicable restrictions as API access. The native implementation can use SAR against the target kcp workspace.

The authorization contract's role is to provide a common mapping of authenticated subjects, group claims and account scopes into permission checks. The webhook then receives the identity context provided by kcp with SAR. IAM and search get the corresponding context from authentication data using the same configured mappings. A user's access should not become broader by simply using a different service. Temporary grants must have documented expiry behavior and possible cache must not extend access violating that expiry rule.

### Managed state and ownership

As described in RFC 008, IAM owns reconciliation of authorization intent into the authorization engine. This means one owner for each projected permission, with backend-specific implementations behind the contract. The security-operator produces authorization intent instead of writing directly into OpenFGA. The flow for engine-specific setup looks approximately like this: _KRM authorization intent -> IAM reconciler -> backend implementation -> engine-specific policies and relationships_. With native RBAC, roles and bindings represent the authorization intent themselves.

| Component                          | Responsibility                                                                                                                                                  |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| IAM                                | Expose role and assignment management, validate changes, and persist authorization intent. Provide the default reconciler where backend projection is required. |
| security-operator                  | Produce base intent for initial organization and account access.                                                                                                |
| Backend implementation used by IAM | Translate authorization intent and account/resource metadata into the authorization engine.                                                                     |
| kcp and optional webhook           | Evaluate access requests using the configured authorization chain.                                                                                              |

Native resources such as `Role`, `ClusterRole`, `RoleBinding` and `ClusterRoleBinding` require no additional projection. Additional CRDs should be used only if existing resources cannot describe a required authorization intent.

### Provisioning and access grants

Permission assignment and user provisioning have independent of each other lifecycles. Creating an `Invite` does not grant permissions. A workflow may require both, but assignments use the same authorization contract for invited and externally provisioned users. Access management must work when Platform Mesh user provisioning is disabled. The contract defines how an invitation email resolves to an authenticated subject in the relevant organization and issuer scope.

Alternative IdP integrations may project explicit grants into provider roles or claim mappings, but KRM remains the source of authorization intent. Owner, scope, expiry and revocation rules are required for each projection. IAM provides a default reconciliation process into authorization backend.

### Deployment scenarios

| Scenario             | Behavior                                                                                                              |
| -------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Default with OpenFGA | OpenFGA and its authorization webhook are deployed.                                                                   |
| Native RBAC          | No external engine or RBAC webhook is deployed or registered. IAM implements permission management using native RBAC. |
| Alternative engine   | An adapter implements permission checks, projection and query contracts. No changes in core components are required.  |

The `PlatformMesh` resource allows to configure an authorization module, and `platform-mesh-operator` propagates its configuration to the relevant components. There are no OpenFGA-specific attributes in `AccountInfo`. Core operators, search, shared APIs and libraries, Portal, and IAM service have no direct dependency on OpenFGA.

## Consequences

- Platform Mesh supports authorization provider replacements.
- Core authorization is based on kcp's evaluation of RBAC resources and IAM's common permission management interface.
- Removing direct tuple writes to OpenFGA requires migration of existing grants to KRM.
- Different authorization models in different configurations.
- Leaner infrastructure if using native RBAC-deployment.
