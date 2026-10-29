# RFC 012: Provider Permissions Configuration

**Note:** This RFC was previously known as RFC 008, but was re-numbered to solve a conflict with [RFC 008: A Modular Framework for Platform Mesh](008-platform-mesh-modularization.md).

## Context and Problem Statement

Currently, for a provider's API, Platform Mesh generates an AuthorizationModel resource with default relations within Kubernetes. For example, an AuthorizationModel for the example-data provider looks like this:

```
module httpbins

extend type core_namespace
  relations
    define create_orchestrate_platform-mesh_io_httpbins: owner
    define list_orchestrate_platform-mesh_io_httpbins: member
    define watch_orchestrate_platform-mesh_io_httpbins: member

type orchestrate_platform-mesh_io_httpbin
  relations
    define parent: [core_namespace]
    define owner: [role#assignee] or owner from parent
    define member: [role#assignee] or owner or member from parent

    define get: member
    define update: member
    define delete: member
    define patch: member
    define watch: member

    define manage_iam_roles: owner
    define get_iam_roles: member
    define get_iam_users: member
```

And it's not flexible right now. With this feature we want to introduce the ability for providers to add their own relations and roles for an API they introduce. It should support the following functionality:

* Alter/overwrite the auto-generated relations of a resource
* Give the possibility to define new relations
* Introduce the possibility to define new roles (IAM service needs to know about them)

## Decision
1. Introduce a `ProviderPermissions` CRD in the `providers.platform-mesh.io` API group which will be responsible for extending/overwriting relations which are defined in AuthorizationModel.
2. The `ProviderPermissions` resource will be part of the `providers.platform-mesh.io` APIExport, making it available in provider's workspaces under `:root:providers` path and in `:root:platform-mesh-system` workspace.
3. CRs of this CRD will be owned by the provider and will be created in the provider's workspace alongside the ApiExport, ApiResourceSchema, ContentConfiguration, ProviderMetadata, and other provider resources.
4. Providers can overwrite object permissions only for resources exposed by their referenced APIExport. They cannot overwrite permissions for another provider's API or for default Kubernetes resources.
5. Providers can also overwrite the generated `create_<group>_<plural>`, `list_<group>_<plural>` and `watch_<group>_<plural>` relations for those resources on the shared parent account or namespace type. This exception does not allow changing shared parent relations such as `member` or `owner`, or adding provider-defined roles to the parent.

## Flow of work
1. Provider creates a ProviderPermissions resource in the provider's workspace. This might happen when an AuthorizationModel already exists or after it.
2. Security-operator reconciles the ApiBinding resource when somebody binds the provider's API. At this moment, security-operator generates AuthorizationModel resources based on bound `ApiResourceSchema` resources. At this point, security-operator will check if there is a defined `ProviderPermissions` resource and if so, it will merge relations from `ProviderPermissions` with the default `AuthorizationModel`.
3. When `AuthorizationModel` is written into the cluster, store reconciliation will be triggered and the updated `AuthorizationModel` will reach OpenFGA.


## Open questions
1. How to inherit provider-defined roles through shared parent types without introducing conflicting relations. Inheriting existing parent relations for object permissions, and overriding the resource-specific collection permissions described below, does not require adding those roles to the parent.

### OpenFGA DSL Syntax for OpenFGA mode

Relations are defined using the following syntax:

```
define <relation_name>: <relation_definition>
```

Where `<relation_definition>` can include:

| Syntax | Meaning | Relation Type |
|--------|---------|---------------|
| `[user]` | Direct assignment — a `user` can be directly assigned this relation. | Direct |
| `[user:*]` | Direct wildcard — any object of type `user` has this relation (useful for public access). | Wildcard |
| `[user, team#member]` | Direct assignment from multiple types — a `user` or a `member` of a `team`. | Direct Userset |
| `member` | Computed userset — anyone with the `member` relation has this relation. | Computed / Alias |
| `owner from parent` | Tuple-to-userset — inherit from a related object's relation. | Inherited |
| `[user] or member` | Union — direct assignment OR inheritance from another relation. | Logical OR |
| `writer and owner` | Intersection — must satisfy BOTH conditions (Schema 1.1+). | Logical AND |
| `[user] but not blocked` | Exclusion — must satisfy the first condition and NOT the second (Schema 1.1+). | Logical NOT |
| `[user with condition_name]` | Conditional — direct assignment subject to a context condition evaluating to `true` (Schema 1.1+). | ABAC / Conditional |

