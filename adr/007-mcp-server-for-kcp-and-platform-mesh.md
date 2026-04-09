# ADR 007: MCP Server for kcp and Platform Mesh with Permission-Aware Access

| Status           | Proposed                                                     |
|------------------|--------------------------------------------------------------|
| Date             | 2026-04-09                                                   |
| Decision-makers  | Platform Mesh team, kcp team                                 |
| Issue            | [platform-mesh/backlog#212](https://github.com/platform-mesh/backlog/issues/212), [kcp-dev/kcp#3839](https://github.com/kcp-dev/kcp/issues/3839) |
| Epic             | [platform-mesh/backlog#159](https://github.com/platform-mesh/backlog/issues/159) |

## Context and Problem Statement

Platform Mesh and kcp need an AI-accessible interface that allows LLM-based agents (Claude, Copilot, etc.) to interact with Kubernetes resources managed by kcp. The [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server) project already provides a Go-native MCP server implementation with a dedicated `kcp` toolset, including a kcp cluster provider ([PR #657](https://github.com/containers/kubernetes-mcp-server/pull/657)) that supports recursive workspace discovery, listing, and a pluggable `--cluster-provider` architecture. However, the current implementation lacks a permission-aware discovery mechanism — it cannot answer the fundamental question: **"Which workspaces does this user have access to?"**

In a Platform Mesh deployment, a single kcp instance may manage ~1,000 workspaces across multiple organizations. Exposing all workspaces to every MCP client is unacceptable from both a security and usability perspective. We need a mechanism that:

1. Allows MCP clients to discover only the workspaces they are authorized to access.
2. Integrates with kcp's existing authorization model (RBAC, FrontProxy identity propagation).
3. Scales to hundreds of concurrent users each with distinct permission sets.
4. Works within Platform Mesh's multi-tenant organizational model.

Currently, no such discovery endpoint exists. kcp issue [#3839](https://github.com/kcp-dev/kcp/issues/3839) proposes an out-of-tree "Access" Virtual Workspace with a SelfClusterAccessReview (SCAR) API to fill this gap. This ADR defines the end-to-end architecture for MCP integration in both kcp and Platform Mesh, building on that proposal. All subsequent references to the SCAR API and Access Virtual Workspace in this document refer to that proposal.

## Decision Drivers

- **Security**: Users must only see and interact with workspaces they are authorized to access. No workspace metadata leakage.
- **Scalability**: The solution must handle ~1,000 workspaces with hundreds of concurrent MCP sessions without degrading kcp control plane performance.
- **Simplicity / MVP-first**: The initial implementation should be minimal and iterable. Avoid over-engineering pluggable abstractions prematurely.
- **Kubernetes-native**: Reuse existing kcp primitives — FrontProxy, Virtual Workspaces, RBAC, APIExport/APIBinding — rather than introducing new authentication or authorization stacks.
- **MCP compatibility**: The solution must produce outputs consumable by standard MCP server implementations (tool lists, resource URIs, kubeconfig contexts).
- **Separation of concerns**: kcp provides the access discovery API; Platform Mesh layers organizational policy and multi-tenancy on top.

## Considered Options

### Option 1: Client-Side Workspace Enumeration

The MCP server iterates over all workspaces using a privileged service account and filters results based on SubjectAccessReview calls per workspace.

**Approach**: For each workspace, issue a `SelfSubjectAccessReview` to check if the calling user has at least `get` permission, then return only accessible workspaces.

### Option 2: Access Virtual Workspace with SCAR API

Build a standalone Virtual Workspace service behind kcp's FrontProxy that maintains an in-memory RBAC graph mapping subjects to LogicalClusters. Expose a `SelfClusterAccessReview` endpoint returning the list of accessible workspace endpoints in a single call.

**Approach**: A dedicated controller watches ClusterRoleBindings and RoleBindings across shards, builds an index, and serves it via a REST endpoint at `/services/access-virtual-workspace`.

### Option 3: Platform Mesh Authorization Webhook Integration

Extend Platform Mesh's existing FGA (Fine-Grained Authorization) system to provide workspace-level discovery. The MCP server queries the FGA store directly to determine accessible workspaces.

**Approach**: Reuse the OpenFGA tuples already maintained by the security-operator to derive a workspace list for the authenticated subject.

## Decision Outcome

**Selected: Option 2 — Access Virtual Workspace with SCAR API**, with Platform Mesh extending it for organizational multi-tenancy (combining elements of Option 3 for the Platform Mesh layer).

This option was selected because it provides a clean, Kubernetes-native API that serves both kcp's open-source community and Platform Mesh's proprietary needs. The SCAR API produces exactly the data MCP servers need (cluster names + endpoints), and the Virtual Workspace architecture is a proven pattern within kcp. Platform Mesh builds on this foundation by adding FGA-based organizational scoping without modifying the upstream kcp component.

### Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│                     MCP Client (LLM Agent)                     │
│                     (Claude, Copilot, etc.)                    │
└───────────────────────────┬────────────────────────────────────┘
                            │ MCP Protocol (stdio/SSE)
                            ▼
┌────────────────────────────────────────────────────────────────┐
│                    kubernetes-mcp-server                       │
│                                                                │
│  ┌──────────────┐ ┌──────────────┐ ┌────────────────────────┐  │
│  │ kcp toolset  │ │ core toolset │ │ workspace discovery    │  │
│  │ (existing)   │ │ (pods, etc.) │ │ (NEW: SCAR integr.)    │  │
│  └──────┬───────┘ └──────┬───────┘ └───────────┬────────────┘  │
│         │                │                     │              │
└─────────┼────────────────┼─────────────────────┼──────────────┘
          │ kubeconfig     │ kubeconfig          │ HTTPS
          │ (per-ws)       │ (per-ws)            │ (Bearer/Cert)
          ▼                ▼                     ▼
┌────────────────────────────────────────────────────────────────┐
│                       kcp FrontProxy                           │
│                                                                │
│  Path Routing:                                                 │
│    /clusters/{name}/*  -> Shard (workspace resources)          │
│    /services/access-vw -> Access Virtual Workspace             │
│                                                                │
│  Identity: X-Remote-User, X-Remote-Group, X-Remote-Extra-*     │
└──────────────┬─────────────────────────────┬───────────────────┘
               │                             │
               ▼                             ▼
┌──────────────────────────┐  ┌──────────────────────────────────┐
│       kcp Shards         │  │   Access Virtual Workspace       │
│                          │  │                                  │
│  ┌────────────────────┐  │  │  ┌───────────────────────────┐   │
│  │ Workspace 1        │  │  │  │ RBAC Graph Controller     │   │
│  │ Workspace 2        │  │  │  │ (watches CRB/RB across    │   │
│  │ ...                │  │  │  │  shards, builds subject   │   │
│  │ Workspace N        │  │  │  │  -> LogicalCluster index) │   │
│  └────────────────────┘  │  │  └───────────────────────────┘   │
│                          │  │                                  │
│                          │  │  ┌───────────────────────────┐   │
│                          │  │  │ SCAR REST Handler         │   │
│                          │  │  │ POST /selfclusteraccess.. │   │
│                          │  │  │ -> {clusters, endpoints}  │   │
│                          │  │  └───────────────────────────┘   │
└──────────────────────────┘  └──────────────────────────────────┘
```

### Platform Mesh Extension

In Platform Mesh, an additional layer sits between the MCP server and the SCAR API:

```
┌────────────────────────────────────────────────────────┐
│              Platform Mesh MCP Proxy                   │
│                                                        │
│  1. Authenticate user (Bearer token / certificate)     │
│  2. Query SCAR API for accessible workspaces           │
│  3. Apply FGA organizational policies                  │
│     (filter by account, org membership)                │
│  4. Scope kubernetes-mcp-server kubeconfig             │
│     to authorized workspaces only                      │
│  5. Serve MCP protocol to client                       │
└────────────────────────────────────────────────────────┘
```

## Component Design

### 1. Access Virtual Workspace (kcp — upstream)

**Repository**: Out-of-tree, new repository under `kcp-dev/` (e.g., `kcp-dev/access-virtual-workspace`)

**Deployment**: Singleton service registered with FrontProxy via PathMapping:
```yaml
pathMappings:
  - path: /services/access-virtual-workspace
    backend: https://access-vw.kcp-system.svc:6443
```

**Opt-in Indexing**: Workspaces are indexed only when they contain an APIBinding to a designated system APIExport (e.g., `access.kcp.io`). This prevents indexing workspaces that have not opted into discovery.

**RBAC Graph Controller**:
- Watches `ClusterRoleBindings` and `RoleBindings` across all shards via kcp's informer infrastructure.
- Maintains an in-memory map: `Subject (User | Group) → Set<LogicalCluster>`.
- A subject has access to a LogicalCluster if any CRB or RB in that cluster references the subject (directly or via group membership).
- The controller does **not** evaluate Warrants or Scopes in the initial MVP. These can be added in future iterations.

**SCAR API**:

```go
// SelfClusterAccessReview allows a user to discover which clusters
// they have access to without requiring list permissions on workspaces.
type SelfClusterAccessReview struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Status SelfClusterAccessReviewStatus `json:"status,omitempty"`
}

type SelfClusterAccessReviewStatus struct {
    // Clusters the authenticated subject has at least 'view' access to.
    Clusters []AccessEndpointSlice `json:"clusters"`
}

type AccessEndpointSlice struct {
    // ClusterName is the logical cluster identifier.
    ClusterName string `json:"clusterName"`
    // Endpoint is the FrontProxy URL for this cluster.
    Endpoint string `json:"endpoint"`
}
```

**Request flow**:
1. MCP server sends `POST /services/access-virtual-workspace/apis/access.kcp.io/v1alpha1/selfclusteraccessreviews` with Bearer token or client certificate.
2. FrontProxy terminates TLS, extracts user identity, forwards to Access VW with `X-Remote-User`, `X-Remote-Group`, `X-Remote-Extra-*` headers.
3. Access VW looks up the subject in the RBAC graph and returns matching `AccessEndpointSlice` entries.

### 2. kubernetes-mcp-server Integration (kcp — upstream)

The [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server) already has a kcp cluster provider (merged via [PR #657](https://github.com/containers/kubernetes-mcp-server/pull/657)) that provides recursive workspace discovery via `DiscoverWorkspacesRecursive()`, URL parsing/construction for kcp workspace endpoints, and a workspace watcher. The existing provider uses the `--cluster-provider=kcp` flag and treats workspaces as clusters.

The SCAR integration extends this existing provider rather than replacing it:

**New capability: permission-aware discovery**
- The kcp provider gains an optional SCAR endpoint configuration.
- When configured, `DiscoverAllWorkspaces()` is filtered through SCAR results — only workspaces present in the `AccessEndpointSlice` response are returned.
- Without SCAR configured, the provider falls back to its current recursive discovery behavior (suitable for single-user or development setups).

**Modified behavior**:
- On startup (or on demand), the kcp provider calls SCAR to populate the authorized workspace list.
- Existing kcp toolset operations (`list_workspaces`, `use_workspace`, etc.) are scoped to SCAR-returned workspaces only.
- Per-workspace kubeconfig contexts continue to be generated from workspace URLs using the existing `ConstructWorkspaceURL()` helper, with the user's original credentials (token passthrough).

### 3. Platform Mesh MCP Proxy (Platform Mesh — proprietary)

Platform Mesh deployments add a thin proxy layer between MCP clients and kcp:

**Responsibilities**:
- **Authentication**: Validates Platform Mesh user tokens and maps them to kcp identities.
- **Organizational scoping**: Queries Platform Mesh's FGA store to further filter SCAR results by organizational membership (account hierarchy, `APIExportPolicy` bindings per ADR-002).
- **Rate limiting**: Prevents individual MCP sessions from overloading the control plane (important at ~1,000 workspace scale).
- **Audit logging**: Records which workspaces each MCP session accessed, for compliance.
- **Dynamic kubeconfig generation**: Produces scoped kubeconfigs that the downstream kubernetes-mcp-server consumes.

**Deployment**: Runs as a Deployment in the `platform-mesh-system` workspace, exposed via an Ingress or Gateway API route.

**Configuration**:
```yaml
apiVersion: mcp.platform-mesh.io/v1alpha1
kind: MCPServerConfig
metadata:
  name: default
  namespace: platform-mesh-system
spec:
  # Maximum workspaces returned per SCAR call
  maxWorkspacesPerSession: 100
  # Cache TTL for SCAR results
  accessCacheTTL: 60s
  # Rate limiting per user
  rateLimiting:
    requestsPerMinute: 120
    burstSize: 20
  # FGA integration
  fga:
    enabled: true
    storeResolver: accountInfo
  # Upstream kcp endpoint
  kcp:
    frontProxyURL: https://kcp.platform-mesh.svc:6443
    scarPath: /services/access-virtual-workspace
```

### 4. Permission Model

The permission model operates at three layers:

| Layer | Mechanism | Scope | Owner |
|---|---|---|---|
| **kcp RBAC** | ClusterRoleBindings / RoleBindings per workspace | Workspace-level access (view, edit, admin) | kcp (upstream) |
| **SCAR Discovery** | In-memory RBAC graph → AccessEndpointSlice | Which workspaces a subject can see | kcp (upstream) |
| **Platform Mesh FGA** | OpenFGA tuples per account/org hierarchy | Organizational policy overlay (which SCAR results to expose) | Platform Mesh |

**Example flow for a Platform Mesh user**:
1. User `alice@example.com` connects an MCP client.
2. Platform Mesh proxy authenticates the user, resolves their account to `org:acme/account:alice`.
3. Proxy calls SCAR API with alice's identity → kcp returns 150 workspaces alice has RBAC access to.
4. Proxy queries FGA store → alice's account is bound to org `acme` with access to workspaces under `:root:orgs:acme:*`.
5. Proxy intersects SCAR results with FGA policy → 87 workspaces remain.
6. Proxy generates scoped kubeconfig and starts kubernetes-mcp-server with those 87 contexts.
7. MCP client sees 87 workspaces and can operate on their resources.

## Consequences

### Positive

- **Clean upstream/downstream separation**: kcp provides a generic SCAR API usable by any kcp consumer, not just Platform Mesh. This benefits the kcp open-source community and avoids proprietary lock-in.
- **Single API call for discovery**: The SCAR endpoint replaces N SubjectAccessReview calls with a single request, dramatically reducing control plane load at scale.
- **In-memory indexing**: The RBAC graph controller avoids expensive per-request authorization checks by maintaining a pre-computed index, enabling sub-millisecond lookups.
- **Reuses proven patterns**: Virtual Workspaces, FrontProxy path routing, and APIExport/APIBinding are all battle-tested kcp primitives.
- **Incremental adoption**: The opt-in indexing model (via APIBinding) means workspaces can progressively enable MCP discoverability.
- **kubernetes-mcp-server reuse**: Building on the existing Go-native MCP server avoids duplicating transport, tooling, and Kubernetes client logic.

### Negative

- **New service to operate**: The Access Virtual Workspace is an additional deployment that must be monitored, scaled, and upgraded alongside kcp.
- **Eventual consistency**: The RBAC graph is updated asynchronously from watch events. There will be a brief window (seconds) between a RBAC change and the SCAR result reflecting it.
- **MVP limitations**: Warrants, Scopes, and external webhook authorizers are explicitly excluded from the initial implementation. Users relying on these features will not get accurate SCAR results until future iterations.
- **Platform Mesh proxy complexity**: The additional proxy layer adds latency and is another point of failure in the MCP request path.
- **Cache invalidation**: The Platform Mesh proxy caches SCAR results for performance. Stale caches may briefly show revoked workspaces. The `accessCacheTTL` must be tuned carefully.

## Confirmation

The decision will be confirmed when:

1. The Access Virtual Workspace is deployed behind a kcp FrontProxy and a `POST /selfclusteraccessreviews` request returns only the workspaces the authenticated user has RBAC bindings in — verified with at least two distinct users having different workspace sets.
2. The kubernetes-mcp-server kcp provider, configured with a SCAR endpoint, returns a filtered workspace list matching the SCAR response — verified by comparing `DiscoverAllWorkspaces()` output with and without SCAR enabled.
3. In a Platform Mesh deployment, an MCP client session for a user in organization A does not expose workspaces belonging to organization B — verified via the FGA intersection logic with at least two organizations.
4. A scale test with ~100 workspaces and 10 concurrent MCP sessions shows SCAR response times under 200ms at p99.

## Pros and Cons of the Options

### Option 1: Client-Side Workspace Enumeration

**Pros**:
- No new kcp components required.
- Simple to implement — just loop and check.
- Works with any authorization backend.

**Cons**:
- **O(N) SubjectAccessReview calls** per discovery request (N = number of workspaces). At 1,000 workspaces this is prohibitively expensive.
- Requires the MCP server to run with a privileged service account that can list all workspaces, which contradicts the principle of least privilege.
- Latency scales linearly with workspace count — unacceptable for interactive MCP sessions.
- No caching primitive; every session re-enumerates.

### Option 2: Access Virtual Workspace with SCAR API *(selected)*

**Pros**:
- **O(1) discovery** — single API call returns all accessible workspaces.
- Pre-computed RBAC graph enables sub-millisecond lookups.
- Clean Kubernetes-native API (follows SubjectAccessReview patterns).
- Usable by any kcp consumer, not just MCP servers.
- FrontProxy integration provides identity propagation for free.

**Cons**:
- Requires building and maintaining a new out-of-tree service.
- Initial MVP scope excludes Warrants/Scopes.
- Eventual consistency window for RBAC changes.

### Option 3: Platform Mesh FGA Direct Integration

**Pros**:
- Reuses existing FGA infrastructure and tuples.
- Tight integration with Platform Mesh's organizational model.
- Could provide richer authorization semantics (e.g., time-bound access).

**Cons**:
- **Platform Mesh-specific** — does not benefit kcp's open-source community.
- FGA stores model organizational relationships, not workspace-level RBAC. A mapping layer would still be needed to translate FGA tuples into workspace lists.
- Couples MCP discovery to the FGA system, making it harder to run kcp standalone.
- Would require the MCP server to understand Platform Mesh's account/org model, violating separation of concerns.

## Implementation Roadmap

### Phase 1: kcp — Access Virtual Workspace MVP (Weeks 1-3)

### Phase 2: kubernetes-mcp-server SCAR Integration (Weeks 2-4)

### Phase 3: Platform Mesh MCP Proxy (Weeks 3-6)

### Phase 4: Hardening and Future Iterations (Weeks 6+)

## More Information

### Open Questions

### References

- [kcp Virtual Workspace Architecture](https://github.com/kcp-dev/kcp/tree/main/staging/src/github.com/kcp-dev/virtual-workspace-framework/architecture.md) — Design patterns for kcp Virtual Workspaces.
- [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server) — The upstream MCP server with kcp toolset support.
- [kubernetes-mcp-server kcp provider (PR #657)](https://github.com/containers/kubernetes-mcp-server/pull/657) — The existing kcp cluster provider this ADR builds on.
- [ADR 002: Fine-Grained Access Control for APIExport Binding](002-apiexport-binding-access-control.md) — Platform Mesh's FGA model for API access control; the MCP proxy's organizational scoping builds on this.
- [ADR 003: User Identifier Claim](003_user-identifier-claim.md) — User identity propagation across the stack; dependency for SCAR and FGA intersection.
- [kcp FrontProxy documentation](https://docs.kcp.io/kcp/main/concepts/front-proxy/) — Identity propagation and path routing used by the Access VW.
- [kcp Authorization documentation](https://docs.kcp.io/kcp/main/concepts/authorization/) — Multi-stage RBAC chain referenced by the RBAC graph controller.
