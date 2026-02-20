# Fine-Grained Access Control for APIExport Binding

## Context

Today any APIExport can be bound in any workspace. With this feature we introduce fine-grained control on where APIs can be bound.

## Decision

1. **APIExports are not bindable by default.**. APIExport should be bindable if it fulfills all requirenments (has apiresourceschema for now ) or platform-mesh administator has configured `APIExportBinding` resource for that APIExport
2. Platform-mesh administrator configure `APIExportBinding` resources which should be deployed within platform-mesh installation in platform-mesh-system workspace.
3. The security-operator reconciles `APIExportBinding` resources and writes tuples directly to the appropriate FGA stores.
4. The authorization webhook checks the **consumer's** FGA store to determine if the consumer account is allowed to bind the APIExport.

### Roles

| Role | Responsibility |
|---|---|
| API Provider | Creates APIExport resource, have no control over bindability of his API. |
| Platform-mesh Operator | Defines `APIExportBinding` resources controling which API's can be used in the organizations. |
| Security Operator | Reconcile `APIExportBinding` resources in platform-mesh-system |
| Organization Admin | Can further restrict binding within their own org by creating `APIExportBinding` resource (future). |

## FGA Authorization Model Changes

Add the following to the core authorization module:

```
type apis_kcp_io_apiexport

type core_platform-mesh_io_account
  relations
    define parent: [core_platform-mesh_io_account]
    define bind: [apis_kcp_io_apiexport] or bind from parent
```

This schema update will make us able to allow apiExport binding for some specific account and all it's child accounts. The organization is also an account, so to allow binding for every account in some organization, a platform-mesh administrator need to set organization's account name.

TODO: add kcp schema example and more explanation based on it

## Binding CRD Schema


```yaml
apiVersion: security.platform-mesh.io/v1alpha1
kind: APIExportBinding
metadata:
  name: orchestrate-platform-mesh-io
  namespace: platform-mesh-system
spec:
  apiExportRef:
    name: orchestrate.platform-mesh.io
    clusterName: <provider-logical-cluster-id>

  pathExpressions:
    # path which ends on workspace name without :* should grant access only for one specific account and his child accounts should have no binding permissions
    - root:orgs:default:abc
    - root:orgs:default:cde 
    # root:orgs:* -- allows binding for every organization
    - root:orgs:*
    # root:orgs:<org name>:* -- allows binding for all accounts within one organization
    - root:orgs:demo:*

status:
  conditions:
    - type: Ready
      status: "True"
  # to efectivelly remove/add tuples if expressions have changed
  managedExpressions:
    - root:orgs:default:abc
    - root:orgs:default:cde 
    - root:orgs:*
    - root:orgs:demo:*
```

The `APIExportBinding` CR is created by platform-mesh administrator. The security operator reconciles the `APIExportBinding` resource and creates required tuples.

## Scenarios

### Scenario 1: Specific paths can bind an APIExport

**Configuration:**

```yaml
spec:
  apiExportRef:
    name: orchestrate.platform-mesh.io
    clusterName: provider-cluster-1
  pathExpressions:
    - root:orgs:default:*
    - root:orgs:demo:*
```

The security-operator resolves each path expression to the corresponding workspace(s), then writes one bind tuple per resolved workspace into that workspace’s FGA store (object = root account for that path so `bind from parent` gives all accounts under that path access).

**Stored tuples** (one per resolved workspace):

```
# In store for root:orgs:default
object:   core_platform-mesh_io_account:<default-cluster-id>/default
relation: bind
user:     apis_kcp_io_apiexport:provider-cluster-1/orchestrate.platform-mesh.io

# In store for root:orgs:demo
object:   core_platform-mesh_io_account:<demo-cluster-id>/demo
relation: bind
user:     apis_kcp_io_apiexport:provider-cluster-1/orchestrate.platform-mesh.io
```

**Authz check:**

```
Store:    consumer's FGA store
Object:   core_platform-mesh_io_account:<consumer-cluster>/<consumer-account>
Relation: bind
User:     apis_kcp_io_apiexport:provider-cluster-1/orchestrate.platform-mesh.io
```

FGA evaluation: `bind from parent` traverses up to the root account for that workspace → root has the tuple → **allowed**. Any path not in `pathExpressions` has no tuple → **denied**.

### Scenario 2: All orgs can bind an APIExport

**Configuration:**

```yaml
spec:
  apiExportRef:
    name: orchestrate.platform-mesh.io
    clusterName: provider-cluster-1
  pathExpressions:
    - root:orgs:*
```

The security-operator resolves `root:orgs:*` to all workspaces under `root:orgs` and writes one bind tuple per workspace store.

**Stored tuples** (1 tuple per known workspace under root:orgs):

```
# In each store:
object:   core_platform-mesh_io_account:<org-workspace-cluster-id>/<org-workspace-name>
relation: bind
user:     apis_kcp_io_apiexport:provider-cluster-1/orchestrate.platform-mesh.io
```

**Authz check** from any account in any matching workspace: same as above; `bind from parent` reaches the root account that has the tuple → **allowed**.

Tuple cost: 1 tuple per resolved workspace store. For `root:orgs:*`, that is one per org workspace.

## Component Responsibilities

### platform-mesh operator/administrator
- Creates `APIExportBinding` resources based on defined configuration, e.g. in helm-charts repo or by some admin view UI in future

### APIExportBinding controller (security-operator)

- Watches `APIExportBinding` CRs in platform-mesh-system.
- For each `APIExportBinding`, interprets `pathExpressions` and resolves each path (or wildcard like `root:orgs:*`, `root:orgs:demo:*`) to the set of workspaces.
- For each resolved workspace, looks up AccountInfo / FGA store and writes one bind tuple 
- When `pathExpressions` change or `APIExportBinding` is deleted: removes stale tuples from FGA.

### Authorization Webhook

For bind operations (verb = `bind`):

1. Extract provider cluster from `Extra` attributes
2. Extract consumer cluster from `system:cluster:<id>` in `Groups`
3. Fetch consumer `AccountInfo` for:
   - Consumer account identifier (`Spec.Account.OriginClusterId`, `Spec.Account.Name`)
   - Consumer FGA store ID (`Spec.FGA.Store.Id`)
4. Perform FGA check against the **consumer's store** (not the provider's):

```go
Check(
    StoreId:  consumerInfo.Spec.FGA.Store.Id,
    Object:   "core_platform-mesh_io_account:<consumerOriginCluster>/<consumerName>",
    Relation: "bind",
    User:     "apis_kcp_io_apiexport:<providerCluster>/<apiExportName>",
)
```


## Tuple Summary

| Scenario | Tuples per APIExport | Where |
|---|---|---|
| No Binding / not bindable | 0 | nowhere |
| All orgs (`pathExpressions`: e.g. `root:orgs:*`) | 1 per resolved workspace | each org’s FGA store |
| Subtree (`pathExpressions`: e.g. `root:orgs:demo:*`) | 1 per resolved workspace under path | each matching FGA store |

## Open Considerations
- **Workspace-level restrictions**: a future iteration should allow workspace admins to remove the bind tuple from their own store.
- **How to support binding permission for only one specigic account without child accounts**