The variety of options for defining relations makes it hard to propose a typed way of defining them.

### ProviderPermissions Resource Definition

The `ProviderPermissions` CR allows providers to define OpenFGA-style types and relations for their resources. A single CR can define multiple types, each with its own set of relations expressed as strings in OpenFGA DSL format.

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ProviderPermissions
metadata:
  name: orchestrate.platform-mesh.io # name can be different from APIExport name
spec:
  # Reference to the APIExport this config applies to
  apiExport:
    ref:
      name: orchestrate.platform-mesh.io

  # Roles metadata for UI display, grouped by resource type, resource types must match only resources provider manages
  roles:
    - groupResource: httpbin.orchestrate.platform-mesh.io
      roles:
        - id: codeviewer
          displayName: Code Viewer
          description: Can view code and related resources.
          definition: "[role#assignee] or member"
        - id: admin
          displayName: Admin
          description: Administrative access with elevated permissions.
          definition: "[role#assignee] or owner"

  permissions:
    # name of permission sections should match {singular}.{group} of resource specified in ApiExport
    httpbin.orchestrate.platform-mesh.io:
      defaultPermissions:
        create: ""  # uses default parent relation - define create_orchestrate_platform-mesh_io_httpbins: owner
        get: "codeviewer or member"  # define get: codeviewer or member
        list: ""  # uses default parent relation - define list_orchestrate_platform-mesh_io_httpbins: member
        update: ""  # uses default relation - define update: member
        delete: ""
        patch: ""
        watch: ""
      additionalPermissions:
        scan: "[user:*] or member"  # define scan: [user:*] or member

    another_resource: {}  # ...

status:
  conditions:
    - type: Ready
      status: "True"
    - type: RelationsValid
      status: "True"
      message: "All relations parsed successfully"
    # This status is specifically important because this is how the provider can see
    # if an AuthorizationModel was generated fine or there's an error due to PP resource
    - type: AuthorizationModelIsGenerated
      status: "False"
      message: "Failed to generate AuthorizationModel due to relations duplication"
```

## How it addresses each requested functionality

### 1.1 Add new permissions without new roles
```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ProviderPermissions
metadata:
  name: orchestrate.platform-mesh.io
spec:
  apiExport:
    ref:
      name: orchestrate.platform-mesh.io

  permissions:
    httpbin.orchestrate.platform-mesh.io:
      defaultPermissions:
        create: ""
        get: ""
        list: ""
        update: ""
        delete: ""
        patch: ""
        watch: ""
      additionalPermissions:
        codeviewer: "[user] or member"  # will be parsed into relation: define codeviewer: [user] or member
        admin: "[user] or owner"
        scan: "[user] or member"
```

This resource will add 3 new relations into OpenFGA AuthorizationModel schema:
```
    define codeviewer: [user] or member
    define admin: [user] or owner
    define scan: [user] or member
```

### 1.2 Add new permissions with new roles
```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ProviderPermissions
metadata:
  name: orchestrate.platform-mesh.io
spec:
  apiExport:
    ref:
      name: orchestrate.platform-mesh.io

  roles:
    - groupResource: httpbin.orchestrate.platform-mesh.io
      roles:
        - id: codeviewer
          displayName: Code Viewer
          description: Can view code and related resources.
          definition: "[role#assignee] or owner"  # optional as relation definition is needed only for OpenFGA
                                                  # and in RBAC mode it will not be needed

  permissions:
    httpbin.orchestrate.platform-mesh.io:
      defaultPermissions:
        create: ""
        get: ""
        list: ""
        update: ""
        delete: ""
        patch: ""
        watch: ""
      additionalPermissions:
        admin: "[user] or owner"  # a user can also define a role-relation here, but it will not be shown on the UI

        approve: "codeviewer or owner"  # here codeviewer is a relation as well, and provider needs to define what is the codeviewer.
                                        # basically it's a role in RBAC system, and 'role' relation in ReBAC system.
                                        # Provider can specify the roles relations in role: section
