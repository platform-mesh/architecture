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

**Selected: Option 2 — Access Virtual Workspace with SCAR API and Pluggable Access Providers.**

This option was selected because it provides a clean, Kubernetes-native API that serves both kcp's open-source community and Platform Mesh's needs. The SCAR API produces exactly the data MCP servers need (cluster names + endpoints), and the Virtual Workspace architecture is a proven pattern within kcp. A direct FGA integration (querying Platform Mesh's OpenFGA store directly from the MCP server) was considered but rejected — it would be Platform Mesh-specific, would not benefit the kcp community, and would couple MCP discovery to the FGA system. Instead, the pluggable provider model handles both authorization backends behind the same API:

- **kcp with native RBAC** → RBAC access provider (watches CRBs/RBs, builds in-memory graph)
- **kcp with external authorizer (e.g., OpenFGA, webhook)** → External access provider (obtains the map from the external system)

The MVP ships the RBAC access provider. The provider interface is defined upfront so that external authorization backends can be supported without changing the SCAR API surface. Platform Mesh deployments using OpenFGA will implement an FGA access provider that builds the workspace map from FGA tuples directly.

For how the MCP server consumes the SCAR API, this ADR presents **three deployment approaches**:

- **Deployment Option A — Standalone Binary**: Run `kubernetes-mcp-server` as a standalone binary with SCAR integration, behind an MCP-aware gateway that handles auth, rate limiting, and audit logging. Best for quick adoption, environments where the MCP server lifecycle should be independent of kcp, or developer/CI setups.
- **Deployment Option B — Bespoke VW Component (kcp upstream)**: Use `kubernetes-mcp-server` as a Go library inside a Virtual Workspace handler behind FrontProxy. The MCP VW calls the Access Provider in-process — no HTTP round-trip. This is a kcp upstream component, usable by any kcp consumer. For standalone kcp the gateway is not needed; for Platform Mesh an MCP-aware gateway sits in front of FrontProxy for OIDC auth, rate limiting, and observability.
- **Deployment Option C — Platform Mesh MCP Server**: A dedicated MCP server for Platform Mesh, either forked from `kubernetes-mcp-server` or built from scratch. Allows deep integration with Platform Mesh-specific concerns (organizational scoping, FGA-aware access, custom toolsets) without being constrained by upstream API compatibility. Deployed behind an MCP-aware gateway, same as Option A.

All three share the same Access VW foundation and SCAR API.


### Architecture Overview

#### Deployment Option A: Standalone Binary with MCP-Aware Gateway

The MCP server runs as a standalone `kubernetes-mcp-server` binary. An MCP-aware gateway (e.g., an AI gateway with MCP protocol support) sits at the edge handling OIDC authentication, rate limiting, audit logging, and identity propagation. The MCP server discovers workspaces by calling the SCAR API over HTTP.

```
┌────────────────────────────────────────────────────────────────┐
│                     MCP Client (LLM Agent)                     │
│                     (Claude, Copilot, etc.)                    │
└───────────────────────────┬────────────────────────────────────┘
                            │ MCP Protocol (streamable HTTP)
                            ▼
┌────────────────────────────────────────────────────────────────┐
│                     MCP-Aware Gateway                          │
│       (OIDC auth, rate limiting, audit, CORS, identity         │
│        propagation, MCP session management)                    │
└───────────────────────────┬────────────────────────────────────┘
                            │ Bearer token passthrough
                            ▼
┌────────────────────────────────────────────────────────────────┐
│              kubernetes-mcp-server binary                      │
│              --cluster-provider=kcp                            │
│                                                                │
│  On session init:                                              │
│  1. Call SCAR API (HTTP) → get accessible workspaces           │
│  2. Build scoped kcp provider with workspace list              │
│  3. Serve MCP tools scoped to those workspaces                 │
└──────────┬─────────────────────────────┬───────────────────────┘
           │                             │
           │ SCAR HTTP call              │ kube API calls
           ▼                             ▼
┌─────────────────────┐      ┌───────────────────────────────────┐
│  kcp FrontProxy     │      │  kcp FrontProxy                   │
│                     │      │                                   │
│  /services/access/* │      │  /clusters/{name}/*               │
└─────────┬───────────┘      └───────────┬───────────────────────┘
          │                              │
          ▼                              ▼
┌─────────────────────┐      ┌───────────────────────────────────┐
│  Access VW          │      │  kcp Shards                       │
│                     │      │  ┌───────────┐                    │
│ ┌─────────────────┐ │      │  │ Workspace │                    │
│ │ Access Provider │ │      │  │ 1..N      │                    │
│ │ RBAC | External │ │      │  └───────────┘                    │
│ └─────────────────┘ │      │                                   │
│                     │      │                                   │
│ SCAR REST Handler   │      │                                   │
└─────────────────────┘      └───────────────────────────────────┘
```

**Key characteristics**: The MCP server is a separate process. It calls the SCAR endpoint over HTTP (one round-trip per session init). The gateway is a separate deployment that sits in front. This gives you three components to operate (gateway + MCP binary + Access VW) but each can be scaled and versioned independently. The gateway handles MCP protocol concerns (session management, CORS, OAuth discovery) natively — no custom proxy code required.

A PoC of this approach has been validated using an MCP-aware gateway with OIDC (Keycloak), dynamic client registration for MCP SDK clients, and bearer token passthrough to the MCP server.

#### Deployment Option B: Bespoke VW Component with MCP as Library

The solution is built as a **bespoke kcp component** using kubernetes-mcp-server as a Go library (not a standalone binary). Two Virtual Workspaces run in a single binary behind kcp's FrontProxy:

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
│  Identity: X-Remote-User, X-Remote-Group, X-Remote-Extra-*    │
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

For **standalone kcp** deployments (no Platform Mesh), the gateway is not needed — FrontProxy handles identity, and the MCP VW handles everything else.

For **Platform Mesh** deployments, an MCP-aware gateway sits in front of FrontProxy, providing OIDC authentication, rate limiting, audit logging, and observability. The gateway is required here — Platform Mesh users authenticate via an OIDC provider, and the gateway handles the OAuth flow before passing the identity through to FrontProxy:

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────┐
│  MCP Client      │────▶│  MCP-Aware       │────▶│  kcp         │
│  (LLM Agent)     │     │  Gateway         │     │  FrontProxy  │
│                  │     │  (OIDC, rate     │     │              │
│                  │     │   limit, audit)  │     │  ┌─────────┐ │
│                  │     │                  │     │  │ MCP VW  │ │
│                  │     │                  │     │  │ Access  │ │
│                  │     │                  │     │  │ VW      │ │
└──────────────────┘     └──────────────────┘     └──┴─────────┴─┘
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
3. Access VW looks up the subject in the RBAC graph and returns matching `AccessEndpointSlice` entries.

**Request flow**:
1. MCP server sends `POST /services/access-virtual-workspace/apis/access.kcp.io/v1alpha1/selfclusteraccessreviews` with Bearer token or client certificate.
2. FrontProxy terminates TLS, extracts user identity, forwards to Access VW with `X-Remote-User`, `X-Remote-Group`, `X-Remote-Extra-*` headers.
3. Access VW calls `AccessProvider.ClustersFor(user, groups)` and returns matching `AccessEndpointSlice` entries.

### 2. MCP Server — Deployment Option A: Standalone Binary

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

**MCP-Aware Gateway**: Instead of building a bespoke proxy, this approach uses an MCP-aware gateway at the edge. The gateway handles:

- **OIDC authentication**: Validates tokens against an identity provider, supports dynamic client registration for MCP SDK clients (per the MCP OAuth spec).
- **Identity propagation**: Passes the authenticated user's bearer token through to the MCP server (`backendAuth: passthrough`).
- **Rate limiting**: Per-user/org request throttling.
- **Audit/observability**: MCP-protocol-aware logging and metrics (tool calls, session lifecycle).
- **CORS and session management**: MCP session ID propagation, cross-origin support for web-based clients.

This eliminates the need for a custom proxy — off-the-shelf MCP-aware gateways already provide these capabilities.

**Deployment**: Three separate components:

```yaml
# 1. Access VW — behind FrontProxy
pathMappings:
  - path: /services/access-virtual-workspace
    backend: https://access-vw.kcp-system.svc:6443

# 2. kubernetes-mcp-server — standalone Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-mcp-server
spec:
  template:
    spec:
      containers:
        - name: mcp-server
          args: ["--cluster-provider=kcp", "--scar-endpoint=https://kcp.example.com/services/access-virtual-workspace"]

# 3. MCP-Aware Gateway — edge Deployment
# Routes MCP clients → kubernetes-mcp-server
# Handles OIDC, rate limiting, audit, CORS
```

### 3. MCP Server — Deployment Option B: Bespoke VW Component (MCP as Library)

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
    // Find the endpoint, build a rest.Config with bearer token.
    ...
}
```

**Why this replaces the separate gateway**: Authentication comes from FrontProxy (free). Access scoping comes from the shared AccessProvider (in-process). MCP serving comes from the library. Rate limiting and audit logging are handled as HTTP middleware on the VW handler. There is no need for a separate gateway component.

**Deployment**: Both VWs (MCP + Access) run in the same binary, registered with FrontProxy via PathMapping:

```yaml
pathMappings:
  - path: /services/mcp
    backend: https://mcp-access-vw.kcp-system.svc:6443/mcp
  - path: /services/access-virtual-workspace
    backend: https://mcp-access-vw.kcp-system.svc:6443/access
