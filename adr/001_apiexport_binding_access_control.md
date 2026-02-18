# Fine-Grained Access Control for APIExport Binding

## Context

Today any APIExport can be bound in any workspace. With this feature we introduce fine-grained control on where APIs can be bound.

## Decision

1. **APIExports are not bindable by default.** To opt in, the provider adds a label `platform-mesh.io/bindable: "true"` on the APIExport resource to make it bindable.
2. A controller in security operator watches labeled APIExports and creates a `Binding` CRD in platform-mesh-system workspace containing the data needed for FGA tuple creation.
3. The security-operator reconciles `Binding` resources and writes tuples directly to the appropriate FGA stores.
4. The authorization webhook checks the **consumer's** FGA store to determine if the consumer account is allowed to bind the APIExport.

### Roles

| Role | Responsibility |
|---|---|
| API Provider | Labels APIExport as bindable. No control over which orgs can bind. |
| Security Operator | Manages `Binding` resources in platform-mesh-system. Controls system-wide defaults and per-workspace access. |
| Organization Admin | Can further restrict binding within their own org by creating `Binding` resource (future). |

## FGA Authorization Model Changes

Add the following to the core authorization module:

```
type apis_kcp_io_apiexport

type core_platform-mesh_io_account
  relations
    define parent: [core_platform-mesh_io_account]
    define bind: [apis_kcp_io_apiexport] or bind from parent

    ... existing relations unchanged ...
```

This schema update will make us able to allow apiExport binding for some specific account and all it's child accounts. The organization is also an account, so to allow binding for every account in some organization, a platform-mesh administrator need to set organization's account name.

TODO: add kcp schema example and more explanation based on it

## Binding CRD Schema

All spec fields under access control are optional. Exactly one of `allWorkspaces` or `workspaces` must be set for the Binding to be valid.

- **allWorkspaces**: when `true`, every workspace (every account in each workspace’s store) can bind this APIExport. One tuple is written per known workspace store.
- **workspaces**: list of allowed workspaces by `name` and `clusterId`. The security-operator resolves each workspace’s AccountInfo to get the FGA store name or ID, then writes one bind tuple per listed workspace.

```yaml
apiVersion: security.platform-mesh.io/v1alpha1
kind: Binding
metadata:
  name: orchestrate-platform-mesh-io
  namespace: platform-mesh-system
spec:
  apiExportRef:
    name: orchestrate.platform-mesh.io
    clusterName: <provider-logical-cluster-id>

  # Option A: allow all workspaces (omit workspaces)
  allWorkspaces: true

  # Option B: allow specific workspaces only (omit allWorkspaces)
  # workspaces:
  #   - name: org_account
  #     clusterId: 2y7sraiyqfh5gv4r
  #   - name: user_account
  #     clusterId: 1ltwhrw6dhz756w2

status:
  conditions:
    - type: Ready
      status: "True"
  writtenTuples:
    - storeId: <fga-store-id>
      object: "core_platform-mesh_io_account:2y7sraiyqfh5gv4r/katalis"
      relation: "bind"
      user: "apis_kcp_io_apiexport:<provider-cluster>/orchestrate.platform-mesh.io"
```

The `Binding` CRD is auto-created when an APIExport gets the `platform-mesh.io/bindable: "true"` label, and deleted when the label is removed. The security operator sets `allWorkspaces` or `workspaces` to control access and resolves each workspace’s FGA store and root account via AccountInfo using `clusterId`.

## Scenarios

### Scenario 1: Specific workspaces can bind an APIExport

**Configuration:**

```yaml
spec:
  apiExportRef:
    name: orchestrate.platform-mesh.io
    clusterName: provider-cluster-1
  workspaces:
    - name:  org_account
      clusterId: 2y7sraiyqfh5gv4r
    - name: user_account
      clusterId: 1ltwhrw6dhz756w2
```

**Stored tuples** (one per listed workspace, written to that workspace’s FGA store; object is the root account for the workspace so `bind from parent` gives all accounts in the workspace access):

```
# In org_account workspace store:
object:   core_platform-mesh_io_account:2y7sraiyqfh5gv4r/katalis
relation: bind
user:     apis_kcp_io_apiexport:provider-cluster-1/orchestrate.platform-mesh.io

# In user_account workspace store:
object:   core_platform-mesh_io_account:1ltwhrw6dhz756w2/partner
relation: bind
user:     apis_kcp_io_apiexport:provider-cluster-1/orchestrate.platform-mesh.io
```

**Authz check** :

```
Store:    org workspace FGA store
Object:   core_platform-mesh_io_account:<consumer-cluster>/<consumer-account>
Relation: bind
User:     apis_kcp_io_apiexport:provider-cluster-1/orchestrate.platform-mesh.io
```

FGA evaluation: `bind from parent` traverses up to the root account for that workspace → root has the tuple → **allowed**. Any other workspace has no tuple → **denied**.

### Scenario 2: Every workspace can bind an APIExport

**Configuration:**

```yaml
spec:
  apiExportRef:
    name: orchestrate.platform-mesh.io
    clusterName: provider-cluster-1
  allWorkspaces: true
```

**Stored tuples** (1 tuple per known workspace store):

```
# In workspace-1 store:
object:   core_platform-mesh_io_account:<cluster-1>/<org-account-name>
relation: bind
user:     apis_kcp_io_apiexport:provider-cluster-1/orchestrate.platform-mesh.io

# In workspace-2 store:
object:   core_platform-mesh_io_account:<cluster-2>/<org-account-name>
relation: bind
user:     apis_kcp_io_apiexport:provider-cluster-1/orchestrate.platform-mesh.io

# ... one per workspace
```

**Authz check** from any account in any workspace: same as above; `bind from parent` reaches the root account that has the tuple → **allowed**.

Tuple cost: 1 tuple per workspace store. At 100 workspaces = 100 total stored tuples for one APIExport.

## Component Responsibilities

### ApiExport Controller

- Watches APIExport resources across all workspaces for the `platform-mesh.io/bindable` label
- When an APIExport gains the label: creates a `Binding` CRD in platform-mesh-system with a default `allWorkspaces: true`
- When the label is removed: deletes the `Binding` CRD
- Does NOT write FGA tuples directly; that is the security-operator's job

### Binding controller

- New controller/subroutine that watches `Binding` CRDs
- For each `Binding`, determines target FGA stores:
  - If `allWorkspaces` is set: enumerates all workspace stores (e.g. via existing Store CRDs or by direct OpenFGA call)
  - If `workspaces` is set: for each entry, resolves the workspace by `clusterId` → AccountInfo → `Spec.FGA.Store.Id`
- Writes one bind tuple per target (object = account, relation = bind, user = apiexport) directly to FGA via gRPC
- Tracks written tuples in `Binding.Status.WrittenTuples` for cleanup. If initially for apiExport: X binding was allowed for every organization and it was changed to a list of accounts, it's required to cleanup remaining tupels from unspecified accounts and vice versa
- When `allWorkspaces` / `workspaces` change or `Binding` is deleted: removes stale tuples from FGA

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
| Not bindable (default) | 0 | nowhere |
| Specific workspaces (`workspaces`) | 1 per listed workspace | each workspace's FGA store |
| All workspaces (`allWorkspaces`) | 1 per known workspace | each workspace's FGA store |

## Open Considerations

- **New workspace creation**: when `allWorkspaces` is set and a new workspace is created, the security-operator must detect the new store and write the bind tuple (e.g. re-reconcile all Bindings with `allWorkspaces` when a new Store is created).
- **Workspace-level restrictions**: a future iteration could allow workspace admins to remove the bind tuple from their own store, effectively opting out.