```

### 2. Change default permissions
The `defaultPermissions` section allows overriding seven Kubernetes verbs:
`create`, `get`, `list`, `update`, `delete`, `patch` and `watch`. Each field
contains an OpenFGA expression valid on the type where the permission is checked.
Omitting a field or setting it to an empty string retains its default. In a future
RBAC implementation, these fields could instead identify roles.

| Verb | Generated relation | Evaluation context | Default |
| --- | --- | --- | --- |
| `create` | `create_<group>_<plural>` | Parent account or namespace | `owner` |
| `list` | `list_<group>_<plural>` | Parent account or namespace | `member` |
| `watch` | `watch_<group>_<plural>` | Parent account or namespace | `member` |
| `get`, `update`, `delete`, `patch` | Corresponding verb | Resource object | `member` |

The group in a generated relation name is normalized in the same way as in the
authorization webhook. ProviderPermissions keys use `{singular}.{group}`.

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ProviderPermissions
metadata:
  name: orchestrate.platform-mesh.io
spec:
  apiExport:
    ref:
      name: orchestrate.platform-mesh.io

  roles:
    - groupResource: httpbin.orchestrate.platform-mesh.io
      roles:
        - id: codeviewer
          displayName: Code Viewer
          description: Can view code and related resources.
          definition: "[role#assignee] or owner"  # optional as relation definition is needed only for OpenFGA

  permissions:
    httpbin.orchestrate.platform-mesh.io:
      defaultPermissions:
        create: "member"  # evaluated on the parent namespace
        get: "codeviewer"  # will be parsed into define get: codeviewer
                           # it requires adding codeviewer role/relation into additionalPermissions or in roles section
                           # otherwise AuthorizationModel schema generation will fail
        list: "owner"  # evaluated on the parent namespace
        update: ""
        delete: ""
        patch: ""
        watch: "owner"  # evaluated on the parent namespace; object watch inherits it
      # additionalPermissions:
      #   codeviewer: "[user] or member"  # will be parsed into relation: define codeviewer: [user] or member
```

### 2.1 Collection permission context

The Kubernetes authorization webhook checks `create`, `list` and `watch` on
`core_platform-mesh_io_account` for cluster-scoped resources, and on
`core_namespace` for namespaced resources. The generator applies the override to
the resource-specific relation on that parent. For the example above it emits:

```fga
extend type core_namespace
  relations
    define create_orchestrate_platform-mesh_io_httpbins: member
    define list_orchestrate_platform-mesh_io_httpbins: owner
    define watch_orchestrate_platform-mesh_io_httpbins: owner
```

Provider-defined roles and additional permissions exist on the resource object.
They cannot be referenced directly by a parent-scoped expression. Every relation
referenced in a collection expression must already exist on the parent type. The
generator does not create missing shared parent relations or seed assignments.

The meaning of `from parent` follows the type where an expression is evaluated.
In an object's GET expression, it traverses to the object's account or namespace.
In a collection expression on an account or namespace, it traverses to that
parent type's own parent.

LIST and WATCH authorize the entire resource collection within that account or
namespace. They do not filter individual objects using GET permissions. GET must
be configured independently if users should also read individual objects. A
create override similarly controls permission to create resources in that
collection; it does not change update, delete or patch permissions.

### 2.2 Compatibility of existing watch overrides

Previously, `defaultPermissions.watch` changed only the resource object's
`watch` relation. Kubernetes watch requests still checked the hardcoded
`member` permission on the parent. The override now changes that parent
permission, and the resource object's watch relation inherits it via
`watch_<group>_<plural> from parent`. Without an override, both the parent and
object watch relations retain their existing `member` defaults.

An omitted watch field needs no migration. Existing custom watch expressions must
be reviewed in the parent context. For example, `viewer or member` must be
rewritten if `viewer` exists only on the resource, while an object expression
`owner from parent or member` becomes `owner or member` when the intended
owner is on the resource's parent account or namespace. Expressions that use existing parent relations can remain syntactically
unchanged, but the configured object watch permission now follows the parent;
resource-local role assignments no longer grant that configured watch permission
on their own.

### 3. Introduce new roles

```yaml
roles:
  - groupResource: httpbin.orchestrate.platform-mesh.io
    roles:
      - id: codeviewer  # relation/role name. In OpenFGA mode it will stand for define <relation_name>: ...
        displayName: Code Viewer  # provider can specify how the role will be shown on the UI
        description: Can view code and related resources.  # provider can explain what the role does here
        definition: "[role#assignee] or owner"  # (optional) for OpenFGA it's essential as in OpenFGA role is just a relation.
                                                # In future with RBAC system (or another authorization module) it can be left empty
```