```

### 4. MCP Server — Deployment Option C: Platform Mesh MCP Server

Platform Mesh may require an MCP server with deeper integration into its own stack than what upstream `kubernetes-mcp-server` provides. This could be:

- A **fork** of `kubernetes-mcp-server` with Platform Mesh-specific extensions (organizational context in tool responses, FGA-aware toolsets, custom resource types).
- A **new MCP server** built from scratch using the Go MCP SDK, purpose-built for Platform Mesh's multi-tenant model.

In either case, the MCP server:

- Calls the SCAR API over HTTP (same as Deployment Option A) to discover accessible workspaces.
- Is deployed behind an MCP-aware gateway for OIDC auth, rate limiting, and audit logging.
- Can embed Platform Mesh-specific knowledge (org hierarchy, account context, custom toolsets) without being constrained by upstream compatibility.

**When to choose this over Option A**: When Platform Mesh needs MCP toolsets that don't belong upstream (e.g., tools that expose organizational structure, FGA policies, or Platform Mesh-specific resources), or when the fork divergence from upstream becomes too costly to maintain.

**Deployment**: Same topology as Option A (gateway + MCP binary + Access VW), with the Platform Mesh MCP server replacing the upstream binary.

### 5. Permission Model

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

**Example flow (Scenario B — Platform Mesh with OpenFGA, Deployment Option B)**:
1. User `alice@example.com` connects an MCP client pointing at `https://kcp.example.com/services/mcp`.
2. FrontProxy terminates TLS, extracts alice's identity, forwards to the MCP VW.
3. MCP VW handler calls `AccessProvider.ClustersFor("alice", groups)` in-process → the FGA access provider returns 87 workspaces (already org-scoped).
4. Handler builds a `ScopedWorkspaceProvider` with those 87 workspaces and alice's bearer token.
5. Handler creates a stateless MCP server via `mcpserver.NewServer(config, provider)`.
6. MCP client sees 87 workspaces and can operate on their resources via kcp toolset.

