# RFC 002: Terminal Controller Manager

| Status  | Proposed   |
|---------|------------|
| Author  | @nexus49   |
| Created | 2026-02-26 |

## Summary

Terminal Controller Manager is a Kubernetes controller that provides browser-based terminal sessions to KCP workspaces. It watches cluster-scoped `Terminal` CRs across KCP workspaces via multicluster-runtime and creates ephemeral pods running [ttyd](https://tsl0922.github.io/ttyd/) with kubectl and KCP plugins on a runtime cluster. User OIDC tokens are never stored in Kubernetes -- they are passed at connection time through the ttyd WebSocket and validated against the user specified in the Terminal spec.

## Motivation

1. **Developer productivity** -- Platform-mesh users need CLI access to their KCP workspaces. Today this requires local installation and configuration of kubectl, KCP plugins (`kubectl-kcp`, `kubectl-ws`), kubeconfig with workspace-scoped API endpoints, and proper CA certificates. This is a non-trivial barrier, especially for users unfamiliar with KCP's workspace model.

2. **Security** -- Terminal sessions should authenticate users via their existing OIDC tokens without persisting credentials in Kubernetes. In a multi-tenant platform, avoiding credential storage in etcd reduces the attack surface and eliminates the need for active credential cleanup.

3. **KCP-native UX** -- KCP workspaces have unique properties: workspace paths, workspace-scoped APIs, virtual API servers. A purpose-built terminal can ship KCP-specific tooling out of the box, including a KCP-aware [k9s](https://k9scli.io/) fork that understands workspace navigation.

## Context and Problem Statement

Platform-mesh is built on [KCP](https://github.com/kcp-dev/kcp) (Kubernetes Control Plane). Users manage their resources through two fully supported clients: the portal UI and kubectl. Both are first-class interfaces to the platform, and making the CLI approachable to all users is a strategic goal.

Today, using kubectl against a KCP workspace requires local setup: installing kubectl, KCP plugins (`kubectl-kcp`, `kubectl-ws`), and downloading the OIDC kubeconfig. The barrier of entry is high, especially for users who primarily work through the portal and only occasionally need CLI access.

A browser-based terminal that comes pre-configured with the right tools and workspace context removes this barrier entirely. Users can go from the portal to a working kubectl session in seconds, without any local setup.

**Core problem: How do we provide secure, ephemeral, browser-based terminal access to KCP workspaces that requires zero local setup and avoids storing user credentials in Kubernetes?**

## Proposal

### Split-Cluster Architecture

The controller operates across two distinct planes:

```
                     KCP (Virtual Workspaces)              Runtime Cluster
                    +------------------------+            +-----------------------+
                    | User's Workspace       |            | Terminal Controller   |
                    |  +- Terminal CR        |<-- watch --+  Reconciliation Loop  |
                    |                        |            |                       |
                    +------------------------+            | terminal-sessions ns  |
                                                          |  +- Pod (ttyd+kubectl)|
                                                          |  +- Service (ClusterIP)|
                                                          |  +- HTTPRoute         |
                                                          +-----------+-----------+
                                                                      |
Browser (xterm.js) ---- WebSocket (wss) via Gateway API --------------+
  Token as first stdin message
```

| Plane | Resources | Client |
|-------|-----------|--------|
| KCP (virtual workspaces) | Terminal CRs | multicluster-runtime manager via APIExport provider |
| Runtime cluster | Pods, Services, HTTPRoutes | standalone controller-runtime client (in-cluster config) |

The multicluster manager connects to KCP to discover and watch Terminal CRs across all workspaces via the [APIExport](https://docs.kcp.io/kcp/main/concepts/apis/) mechanism. A separate standalone client manages runtime resources. This separation is necessary because the multicluster-runtime manager provides clients scoped to KCP workspace resources, while pods, services, and routes must be created on the physical cluster.

### Terminal Custom Resource

```yaml
apiVersion: terminal.platform-mesh.io/v1alpha1
kind: Terminal
metadata:
  name: my-terminal
spec:
  user: user@example.com  # OIDC subject of the terminal owner
```

**Design decisions:**

- **Cluster-scoped** -- Matches KCP's workspace model where the workspace path provides isolation. The workspace context is implicit from where the CR is created.
- **Minimal spec** -- The spec contains only the `user` field (OIDC subject) to identify the terminal owner. All other configuration (image, namespace, lifetime, gateway settings) is controller-side, preventing users from escalating privileges by specifying custom images or target URLs. Additional fields can be exposed in the API if needed in the future (e.g. idle timeout, shell preference).
- **Mutating webhook** -- A mutating admission webhook enforces that `spec.user` is always set to the currently authenticated user. This prevents users from creating terminals on behalf of other users, regardless of what value they provide in the spec.
- **Status fields**: `phase` (Pending/Creating/Ready/Failed/Terminating), `sessionId` (UUID for non-guessable URL), `podName`, `workspacePath`, standard `conditions`.

### Reconciliation Pipeline

Four subroutines executed in order via the [golang-commons lifecycle manager](https://github.com/platform-mesh/golang-commons):

```
Normal reconciliation:    Lifetime --> Pod --> Service --> HTTPRoute
Finalization (reverse):   HTTPRoute --> Service --> Pod --> Lifetime
```

1. **LifetimeSubroutine** -- Checks terminal age against configured lifetime (default 2h). Deletes expired Terminal CRs. Uses the manager's cache sync period to trigger periodic checks. No finalizer needed.

2. **PodSubroutine** -- Resolves the workspace URL and CA certificate. Generates a UUID session ID. Creates an ephemeral pod with ttyd on port 8080, injecting workspace URL, CA data, and the expected user ID (from `spec.user`) as environment variables. Requeues until the pod reaches `Running` phase, then sets Terminal status to `Ready`.

3. **ServiceSubroutine** -- Creates a ClusterIP Service targeting the terminal pod on port 8080. Uses `CreateOrUpdate` for idempotency.

4. **HTTPRouteSubroutine** -- Creates a Gateway API HTTPRoute with path `/terminals/{sessionId}`. Applies a URL rewrite filter stripping the prefix to `/` (ttyd expects requests at root). References the platform's existing gateway.

Each subroutine (except Lifetime) registers a finalizer. On Terminal deletion, finalization runs in reverse order to ensure clean teardown: route removed first so traffic stops, then service, then pod.

### Connection Flow and Token Handling

```
 Browser                         Gateway            Terminal Pod
    |                               |                    |
    | 1. Create Terminal CR in KCP workspace             |
    |                               |                    |
    |   [Controller reconciles: creates Pod, Service, HTTPRoute]
    |   [Terminal status.phase = Ready, status.sessionId = UUID]
    |                               |                    |
    | 2. WebSocket connect          |                    |
    |   /terminals/{sessionId}      |                    |
    +------------------------------>| URL rewrite -> /   |
    |                               +------------------->| ttyd
    |                               |                    | -> setup.sh
    | 3. Send OIDC token            |                    |    (waiting
    |   (first stdin message)       |                    |     on stdin)
    +------------------------------>+------------------->|
    |                               |                    |
    |                               | 4. Validate token  |
    |                               |    JWT sub == spec.user?
    |                               |    Create kubeconfig in /tmp
    |                               |                    |
    | 5. Interactive shell ready    |                    |
    |<------------------------------+<-------------------+
    |                               |                    |
    | 6. kubectl, k9s, etc.         |                    |
    +------------------------------>+------------------->|
    |                               |                    |
    |   [After 2h or user deletes Terminal CR]           |
    |   [Finalizers: HTTPRoute -> Service -> Pod]        |
```

The key innovation is **step 3**: the OIDC token is sent as the first stdin line through the ttyd WebSocket. The `setup.sh` script inside the pod:

1. Reads the token from stdin (30-second timeout)
2. Validates the JWT signature and expiry
3. Extracts the `sub` claim and compares against the `EXPECTED_USER_ID` environment variable (set from `spec.user`)
4. On validation failure or user mismatch: exits with error (connection terminated)
5. On success: writes kubeconfig to `/tmp/kubeconfig` with the token, workspace URL, and cluster CA
6. Starts an interactive bash shell with kubectl configured

### Security Model

| Concern | Design Decision | Rationale |
|---------|----------------|-----------|
| Token storage | Never stored in Kubernetes (no Secrets, ConfigMaps, or CR fields) | Eliminates etcd as an attack vector for user tokens |
| Token transport | Sent as first WebSocket stdin message over TLS | Token only traverses encrypted channels |
| Token at rest | Only in tmpfs inside pod (`/tmp/kubeconfig`, `/tmp/token`) | Memory-backed, never written to disk, gone when pod terminates |
| Identity verification | JWT `sub` claim validated against `spec.user` | Prevents session hijacking even if session URL is guessed |
| Session URLs | UUID-based (`/terminals/{sessionId}`) | Non-guessable, prevents enumeration |
| Pod security | Non-root (uid 1000), read-only rootfs, drop ALL capabilities, seccomp RuntimeDefault | Defense in depth |
| Connection limit | ttyd `--max-clients 1` | One session per pod, no session sharing |
| Lifetime | Automatic deletion after 2h (configurable) or when the user closes the terminal in the browser | Bounds exposure window, forces token rotation |
| RBAC | Separate KCP and runtime clients with minimal permissions | Principle of least privilege per API boundary |

### Terminal Pod Image

Multi-stage build producing an Alpine 3.20-based image with:

- **ttyd** -- WebSocket terminal server (`--port 8080 --writable --max-clients 1`)
- **kubectl** -- Latest stable release
- **kubectl-kcp, kubectl-ws** -- KCP workspace navigation plugins
- **k9s** -- Terminal UI for Kubernetes
- **bash, curl, jq, vim** -- Essential CLI tools

The pod mounts two emptyDir volumes (`/tmp` for kubeconfig/token, `/home/terminal` for user workspace) over a read-only root filesystem.

### Gateway API Integration

Gateway API is used instead of Ingress because:

- WebSocket support is first-class in HTTPRoute
- Platform-mesh already deploys a Gateway API-compatible gateway
- Path-based routing with prefix rewrite maps cleanly to per-session URLs
- Each terminal gets its own HTTPRoute, allowing independent lifecycle management

The controller creates HTTPRoutes referencing the platform's existing gateway (configurable via `--gateway-name` and `--gateway-namespace` flags).

### Platform Integration

- **APIExport** (`terminal.platform-mesh.io`) exposes the Terminal API to all KCP workspaces
- **Two container images**: controller (Go binary) and terminal (Alpine with tools)
- **Helm chart** deployment via platform-mesh-operator (planned for Phase 2)

## Prior Art

### Gardener terminal-controller-manager

[Gardener's terminal-controller-manager](https://github.com/gardener/terminal-controller-manager) provides browser-based terminal access to Gardener Shoot and Seed clusters. It stores ServiceAccount tokens in Kubernetes Secrets and uses a host/target cluster model with service account impersonation. This design does not map well to KCP's workspace abstraction, and persisting OIDC tokens in etcd creates credential sprawl in a multi-tenant platform. The terminal-controller-manager takes inspiration from Gardener's approach but uses connection-time token injection to avoid storing credentials in Kubernetes.

## Additional Design Decisions

### ttyd in pod over WebSocket proxy in the controller

Each terminal pod runs ttyd directly and is exposed via Gateway API HTTPRoute, rather than having the controller maintain a WebSocket endpoint and proxy connections to pod exec. This means the controller is only needed for reconciliation -- active sessions survive controller restarts and the controller is not a bottleneck for active connections.

### Cluster-scoped over namespace-scoped Terminal CRs

Terminal is a cluster-scoped resource because KCP workspaces do not have meaningful namespaces for this purpose. Cluster-scoped aligns with KCP's model where workspace identity is the primary isolation boundary.

### ttyd over kubectl exec

ttyd provides a proper terminal server with WebSocket support, terminal resizing, and connection management out of the box. Using kubectl exec would require the controller to implement WebSocket proxy state management, resize event handling, and Kubernetes exec API specifics.

## Drawbacks

1. **Two container images to maintain** -- The terminal image pins external tools (kubectl, KCP plugins, k9s) that need regular updates independent of the controller.

2. **Token expiry during sessions** -- If the OIDC token expires during a long session, kubectl commands fail with 401. The architecture supports token refresh via a `__update_token__` shell function but frontend integration is not yet complete.

3. **No cross-cluster OwnerReference** -- Pods on the runtime cluster cannot reference Terminal CRs on KCP via OwnerReference. Cleanup relies entirely on finalizers. If the controller is down when a Terminal is deleted, cleanup requires the controller to restart and process the finalizer queue.

4. **Gateway API requirement** -- The design requires a Gateway API-compatible gateway deployed in the runtime cluster.

## References

- [Gardener terminal-controller-manager](https://github.com/gardener/terminal-controller-manager) -- Inspiration for the project
- [ttyd](https://tsl0922.github.io/ttyd/) -- WebSocket terminal server used in terminal pods
- [xterm.js](https://xtermjs.org/) -- Terminal emulator for frontend integration
- [KCP](https://github.com/kcp-dev/kcp) -- Kubernetes Control Plane
- [multicluster-runtime](https://github.com/kubernetes-sigs/multicluster-runtime) -- Multi-cluster controller framework
- [multicluster-provider](https://github.com/kcp-dev/multicluster-provider) -- KCP APIExport provider for multicluster-runtime
- [Gateway API](https://gateway-api.sigs.k8s.io/) -- Kubernetes networking API
- [platform-mesh/golang-commons](https://github.com/platform-mesh/golang-commons) -- Shared lifecycle manager, logging, configuration