Providers can introduce roles to the resource types they manage and only to them. Roles are grouped by resource types using the `{kind}.{group}` format (e.g., `httpbin.orchestrate.platform-mesh.io`). While generating AuthorizationModel, security-operator will check if there are custom roles from the provider. If roles are defined for the resource type for which AuthorizationModel is generated, security-operator will create role relations based on the `definition` field. A `definition` like `definition: "[role#assignee] or owner", id: codeviewer` will be transformed into this relation `define codeviewer: [role#assignee] or owner`.

## IAM service integration

The IAM service retrieves custom roles from `ProviderPermissions` resources using a cache-based approach. The cache is populated via an informer watching `ProviderPermissions` across all provider clusters through the multicluster-provider.

```go
// RolesRetriever interface for getting roles related to a resource type
type RolesRetriever interface {
    GetRoleDefinitions(rctx graph.ResourceContext) ([]RoleDefinition, error)
}

// Common resource context used across queries and mutations
type ResourceContext struct {
    Group       string    `json:"group"`
    Kind        string    `json:"kind"`
    Resource    *Resource `json:"resource"`
    AccountPath string    `json:"accountPath"`
}
```

The role lookup works as follows:
1. IAM service watches all `ProviderPermissions` resources across provider clusters using an aggregate informer from multicluster-provider.
2. When a `ProviderPermissions` is created/updated/deleted, the cache is updated with roles keyed by `groupResource` (e.g., `httpbin.orchestrate.platform-mesh.io`).
3. When `GetRoleDefinitions()` is called with a `ResourceContext`, the cache is queried using a key built from `{kind-lowercase}.{group}` format.
4. Matching roles are returned to be displayed in the UI alongside default roles.

This cache-based approach avoids the need to traverse ApiBindings on each request, providing fast lookups even with many providers.

The GraphQL mutation which can assign a user a role to a resource type:

```graphql
  {
    "context": {
      "group": "orchestrate.platform-mesh.io",
      "kind": "HTTPBin",
      "resource": { "name": "my-httpbin-1" },
      "accountPath": "root:orgs:test:testAccount"
    },
    "changes": [
      { "userId": "john@gmail.com", "roles": ["reviewer"] }
    ]
  }
```

### Validation

The validation webhook must validate that a provider can only define permissions for resources it owns and ideally also validate defined permissions and roles before applying the resource into the cluster. This prevents a malicious or misconfigured provider from overwriting permissions for other providers' or system resources and improves the user's experience.

Potential validation rules:
- The resource names declared in `spec.permissions` keys (e.g., `httpbin.orchestrate.platform-mesh.io`) must correspond to API resources exposed by the referenced `apiExport`
- The `groupResource` values in `spec.roles` must match API resources managed by the provider
- Expressions for `create`, `list` and `watch` must be valid on the resource's parent account or namespace type. Validation must not invent missing parent relations or copy provider-defined roles to shared parent types.
- Overrides of parent relations must be limited to the generated resource-specific create, list and watch relations for resources in the referenced APIExport; shared relations and other providers' permissions remain outside the provider's ownership.

### Finalization

When a `ProviderPermissions` CR is deleted, the operator will just regenerate AuthorizationModel without relations from `ProviderPermissions` resource. So no finalization logic is needed for this resource.

### Roles sharing between different ApiExports
If one provider has 2 or more `ApiExports` and wants to have the same role-relation for their API, they can do this by defining the role with an identical name in both `ProviderPermissions` resources.

## Future improvements
1. How to deal with the `from` operator in OpenFGA.
The `from` operator brings some problems:
* Each provider introduces custom roles and relations only for their API. But with custom `from` relations and the default `from parent`, providers might have the same parent type. If one provider defines a relation like `define approver: [role#assignee] or approver from parent` and another provider defines a relation like `define approver: [role#assignee] or admin or approver from parent`, and the `parent` relation for both of them is the same (e.g., `core_namespace` type), it's not clear which relation should be used in the parent `core_namespace` type.

If providers don't use a custom `from` operator or the default `from parent` relation, they might define relations with identical names but different content because they manage different APIs - until they use the `from` operator.

* Imagine a provider defines a role which is translated into this relation: `define admin: [role#assignee] or admin from parent` and `parent` is an `account` type. In this case, we can create an identical relation `define admin: [role#assignee] or admin from parent` in the `account` type to let the OpenFGA schema compile. But if we do this, we introduce the logic that an admin of a parent account is an admin of a child account - but maybe the user doesn't want this. They might want an admin per account without inheritance between accounts, while still wanting inheritance between resource types.

2. How can a provider create an organization-wide role?
