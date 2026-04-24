# ADR 007: MCP Server for kcp and Platform Mesh with Permission-Aware Access

| Status           | Proposed                                                     |
|------------------|--------------------------------------------------------------|
| Date             | 2026-04-09                                                   |
| Decision-makers  | Platform Mesh team, kcp team                                 |
| Issue            | [platform-mesh/backlog#212](https://github.com/platform-mesh/backlog/issues/212), [kcp-dev/kcp#3839](https://github.com/kcp-dev/kcp/issues/3839) |
| Epic             | [platform-mesh/backlog#159](https://github.com/platform-mesh/backlog/issues/159) |

## Context and Problem Statement

Platform Mesh and kcp need an AI-accessible interface that allows LLM-based agents (Claude, Copilot, etc.) to interact with Kubernetes resources managed by kcp. Prior art exists — the [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server) project provides a Go-native MCP server with a kcp cluster provider ([PR #657](https://github.com/containers/kubernetes-mcp-server/pull/657)) supporting recursive workspace discovery and a pluggable `--cluster-provider` architecture — but it lacks a permission-aware discovery mechanism and its library surface is not stable enough to embed. It cannot answer the fundamental question: **"Which workspaces does this user have access to?"**

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

**Selected: Option 2 — Access Virtual Workspace with SCAR API and pluggable access providers, consumed by a bespoke kcp-upstream MCP Virtual Workspace. Both Virtual Workspaces are delivered under the `kcp-dev` contrib repos.**

This option provides a clean, Kubernetes-native API that serves both kcp's open-source community and Platform Mesh's needs. The SCAR API produces exactly the data MCP servers need (cluster names + endpoints), and the Virtual Workspace architecture is a proven pattern within kcp. A direct FGA integration (querying Platform Mesh's OpenFGA store directly from the MCP server) was considered but rejected — it would be Platform Mesh-specific, would not benefit the kcp community, and would couple MCP discovery to the FGA system. Instead, the pluggable provider model handles both authorization backends behind the same API:

- **kcp with native RBAC** → RBAC access provider (watches CRBs/RBs, builds in-memory graph)
- **kcp with external authorizer (e.g., OpenFGA, webhook)** → External access provider (obtains the map from the external system)

The MVP ships the RBAC access provider. The provider interface is defined upfront so that external authorization backends can be supported without changing the SCAR API surface. Platform Mesh deployments using OpenFGA will implement an FGA access provider that builds the workspace map from FGA tuples directly.

MCP is served as a second Virtual Workspace in the same binary — a bespoke implementation of the MCP protocol with a kcp-specific toolset, **not** wrapping `kubernetes-mcp-server` and not committed to any specific third-party MCP SDK. Choice of SDK (or a from-scratch implementation) is an open decision for Issue #2. This keeps the deployment footprint small (one binary, two VWs), removes the need for a separate gateway in standalone kcp (FrontProxy handles identity), and calls the `AccessProvider` in-process — no HTTP round-trip per session. The SCAR REST endpoint on the Access VW still exists for external consumers (CLI tools, dashboards, other services). We deliberately trade upstream reuse for full control over the kcp toolset, request pipeline, and session model.

Platform Mesh deploys the same two VWs and adds an MCP-aware gateway in front of FrontProxy for OIDC, rate limiting, audit, CORS, and MCP session management. The Platform Mesh-specific authorization logic lives entirely inside the FGA access provider; no Platform Mesh-specific MCP server or proxy component is introduced. [agentgateway](https://github.com/agentgateway/agentgateway) is the reference implementation of the gateway role and was validated in the PoC.

Concrete implementation details — Go interfaces, API types, request flows, and deployment YAML — live in the [design doc](./mcp-access-vw-design.md).

### Alternatives considered and rejected

- **Standalone `kubernetes-mcp-server` binary behind an MCP-aware gateway, SCAR over HTTP.** Ships faster because the binary already exists, but adds a permanent extra component, an HTTP round-trip per session init, and couples us to upstream's toolset and lifecycle decisions. Useful only as a dev-or-CI quick-start; not a long-term target.
- **`kubernetes-mcp-server` as an embedded Go library.** Considered as a faster on-ramp for the MCP VW, but its public API is not stable enough to depend on and it would constrain our kcp-specific toolset and request pipeline. We build bespoke instead; `kubernetes-mcp-server` remains useful prior art and a reference for the MCP protocol flow.
- **Platform Mesh-specific fork of any MCP server.** Unnecessary. The bespoke component is a kcp upstream component, and Platform Mesh's differences are absorbed entirely by the `AccessProvider` implementation and the edge gateway — no Platform Mesh-only MCP server needed.

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
│ │ 1..N      │ │  │ extract identity,  │  │ │ Access Provider │ │
│ └───────────┘ │  │ call AccessProvider│◄─┼─│ (in-process)    │ │
│               │  │ in-process, build  │  │ │ RBAC | External │ │
│               │  │ scoped provider,   │  │ └─────────────────┘ │
│               │  │ serve MCP via HTTP │  │                     │
│               │  └────────────────────┘  │ SCAR REST handler   │
│               │                          │ (external consumers)│
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

## Component Responsibilities

### Access Virtual Workspace (kcp — upstream, contrib)

Singleton service registered with FrontProxy. Maintains an **RBAC permission graph** in memory behind a pluggable `AccessProvider` interface. The graph models subjects (users, groups, service accounts), bindings (CRBs, RBs), the roles they reference, and derived access to `LogicalClusters`. SCAR is the first query exposed on the graph ("which workspaces does this subject have view access to?"), but the graph is general enough to support later queries (e.g., "what actions can this subject take on resource X?").

Ships two providers:

- **RBAC provider (MVP)** — watches CRBs/RBs and referenced roles across shards; builds and maintains the graph.
- **External/webhook provider (later)** — integrates with non-RBAC authorizers (e.g., OpenFGA) via push/pull/hybrid sync.

Exposes the SCAR API (`POST …/selfclusteraccessreviews`) for external consumers. Opt-in indexing: only workspaces that bind a designated system `APIExport` are indexed. Warrants and Scopes are out of scope for the MVP.

### MCP Virtual Workspace (kcp — upstream, contrib)

Runs in the same binary as the Access VW. Bespoke MCP protocol implementation with a kcp-specific toolset (SDK choice deferred to Issue #2). Per request: extracts the caller's identity from FrontProxy headers, calls the `AccessProvider` in-process to determine the caller's authorized workspaces, and serves MCP over streamable HTTP with all tool operations scoped to those workspaces. All per-workspace operations use the caller's bearer token (passthrough) — no privileged kubeconfig.

### Permission Model

The MCP VW treats the `AccessProvider` result as ground truth. The mechanism behind it is a deployment concern:

| Deployment | Provider | Authorization source |
|---|---|---|
| kcp-native RBAC | RBAC | CRBs / RBs across shards |
| External authorizer (Platform Mesh / OpenFGA) | External | FGA tuples / webhook callbacks |

These are mutually exclusive in a given deployment; SCAR consumers see the same API shape either way. Details in the [design doc](./mcp-access-vw-design.md).

## Consequences

### Positive

- **Clean upstream/downstream separation**: kcp provides a generic SCAR API usable by any kcp consumer, not just Platform Mesh. This benefits the kcp open-source community and avoids proprietary lock-in.
- **Single API call for discovery**: The SCAR endpoint replaces N SubjectAccessReview calls with a single request, dramatically reducing control plane load at scale. Inside the MCP VW the cost drops to an in-process call.
- **In-memory indexing**: The access graph avoids expensive per-request authorization checks by maintaining a pre-computed index, enabling sub-millisecond lookups.
- **Reuses proven patterns**: Virtual Workspaces, FrontProxy path routing, and APIExport/APIBinding are all battle-tested kcp primitives.
- **Incremental adoption**: The opt-in indexing model (via APIBinding) means workspaces can progressively enable MCP discoverability.
- **Full control over the MCP surface**: A bespoke implementation lets us tailor the kcp toolset, request pipeline, and session model exactly to kcp's needs, without being constrained by an upstream MCP server's design decisions or release cadence.
- **Single Platform Mesh seam**: Platform Mesh's differences are a gateway deployment and an `AccessProvider` implementation — not a separate MCP server or proxy component.

### Negative

- **New service to operate**: The Access Virtual Workspace is an additional deployment that must be monitored, scaled, and upgraded alongside kcp.
- **Eventual consistency**: The access graph is updated asynchronously from watch events or external sync. There will be a brief window (seconds) between an access change and the SCAR result reflecting it.
- **MVP limitations**: Warrants, Scopes, and external webhook authorizers are explicitly excluded from the initial implementation. Users relying on these features will not get accurate SCAR results until future iterations.
- **Platform Mesh gateway adds a hop**: The MCP-aware gateway adds latency and is another point of failure in the MCP request path. Rate-limit and audit configuration must be tuned.
- **No upstream MCP reuse**: We take on the cost of implementing and maintaining an MCP protocol server ourselves — toolset definitions, session handling, and ongoing SDK upgrades. If the MCP protocol evolves, we own the follow-up work.

## Confirmation

The decision will be confirmed when:

1. The Access VW is deployed behind a kcp FrontProxy with the RBAC access provider. A `POST /selfclusteraccessreviews` returns only the workspaces the authenticated user has RBAC bindings in — verified with at least two distinct users.
2. The `AccessProvider` interface is validated by swapping the RBAC provider for a stub/mock provider and confirming the SCAR API returns the stub's results unchanged.
3. The MCP VW is deployed in standalone kcp and an MCP client sees only its authorized workspaces, with kcp toolset operations scoped to those workspaces.
4. In a Platform Mesh deployment with the external access provider, an MCP client session for a user in organization A does not expose workspaces belonging to organization B.

## Implementation Plan

### Phase 1 — Upstream kcp foundation, contrib (Weeks 1–5)

Two parallel tracks, both delivered under `kcp-dev` contrib repositories (exact repo naming decided with kcp maintainers). Tracked as:

- **Issue #1 — [PoC: Access Virtual Workspace with SCAR (RBAC graph + provider)](./issue-1-poc-scar-access-vw.md).** Design and build the **RBAC permission graph** (subjects, bindings, roles, group membership, LogicalClusters); implement the `AccessProvider` interface and the RBAC provider on top of the graph, stand up the Access VW behind FrontProxy and expose the SCAR REST API.
- **Issue #2 — [Build MCP Virtual Workspace as bespoke kcp component](./issue-2-mcp-vw-bespoke.md).** Build the MCP VW bespoke — not neceserily wrapping `kubernetes-mcp-server`. Runs alongside the Access VW. In-process `AccessProvider` call, `WorkspaceScope` per request, bearer-token passthrough, kcp-specific toolset. End-to-end MCP client verification.

The two tracks run in parallel. Issue #2 depends on Issue #1 for the `AccessProvider` interface and a working (even stubbed) provider — integration can start as soon as the interface lands. The RBAC graph design in Issue #1 is a standalone deliverable that should be reviewed with kcp maintainers before implementation.

### Phase 2 — Platform Mesh integration (Weeks 5–7)

Deploy the MCP-aware gateway (agentgateway as reference) in front of FrontProxy with OIDC, rate limiting, and audit. Implement the external/FGA access provider and plug it into the Access VW.

### Phase 3 — Hardening (Weeks 7+)

Scale testing, Warrants/Scopes support in the RBAC provider, workspace change notifications (SSE/watch), and follow-ups surfaced by production use.

## More Information

### Prior Art / PoC Findings

A hackathon PoC validated the end-to-end MCP flow against a local kcp setup with agentgateway + Keycloak in front of `kubernetes-mcp-server`. The gateway-side pieces (OIDC, dynamic client registration, bearer passthrough, MCP session handling) work as expected. The PoC also identified the gap this ADR closes: the MCP server used a privileged kubeconfig and did not use the authenticated user's identity for scoping. See [`mcp-poc-findings.md`](./mcp-poc-findings.md) for the detailed writeup.

### References

- [kcp Virtual Workspace Architecture](https://github.com/kcp-dev/kcp/tree/main/staging/src/github.com/kcp-dev/virtual-workspace-framework/architecture.md) — Design patterns for kcp Virtual Workspaces.
- [kubernetes-mcp-server](https://github.com/containers/kubernetes-mcp-server) — Prior art / reference. Not used as a library or binary in the chosen design.
- [kubernetes-mcp-server kcp provider (PR #657)](https://github.com/containers/kubernetes-mcp-server/pull/657) — Reference for how an MCP-over-kcp integration has been modelled elsewhere.
- [agentgateway](https://github.com/agentgateway/agentgateway) — Reference implementation of the MCP-aware gateway role for Platform Mesh.
- [ADR 002: Fine-Grained Access Control for APIExport Binding](002-apiexport-binding-access-control.md) — Platform Mesh's FGA model for API access control; the FGA access provider builds on this.
- [ADR 003: User Identifier Claim](003_user-identifier-claim.md) — User identity propagation across the stack; dependency for SCAR and FGA intersection.
- [kcp FrontProxy documentation](https://docs.kcp.io/kcp/main/concepts/front-proxy/) — Identity propagation and path routing used by the Access VW.
- [kcp Authorization documentation](https://docs.kcp.io/kcp/main/concepts/authorization/) — Multi-stage RBAC chain referenced by the RBAC access provider.