No separate gateway, no extra HTTP hop for access resolution, no kubeconfig generation — the MCP server library handles everything in-process.

**Same scenario with Deployment Option A**: Alice's MCP client connects to the MCP-aware gateway, which authenticates her via OIDC and forwards to the standalone `kubernetes-mcp-server` binary with her bearer token. The binary calls the SCAR endpoint over HTTP — the FGA access provider returns the same 87 workspaces. The binary scopes its kcp provider to those workspaces. The end result is the same; the difference is the extra HTTP hop and the separate gateway component.

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

### Deployment Option Comparison

| | Option A: Standalone Binary | Option B: Bespoke VW | Option C: PM MCP Server |
|---|---|---|---|
| **Components** | Gateway + MCP binary + Access VW | MCP VW + Access VW (single binary); gateway for PM | Gateway + PM MCP binary + Access VW |
| **SCAR call** | HTTP round-trip | In-process | HTTP round-trip |
| **Gateway** | Required | Not needed for kcp; required for PM | Required |
| **Upstream reuse** | kubernetes-mcp-server as-is | kubernetes-mcp-server as library | Fork or new build |
| **PM-specific toolsets** | No | No | Yes |
| **Best for** | Quick start, dev/CI, any kcp consumer | kcp upstream, production PM with tight integration | PM with domain-specific needs |

## Confirmation

The decision will be confirmed when:

1. The Access VW is deployed behind a kcp FrontProxy with the RBAC access provider. A `POST /selfclusteraccessreviews` returns only the workspaces the authenticated user has RBAC bindings in — verified with at least two distinct users.
2. The `AccessProvider` interface is validated by swapping the RBAC provider for a stub/mock provider and confirming the SCAR API returns the stub's results unchanged.
3. At least one deployment option (A, B, or C) is validated end-to-end: an MCP client sees only its authorized workspaces and can perform kcp toolset operations scoped to those workspaces.
4. In a Platform Mesh deployment with the external access provider, an MCP client session for a user in organization A does not expose workspaces belonging to organization B.

