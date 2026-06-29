# RFC 008: Provider Permissions Configuration

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
1. Introduce a `ProviderPermissions` CRD in the `provider.platform-mesh.io` API group which will be responsible for extending/overwriting relations which are defined in AuthorizationModel.
2. The `ProviderPermissions` resource will be part of the `providers.platform-mesh.io` APIExport, making it available in provider's workspaces under `:root:providers` path
3. CRs of this CRD will be owned by the provider and will be created in the provider's workspace alongside the ApiExport, ApiResourceSchema, ContentConfiguration, ProviderMetadata, and other provider resources.
4. Providers cannot overwrite relations for types in OpenFGA which they don't own. For example, a provider cannot overwrite relations for an API of another provider or for default Kubernetes resources.

## Flow of work
1. Provider creates a ProviderPermissions resource in the provider's workspace. This might happen when an AuthorizationModel already exists or after it.
2. Security-operator reconciles the ApiBinding resource when somebody binds the provider's API. At this moment, security-operator generates AuthorizationModel resources based on bound `ApiResourceSchema` resources. At this point, security-operator will check if there is a defined `ProviderPermissions` resource and if so, it will merge relations from `ProviderPermissions` with the default `AuthorizationModel`.
3. When `AuthorizationModel` is written into the cluster, store reconciliation will be triggered and the updated `AuthorizationModel` will reach OpenFGA.


## Open questions
1. How to deal with `from` operator and `from parent` relation in terms of our OpenFGA structure

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
    - groupResource: orchestrate.platform-mesh.io.httpbin
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
    # name of permission sections should match gvk of resource specified in ApiExport
    orchestrate.platform-mesh.io.httpbin:
      defaultPermissions:
        get: "codeviewer or member"  # define get: codeviewer or member
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
    orchestrate_platform-mesh_io_httpbin:
      defaultPermissions:
        get: ""
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
    - groupResource: orchestrate.platform-mesh.io.httpbin
      roles:
        - id: codeviewer
          displayName: Code Viewer
          description: Can view code and related resources.
          definition: "[role#assignee] or owner"  # optional as relation definition is needed only for OpenFGA
                                                  # and in RBAC mode it will not be needed

  permissions:
    orchestrate_platform-mesh_io_httpbin:
      defaultPermissions:
        get: ""
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
To change default permission-relations in the generated AuthorizationModel, providers have the `defaultPermissions` section. Currently, we allow overriding only 5 default Kubernetes verbs: `get`, `update`, `delete`, `patch`, `watch`. From an OpenFGA perspective, providers can write any relation instead of the default one. From an RBAC perspective, providers can use this for role name override.

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
    - groupResource: orchestrate.platform-mesh.io.httpbin
      roles:
        - id: codeviewer
          displayName: Code Viewer
          description: Can view code and related resources.
          definition: "[role#assignee] or owner"  # optional as relation definition is needed only for OpenFGA

  permissions:
    orchestrate_platform-mesh_io_httpbin:
      defaultPermissions:
        get: "codeviewer"  # will be parsed into define get: codeviewer
                           # it requires adding codeviewer role/relation into additionalPermissions or in roles section
                           # otherwise AuthorizationModel schema generation will fail
        update: ""
        delete: ""
        patch: ""
        watch: ""
      # additionalPermissions:
      #   codeviewer: "[user] or member"  # will be parsed into relation: define codeviewer: [user] or member
```

### 3. Introduce new roles

```yaml
roles:
  - groupResource: orchestrate.platform-mesh.io.httpbin
    roles:
      - id: codeviewer  # relation/role name. In OpenFGA mode it will stand for define <relation_name>: ...
        displayName: Code Viewer  # provider can specify how the role will be shown on the UI
        description: Can view code and related resources.  # provider can explain what the role does here
        definition: "[role#assignee] or owner"  # (optional) for OpenFGA it's essential as in OpenFGA role is just a relation.
                                                # In future with RBAC system (or another authorization module) it can be left empty
```


Providers can introduce roles to the resource types they manage and only to them. Roles are grouped by resource types which are `gvk` like `orchestrate.platform-mesh.io.httpbin`. While generating AuthorizationModel, security-operator will check if there are custom roles from the provider. If roles are defined for the resource type for which AuthorizationModel is generated, security-operator will create role relations based on the `definition` field. A `definition` like `definition: "[role#assignee] or owner", id: codeviewer` will be transformed into this relation `define codeviewer: [role#assignee] or owner`.

## IAM service integration 

The ProviderPermissions resource is organization-agnostic, which means that the IAM service needs a way to understand if the PP resource is relevant to the current request context. To do this, all that is needed is the gvk of the resource and the clusterName where the resource is created. This is very similar to what the IAM service currently has: 

```go
// resolver api for getting roles related to the gvk
Roles(ctx context.Context, context graph.ResourceContext) ([]*graph.Role, error)

type Resource struct {
	Name      string  `json:"name"`
	Namespace *string `json:"namespace,omitempty"`
}

// Common resource context used across queries and mutations
type ResourceContext struct {
	Group       string    `json:"group"`
	Kind        string    `json:"kind"`
	Resource    *Resource `json:"resource"`
	AccountPath string    `json:"accountPath"`
}
```

The Roles() call already has enough information to get roles which are defined for the resource type. It might work like this:
1. Using AccountPath, get the workspace which has been created for this account.
2. Get the logical cluster name from the workspace's `spec.cluster` field.
3. From the logical cluster, get `ApiBindings` and filter out all KCP's and platform-mesh default apibindings. The result will contain only provider's bindings.
4. Each `ApiBinding` has a reference to the `ApiExport` it's related to. We can get `ProviderPermissions` resources of providers whose API is bound to this specific logical cluster.
5. Then we can get roles for every gvk from ProviderPermissions resources.

Also with a lot of providers it may become slow. To resolve this the cache in iam service might be used which will  be dynamicly populated when ProviderPermissions resources are updated/created/deleted.

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
- The resource names declared in `spec.permissions` keys (e.g., `orchestrate.platform-mesh.io.httpbin`) must correspond to API resources exposed by the referenced `apiExportRef`
- The `groupResource` values in `spec.roles` must match API resources managed by the provider

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