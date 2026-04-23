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

Build a standalone Virtual Workspace service behind kcp's FrontProxy that maintains an in-memory access map of subjects to LogicalClusters. Expose a `SelfClusterAccessReview` endpoint returning the list of accessible workspace endpoints in a single call. The access map is built by a pluggable access provider — different authorization backends (RBAC, external webhook/FGA) use different provider implementations.

**Approach**: A dedicated access provider watches the authorization backend (CRBs/RBs for RBAC, or queries an external system for non-RBAC deployments), builds an index, and serves it via a REST endpoint at `/services/access-virtual-workspace`.

## Decision Outcome

**Selected: Option 2 — Access Virtual Workspace with SCAR API and pluggable access providers, consumed by a kcp-upstream MCP Virtual Workspace that uses `kubernetes-mcp-server` as a library.**

This option provides a clean, Kubernetes-native API that serves both kcp's open-source community and Platform Mesh's needs. The SCAR API produces exactly the data MCP servers need (cluster names + endpoints), and the Virtual Workspace architecture is a proven pattern within kcp. A direct FGA integration (querying Platform Mesh's OpenFGA store directly from the MCP server) was considered but rejected — it would be Platform Mesh-specific, would not benefit the kcp community, and would couple MCP discovery to the FGA system. Instead, the pluggable provider model handles both authorization backends behind the same API:

- **kcp with native RBAC** → RBAC access provider (watches CRBs/RBs, builds in-memory graph)
- **kcp with external authorizer (e.g., OpenFGA, webhook)** → External access provider (obtains the map from the external system)

The MVP ships the RBAC access provider. The provider interface is defined upfront so that external authorization backends can be supported without changing the SCAR API surface. Platform Mesh deployments using OpenFGA will implement an FGA access provider that builds the workspace map from FGA tuples directly.

MCP is served as a second Virtual Workspace in the same binary, importing `kubernetes-mcp-server` as a Go library rather than running it as a standalone binary behind a proxy. This keeps the deployment footprint small (one binary, two VWs), removes the need for a separate gateway in standalone kcp (FrontProxy handles identity), and calls the `AccessProvider` in-process — no HTTP round-trip per session. The SCAR REST endpoint on the Access VW still exists for external consumers (CLI tools, dashboards, other services).

Platform Mesh deploys the same two VWs and adds an MCP-aware gateway in front of FrontProxy for OIDC, rate limiting, audit, CORS, and MCP session management. The Platform Mesh-specific authorization logic lives entirely inside the FGA access provider; no Platform Mesh-specific MCP server or proxy component is introduced. [agentgateway](https://github.com/agentgateway/agentgateway) is the reference implementation of the gateway role and was validated in the PoC.

### Alternatives considered and rejected

- **Standalone `kubernetes-mcp-server` binary behind an MCP-aware gateway, SCAR over HTTP.** Ships faster because the binary already exists, but adds a permanent extra component and an HTTP round-trip per session init. Retained only as a transitional / dev-and-CI path; not the long-term target.
- **Platform Mesh fork of `kubernetes-mcp-server` (or bespoke rebuild).** Only justified if Platform Mesh needs MCP toolsets that cannot land upstream. No such toolset is on the current roadmap, and the `AccessProvider` extension point already absorbs the known Platform Mesh-specific behaviors. Deferred — revisit if a concrete upstream-incompatible requirement emerges.

### Architecture Overview

Two Virtual Workspaces run in a single binary behind kcp's FrontProxy:

```
┌────────────────────────────────────────────────────────────────┐
│                     MCP Client (LLM Agent)                     │
│                     (Claude, Copilot, etc.)                    │
└───────────────────────────┬────────────────────────────────────┘
                            │ MCP Protocol (streamable HTTP)
                            ▼
┌────────────────────────────────────────────────────────────────┐
│                       kcp FrontProxy                           │
│                                                                │
│  Path Routing:                                                 │
│    /clusters/{name}/*   -> Shard (workspace resources)         │
│    /services/mcp/*      -> MCP Virtual Workspace               │
│    /services/access/*   -> Access Virtual Workspace            │
│                                                                │
│  Identity: X-Remote-User, X-Remote-Group, X-Remote-Extra-*     │
└───────┬──────────────────────┬─────────────────────┬───────────┘
        │                      │                     │
        ▼                      ▼                     ▼
┌───────────────┐  ┌────────────────────┐  ┌─────────────────────┐
│  kcp Shards   │  │  MCP VW            │  │  Access VW          │
│               │  │  /services/mcp     │  │  /services/access   │
│ ┌───────────┐ │  │                    │  │                     │
│ │ Workspace │ │  │ Per-request:       │  │ ┌─────────────────┐ │
│ │ 1..N      │ │  │ 1. Extract user    │  │ │ Access Provider │ │
│ └───────────┘ │  │ 2. Call Access     │◄─┼─│ (in-process)    │ │
│               │  │    Provider        │  │ │                 │ │
│               │  │    (in-process)    │  │ │ RBAC | External │ │
│               │  │ 3. Build scoped    │  │ └─────────────────┘ │
│               │  │    MCP Provider    │  │                     │
│               │  │ 4. mcpserver.New() │  │ SCAR REST Handler   │
│               │  │ 5. Serve HTTP      │  │ POST /selfcluster.. │
│               │  └────────────────────┘  │ -> {clusters, eps}  │
│               │                          │                     │
└───────────────┘                          └─────────────────────┘
```

**Key design choice**: The MCP VW calls the Access Provider **in-process** — not over HTTP. Both VWs share the same binary and the same `AccessProvider` instance. The SCAR endpoint on the Access VW exists for external consumers (CLI tools, dashboards, other services), but the MCP VW skips the HTTP round-trip entirely.

For **standalone kcp** deployments (no Platform Mesh), no gateway is needed — FrontProxy handles identity, and the MCP VW handles everything else.

For **Platform Mesh** deployments, an MCP-aware gateway sits in front of FrontProxy, providing OIDC authentication, rate limiting, audit logging, and observability. The gateway is required here — Platform Mesh users authenticate via an OIDC provider, and the gateway handles the OAuth flow before passing the identity through to FrontProxy:

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────┐
│  MCP Client      │────▶│  MCP-Aware       │────▶│  kcp         │
│  (LLM Agent)     │     │  Gateway         │     │  FrontProxy  │
│                  │     │  (OIDC, rate     │     │              │
│                  │     │   limit, audit)  │     │  ┌─────────┐ │
│                  │     │  e.g. agent-     │     │  │ MCP VW  │ │
│                  │     │  gateway         │     │  │ Access  │ │
│                  │     │                  │     │  │ VW      │ │
└──────────────────┘     └──────────────────┘     └──┴─────────┴─┘
```

### Platform Mesh Extension

Platform Mesh deploys the same two Virtual Workspaces as standalone kcp, with two additions:

1. An **MCP-aware gateway** (agentgateway as the reference implementation) in front of FrontProxy. It handles OIDC authentication, dynamic client registration for MCP SDK clients, rate limiting, audit, CORS, and MCP session management. The bearer token is passed through unchanged to FrontProxy.
2. An **FGA access provider** implementing the same `AccessProvider` interface. It builds the workspace map from OpenFGA tuples (pull and/or webhook). Organizational scoping (filter by account, org membership) is applied inside the provider, so the SCAR result is already org-correct.

There is no Platform Mesh-specific MCP server component and no Platform Mesh-specific MCP proxy. The Platform Mesh-specific logic lives entirely in the FGA access provider and in the gateway configuration.

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

#### Access Provider Interface

The Access VW delegates the "who has access to what" question to a pluggable access provider. The provider is responsible for building and maintaining the subject-to-workspace map; the SCAR handler simply queries it.

```go
// AccessProvider builds and maintains the mapping of subjects to
// accessible LogicalClusters. Different implementations handle
// different authorization backends.
type AccessProvider interface {
    // ClustersFor returns the set of LogicalClusters the given
    // subject (user + groups) has at least 'view' access to.
    ClustersFor(ctx context.Context, user string, groups []string) ([]AccessEndpointSlice, error)

    // Start begins the provider's background sync (watching RBAC
    // resources, polling an external system, etc.). It blocks until
    // the context is cancelled or an unrecoverable error occurs.
    Start(ctx context.Context) error

    // Ready returns true once the provider has completed its initial
    // sync and can serve accurate results.
    Ready() bool
}
```

#### RBAC Access Provider (default — kcp-native RBAC)

For deployments using kcp's built-in authorization:

- Watches `ClusterRoleBindings` and `RoleBindings` across all shards via kcp's informer infrastructure.
- Maintains an in-memory map: `Subject (User | Group) → Set<LogicalCluster>`.
- A subject has access to a LogicalCluster if any CRB or RB in that cluster references the subject (directly or via group membership).
- The provider does **not** evaluate Warrants or Scopes in the initial MVP. These can be added in future iterations.

#### External/Webhook Access Provider (for non-RBAC deployments)

For deployments where kcp is configured with an external authorizer (e.g., OpenFGA via webhook), there are no CRBs/RBs to watch. The access map must be obtained differently:

- **Push model**: The external authorization system calls a webhook on the Access VW to notify it of access changes (e.g., "user X gained access to workspace Y").
- **Pull model**: The provider periodically queries the external system for the current access map (e.g., listing FGA tuples and translating them to workspace sets).
- **Hybrid**: The provider does an initial full sync via pull, then subscribes to change events.

The exact protocol between the Access VW and the external authorizer is defined by the provider implementation. The SCAR API surface remains identical regardless of which provider is active.

These are mutually exclusive backends — a given kcp deployment uses one or the other. The SCAR API abstracts this away from consumers.

> **Note**: The MVP ships only the RBAC access provider. The provider interface is defined upfront to ensure the architecture supports external backends without API-breaking changes. The external/webhook provider is implemented when a deployment requires it (e.g., Platform Mesh with OpenFGA).

#### SCAR API

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
3. Access VW calls `AccessProvider.ClustersFor(user, groups)` and returns matching `AccessEndpointSlice` entries.

### 2. MCP Virtual Workspace (kcp — upstream)

The MCP VW uses [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server) as a **Go library** — importing `mcpserver.NewServer()` and the `mcpkubernetes.Provider` interface — rather than running it as a standalone binary. This is a kcp upstream component, usable by any kcp consumer.

**Request flow**:

On each incoming MCP request, the handler:

1. **Extracts user identity** from FrontProxy headers (`X-Remote-User`, `X-Remote-Group`).
2. **Calls `AccessProvider.ClustersFor(user, groups)`** in-process to get the list of accessible workspaces. No HTTP round-trip to the Access VW.
3. **Builds a scoped `mcpkubernetes.Provider`** that implements `GetTargets()` returning only the authorized workspaces, and `GetDerivedKubernetes()` constructing per-workspace clients using the caller's bearer token (passthrough).
4. **Creates a stateless MCP server** via `mcpserver.NewServer(config, provider)` with the kcp, core, and other configured toolsets.
5. **Serves via streamable HTTP transport** — `srv.ServeHTTP().ServeHTTP(w, r)`.

**Custom Provider implementation** (sketch):

```go
// ScopedWorkspaceProvider implements mcpkubernetes.Provider,
// exposing only the workspaces the caller has access to.
type ScopedWorkspaceProvider struct {
    workspaces  []AccessEndpointSlice  // from AccessProvider
    bearerToken string                  // caller's token for passthrough
    frontProxy  string                  // kcp FrontProxy base URL
}

func (p *ScopedWorkspaceProvider) GetTargets(ctx context.Context) ([]string, error) {
    names := make([]string, len(p.workspaces))
    for i, ws := range p.workspaces {
        names[i] = ws.ClusterName
    }
    return names, nil
}

func (p *ScopedWorkspaceProvider) GetDerivedKubernetes(ctx context.Context, clusterName string) (*mcpkubernetes.Kubernetes, error) {
    // Find the endpoint for this cluster, build a rest.Config
    // with the caller's bearer token, return a Kubernetes client.
    ...
}
```

**Why this replaces a separate gateway (in standalone kcp)**: Authentication comes from FrontProxy (free). Access scoping comes from the shared AccessProvider (in-process). MCP serving comes from the library. Rate limiting and audit logging are handled as HTTP middleware on the VW handler. There is no need for a separate gateway component unless external OIDC or organizational policy enforcement is required — which is why Platform Mesh adds one.

**Deployment**: Both VWs (MCP + Access) run in the same binary, registered with FrontProxy via PathMapping:

```yaml
pathMappings:
  - path: /services/mcp
    backend: https://mcp-access-vw.kcp-system.svc:6443/mcp
  - path: /services/access-virtual-workspace
    backend: https://mcp-access-vw.kcp-system.svc:6443/access
```

### 3. Permission Model

The permission model depends on which access provider is active:

**Scenario A: kcp-native RBAC deployment**

| Layer | Mechanism | Scope |
|---|---|---|
| **kcp RBAC** | ClusterRoleBindings / RoleBindings per workspace | Workspace-level access (view, edit, admin) |
| **SCAR Discovery** | RBAC access provider → AccessEndpointSlice | Which workspaces a subject can see |

The RBAC provider reads directly from Kubernetes RBAC resources. The SCAR result reflects the current state of CRBs/RBs across shards.

**Scenario B: External authorizer deployment (e.g., Platform Mesh with OpenFGA)**

| Layer | Mechanism | Scope |
|---|---|---|
| **External AuthZ** | OpenFGA tuples / webhook authorizer | Workspace-level access decisions |
| **SCAR Discovery** | External access provider → AccessEndpointSlice | Which workspaces a subject can see |

The external provider obtains the access map from the authorization system (FGA tuples, webhook callbacks). There are no CRBs/RBs to read — the external system is the sole source of truth.

**Example flow (Scenario B — Platform Mesh with OpenFGA)**:
1. User `alice@example.com` connects an MCP client to the Platform Mesh gateway.
2. Gateway authenticates alice via OIDC and forwards to FrontProxy with her bearer token.
3. FrontProxy terminates TLS, extracts alice's identity, forwards to the MCP VW.
4. MCP VW handler calls `AccessProvider.ClustersFor("alice", groups)` in-process → the FGA access provider returns 87 workspaces (already org-scoped).
5. Handler builds a `ScopedWorkspaceProvider` with those 87 workspaces and alice's bearer token.
6. Handler creates a stateless MCP server via `mcpserver.NewServer(config, provider)`.
7. MCP client sees 87 workspaces and can operate on their resources via kcp toolset.

No separate MCP proxy, no extra HTTP hop for access resolution, no kubeconfig generation — the MCP server library handles everything in-process.

## Consequences

### Positive

- **Clean upstream/downstream separation**: kcp provides a generic SCAR API usable by any kcp consumer, not just Platform Mesh. This benefits the kcp open-source community and avoids proprietary lock-in.
- **Single API call for discovery**: The SCAR endpoint replaces N SubjectAccessReview calls with a single request, dramatically reducing control plane load at scale. Inside the MCP VW the cost drops to an in-process call.
- **In-memory indexing**: The access graph avoids expensive per-request authorization checks by maintaining a pre-computed index, enabling sub-millisecond lookups.
- **Reuses proven patterns**: Virtual Workspaces, FrontProxy path routing, and APIExport/APIBinding are all battle-tested kcp primitives.
- **Incremental adoption**: The opt-in indexing model (via APIBinding) means workspaces can progressively enable MCP discoverability.
- **kubernetes-mcp-server reuse**: Building on the existing Go-native MCP server as a library avoids duplicating transport, tooling, and Kubernetes client logic, and keeps Platform Mesh on the upstream track (no fork).
- **Single Platform Mesh seam**: Platform Mesh's differences are a gateway deployment and an `AccessProvider` implementation — not a separate MCP server or proxy component.

### Negative

- **New service to operate**: The Access Virtual Workspace is an additional deployment that must be monitored, scaled, and upgraded alongside kcp.
- **Eventual consistency**: The access graph is updated asynchronously from watch events or external sync. There will be a brief window (seconds) between an access change and the SCAR result reflecting it.
- **MVP limitations**: Warrants, Scopes, and external webhook authorizers are explicitly excluded from the initial implementation. Users relying on these features will not get accurate SCAR results until future iterations.
- **Platform Mesh gateway adds a hop**: The MCP-aware gateway adds latency and is another point of failure in the MCP request path. Rate-limit and audit configuration must be tuned.

## Confirmation

The decision will be confirmed when:

1. The Access VW is deployed behind a kcp FrontProxy with the RBAC access provider. A `POST /selfclusteraccessreviews` returns only the workspaces the authenticated user has RBAC bindings in — verified with at least two distinct users.
2. The `AccessProvider` interface is validated by swapping the RBAC provider for a stub/mock provider and confirming the SCAR API returns the stub's results unchanged.
3. The MCP VW is deployed in standalone kcp and an MCP client sees only its authorized workspaces, with kcp toolset operations scoped to those workspaces.
4. In a Platform Mesh deployment with the external access provider, an MCP client session for a user in organization A does not expose workspaces belonging to organization B.

## Implementation Roadmap

### Phase 1: Access VW + RBAC Provider MVP (Weeks 1–3)

Deliver the Access VW with the SCAR API and the RBAC access provider. Opt-in indexing via APIBinding. Basic scale test against the expected workspace count.

### Phase 2: MCP VW (Weeks 3–5)

Build the MCP VW using kubernetes-mcp-server as a Go library. Runs alongside the Access VW in a single binary behind FrontProxy. Ships as a kcp upstream component. Bearer-token passthrough, in-process access resolution, scoped provider.

### Phase 3: Platform Mesh Integration (Weeks 5–7)

Deploy the MCP-aware gateway (agentgateway as reference) in front of FrontProxy with OIDC, rate limiting, and audit. Implement the external/FGA access provider and plug it into the Access VW.

### Phase 4: Hardening (Weeks 7+)

Scale testing, Warrants/Scopes support in the RBAC provider, workspace change notifications (SSE/watch), and follow-ups surfaced by production use.

## More Information

### Prior Art / PoC Findings

A hackathon PoC validated the end-to-end MCP flow against a local kcp setup with agentgateway + Keycloak in front of `kubernetes-mcp-server`. The gateway-side pieces (OIDC, dynamic client registration, bearer passthrough, MCP session handling) work as expected. The PoC also identified the gap this ADR closes: the MCP server used a privileged kubeconfig and did not use the authenticated user's identity for scoping. See [`mcp-poc-findings.md`](./mcp-poc-findings.md) for the detailed writeup.

### References

- [kcp Virtual Workspace Architecture](https://github.com/kcp-dev/kcp/tree/main/staging/src/github.com/kcp-dev/virtual-workspace-framework/architecture.md) — Design patterns for kcp Virtual Workspaces.
- [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server) — The upstream MCP server with kcp toolset support.
- [kubernetes-mcp-server kcp provider (PR #657)](https://github.com/containers/kubernetes-mcp-server/pull/657) — The existing kcp cluster provider this ADR builds on.
- [agentgateway](https://github.com/agentgateway/agentgateway) — Reference implementation of the MCP-aware gateway role for Platform Mesh.
- [ADR 002: Fine-Grained Access Control for APIExport Binding](002-apiexport-binding-access-control.md) — Platform Mesh's FGA model for API access control; the FGA access provider builds on this.
- [ADR 003: User Identifier Claim](003_user-identifier-claim.md) — User identity propagation across the stack; dependency for SCAR and FGA intersection.
- [kcp FrontProxy documentation](https://docs.kcp.io/kcp/main/concepts/front-proxy/) — Identity propagation and path routing used by the Access VW.
- [kcp Authorization documentation](https://docs.kcp.io/kcp/main/concepts/authorization/) — Multi-stage RBAC chain referenced by the RBAC access provider.
