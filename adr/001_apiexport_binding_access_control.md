# Fine-Grained Access Control for APIExport Binding

## Context

Today any APIExport can be bound in any workspace. With this feature we introduce fine-grained control on where APIs can be bound.

## Decision

1. **APIExports are not bindable by default.** APIExport should be bindable if it fulfills all requirenments (has apiresourceschema for now ) or platform-mesh administator has configured `APIExportPolicy` resource for that APIExport
2. Platform-mesh administrator configure `APIExportPolicy` resources which should be deployed within platform-mesh installation in platform-mesh-system workspace.
3. The security-operator reconciles `APIExportPolicy` resources and writes tuples directly to the appropriate FGA stores.
4. The authorization webhook checks the **consumer's** FGA store to determine if the consumer account is allowed to bind the APIExport.

### Roles

| Role | Responsibility |
|---|---|
| API Provider | Creates APIExport resource, have no control over bindability of his API. |
| Platform-mesh Operator/Administator | Defines `APIExportPolicy` resources controling which API's can be used in the organizations. |
| Security Operator | Reconcile `APIExportPolicy` resources in platform-mesh-system and creates needed tuples |
| Organization Admin | Can further restrict binding permissions within their own org (future). |

## FGA Authorization Model Changes

Add the following to the core authorization module:

```
type apis_kcp_io_apiexport

type core_platform-mesh_io_account
  relations
    define parent: [core_platform-mesh_io_account]
    define bind_inherited: [apis_kcp_io_apiexport] or bind_inherited from parent
    define bind: [apis_kcp_io_apiexport] or bind_inherited
```

This schema update will make us able to allow apiExport binding for:
1. one specific workspace(account) only, for this security-operator will create a tuple with `bind` relation
2. workspace(account) and all child workspaces(accounts), for this security-operator will create a tuple with `bind_inherited` relation in organization store
3. to allow binding in every user's workspace, security operator will create a tuple with `bind_inherited` relation in every store

- root
    - orgs
        - org1
            - accountA
                - accountB
                    - accountC
        - org2
            - accountD
    - platform-mesh-system

E.g in this kcp structure:
1. to grant binding permissions only for account C, the system would need to have `APIExportPolicy` resource with - `root:orgs:org1:accountA:accountB:accountC` path expression
2. to grant binding persmissions for every account in org1, the system would need to have `APIExportPolicy` resource with - `root:orgs:org1:*` path expression
3. to grant binding persmissions for every account in the system, the system would need to have `APIExportPolicy` resource with - `root:orgs:*` path expression

## Binding CRD Schema


```yaml
apiVersion: security.platform-mesh.io/v1alpha1
kind: APIExportPolicy
metadata:
  name: orchestrate-platform-mesh-io
  namespace: platform-mesh-system
spec:
  apiExportRef:
    name: orchestrate.platform-mesh.io
    clusterName: <provider-logical-cluster-id>

  allowPathExpressions:
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
  # to efectivelly remove tuples if expressions have changed
  managedExpressions:
    - root:orgs:default:abc
    - root:orgs:default:cde 
    - root:orgs:*
    - root:orgs:demo:*
```

The `APIExportPolicy` CR is created by platform-mesh administrator. The security operator reconciles the `APIExportPolicy` resource and creates required tuples.

## Scenarios

### Scenario 1: Only one specific account can bind an APIExport

**Configuration:**

```yaml
spec:
  apiExportRef:
    name: orchestrate.platform-mesh.io
    clusterName: provider-cluster-1
  allowedPathExpressions:
    - root:orgs:org1:accountA:accountB:accountC
```

The security-operator resolves the path expression to the referenced account and writes a single tuple using relation `bind`. This grants bind permission only to that specific account. Child accounts do **not** inherit it because inheritance is only implemented via `bind_inherited from parent`.

**Stored tuple**:

```
# For accountC only:
object:   core_platform-mesh_io_account:<accountC-cluster-id>/accountC
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

FGA evaluation:
- For `accountC`: direct `bind` tuple match → **allowed**
- For any child of `accountC`: no direct `bind` tuple; `bind_inherited` is not set anywhere in the chain → **denied**

### Scenario 2: All accounts in one organization can bind an APIExport

**Configuration:**

```yaml
spec:
  apiExportRef:
    name: orchestrate.platform-mesh.io
    clusterName: provider-cluster-1
  allowedPathExpressions:
    - root:orgs:org1:*
```

The security-operator resolves `root:orgs:org1:*` to the organization root account (org1) and writes a single tuple using relation `bind_inherited`. This allows all descendant accounts to bind via `bind_inherited from parent`.

**Stored tuple** (written into org1's FGA store):

```
# In org1 store:
object:   core_platform-mesh_io_account:<org1-cluster-id>/org1
relation: bind_inherited
user:     apis_kcp_io_apiexport:provider-cluster-1/orchestrate.platform-mesh.io
```

**Authz check** from any account under org1: same as above; `bind_inherited from parent` reaches org1 which has the tuple → **allowed**.

Tuple cost: 1 tuple in org1's store for this APIExport.

### Scenario 3: All accounts in all organizations can bind an APIExport

**Configuration:**

```yaml
spec:
  apiExportRef:
    name: orchestrate.platform-mesh.io
    clusterName: provider-cluster-1
  allowedPathExpressions:
    - root:orgs:*
```

The security-operator resolves `root:orgs:*` to all org root accounts and writes one `bind_inherited` tuple per org store.

**Stored tuples** (1 tuple per org store):

```
# In org-1 store:
object:   core_platform-mesh_io_account:<org1-cluster-id>/org1
relation: bind_inherited
user:     apis_kcp_io_apiexport:provider-cluster-1/orchestrate.platform-mesh.io

# In org-2 store:
object:   core_platform-mesh_io_account:<org2-cluster-id>/org2
relation: bind_inherited
user:     apis_kcp_io_apiexport:provider-cluster-1/orchestrate.platform-mesh.io

# ... one per org
```

**Authz check** from any account in any org: same as above; `bind_inherited from parent` reaches the org root that has the tuple → **allowed**.

## Component Responsibilities

### platform-mesh operator/administrator
- Create `APIExportPolicy` resources based on defined configuration, e.g. in helm-charts repo or by some admin view UI in future

### platform-mesh operator
- define ApiResourceSchema for `APIExportPolicy` resource
- define ApiExport security.platform-mesh.io (naming may change in future) which references ApiResourceSchema for `APIExportPolicy` resource
- define ApiBinding for security.platform-mesh.io ApiExport

### APIExportPolicy controller (security-operator)

- Reconcile `APIExportPolicy` CRs by looking for the dedicated ApiExportEndpointSlice (e.g. security.platform-mesh.io).
- For each `APIExportPolicy`, interprets `allowedPathExpressions` and resolves each path (or wildcard like `root:orgs:*`, `root:orgs:demo:*`) to the set of workspaces.
- For each resolved workspace, looks up AccountInfo / FGA store and writes one bind tuple 
- When `allowedPathExpressions` change or `APIExportPolicy` is deleted: removes stale tuples from FGA.

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
| One account only (`allowedPathExpressions`: concrete account path) | 1 | consumer org store containing that account |
| One org subtree (`allowedPathExpressions`: `root:orgs:<org>:*`) | 1 | that org’s FGA store |
| All orgs (`allowedPathExpressions`: `root:orgs:*`) | 1 per org | each org’s FGA store |

## Open Considerations
- **Organization-level restrictions**: a future iteration should allow workspace admins to remove the bind tuple from their own store.