## Implementation Roadmap

### Phase 1: Access VW + RBAC Provider MVP (Weeks 1-3)

Shared foundation for both deployment approaches. Delivers the Access VW with the SCAR API and the RBAC access provider.

### Phase 2a: Standalone Binary — SCAR Integration + Gateway (Weeks 2-4)

Upstream a SCAR-aware mode into the kubernetes-mcp-server kcp provider. Deploy with an MCP-aware gateway for auth and observability. This path delivers MCP access faster — the binary already exists, it just needs SCAR integration.

### Phase 2b: Bespoke VW — MCP as Library (Weeks 3-5)

Build the MCP VW using kubernetes-mcp-server as a Go library. Runs alongside the Access VW in a single binary behind FrontProxy. This is a kcp upstream component — delivers deeper integration and eliminates the gateway.

### Phase 2c: Platform Mesh MCP Server — Evaluate Fork vs Build (Weeks 3-5)

Evaluate whether to fork kubernetes-mcp-server or build a dedicated Platform Mesh MCP server. Decision depends on how much Platform Mesh-specific tooling is needed and how far the fork would diverge from upstream.

### Phase 3: External/FGA Access Provider (Weeks 5-7)

Implement the external/webhook access provider for Platform Mesh deployments using OpenFGA. Applies to both deployment approaches — the provider plugs into the same Access VW.

### Phase 4: Hardening and Future Iterations (Weeks 7+)

Scale testing, Warrants/Scopes support in the RBAC provider, workspace change notifications (SSE/watch), and convergence on a single recommended deployment model based on production experience.

## More Information

### Prior Art / PoC Findings

Hackathon team ran a local-setup experiment with agentgateway + Keycloak in front of `kubernetes-mcp-server` (`--cluster-provider=kcp --stateless`). The full MCP flow works: Claude Code connects to the gateway, authenticates via Keycloak OIDC, and talks to kcp workspaces through the MCP server.

A few things we could learn about the OIDC/gateway setup:

- MCP SDK clients auto-discover the OAuth config via `/.well-known/oauth-protected-resource` and register dynamically — no manual client setup needed.
- Keycloak's dynamic client registration requires explicit config: trusted hosts must include the gateway hostname, and the allowed scopes policy must list every scope from the OIDC discovery endpoint (MCP SDKs request all of `scopes_supported`).
- Token validation uses a shared `agent-access` audience scope with an audience mapper. The gateway checks the audience claim; the bearer token is passed through to the MCP server as-is.
- TLSRoute (Gateway API) routes by hostname to agentgateway. Gateway terminates TLS, re-encrypts to the backend.

**The gap**: the MCP server in the PoC uses a privileged kubeconfig (a `providerConnection` secret). It sees all workspaces regardless of who the caller is. The gateway authenticates the user, but the MCP server doesn't use that identity for scoping — it discovers everything with the service account. The SCAR API proposed here closes this: the MCP server uses the caller's bearer token to call SCAR, gets back only their workspaces, and the privileged kubeconfig is no longer needed for discovery.

### Open Questions

### References

- [kcp Virtual Workspace Architecture](https://github.com/kcp-dev/kcp/tree/main/staging/src/github.com/kcp-dev/virtual-workspace-framework/architecture.md) — Design patterns for kcp Virtual Workspaces.
- [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server) — The upstream MCP server with kcp toolset support.
- [kubernetes-mcp-server kcp provider (PR #657)](https://github.com/containers/kubernetes-mcp-server/pull/657) — The existing kcp cluster provider this ADR builds on.
- [ADR 002: Fine-Grained Access Control for APIExport Binding](002-apiexport-binding-access-control.md) — Platform Mesh's FGA model for API access control; the MCP proxy's organizational scoping builds on this.
- [ADR 003: User Identifier Claim](003_user-identifier-claim.md) — User identity propagation across the stack; dependency for SCAR and FGA intersection.
- [kcp FrontProxy documentation](https://docs.kcp.io/kcp/main/concepts/front-proxy/) — Identity propagation and path routing used by the Access VW.
- [kcp Authorization documentation](https://docs.kcp.io/kcp/main/concepts/authorization/) — Multi-stage RBAC chain referenced by the RBAC graph controller.
