# RFC 005: Portal Backend Proxy for Provider Micro-Frontends

| Status  | Proposed                                                     |
|---------|--------------------------------------------------------------|
| Author  | @mjudeikis                                                   |
| Created | 2026-03-06                                                   |

## Summary

Provider micro-frontends should be served through the portal backend as a reverse proxy, under the same domain as the portal itself. Providers should not need their own public domains, DNS entries, or TLS certificates to integrate a UI into Platform Mesh.

## Motivation

While building a sample provider with a custom micro-frontend, we hit a wall: the browser blocks mixed content (HTTPS portal loading HTTP micro-frontend iframes), and fixing that requires every provider to have its own publicly resolvable domain with valid TLS certificates. This creates a high barrier to entry for providers and becomes unmanageable at scale.

Think about it from an enterprise perspective: every new provider domain needs to go through corporate proxy allowlists, DNS delegation, certificate issuance. For more security-aware companies this can take weeks per provider. That's not a viable onboarding story.

The portal should handle this. Backend routing under a single domain, with the platform managing trust internally.

## Context and Problem Statement

Today, `ContentConfiguration` resources point to micro-frontend URLs directly:

```json
{
  "url": "http://localhost:4200/index.html"
}
```

This works in local development (`localhost` is treated as a secure origin by browsers). But in production:

1. **Mixed Content** - The portal runs on HTTPS (`https://console.pm.io`). Provider micro-frontends served over HTTP get blocked by the browser. Even provider UIs served over HTTPS on a different domain face CORS issues.

2. **Certificate Burden** - Each provider needs valid TLS certificates for their UI endpoint. If a provider serves 5 Platform Mesh installations, that's 5 certificates to manage with potentially different lifetimes and issuers.

3. **Domain Management** - Giving subdomains to providers (`provider1.pm.io`, `provider2.pm.io`) doesn't scale. Who manages the DNS? What happens when a provider is removed? Who rotates the certs?

4. **Enterprise Proxy Rules** - Corporate environments often allowlist specific domains. Adding a new provider means updating firewall rules, proxy configs, and security policies. This is a deployment blocker.

### What We Observed

During the sample provider build-out:

- `npm start` locally works fine (browser trusts `localhost`)
- Docker container with nginx—same content, doesn't load (no HTTPS handshake)
- In-cluster service URL (`resource-broker-portal.resource-broker-system.svc:8080`)—blocked as mixed content
- CORS errors when the micro-frontend tries to call GraphQL gateway from a different origin

The current workarounds (adjusting Traefik CORS headers, adding `localhost` exceptions) are local-dev band-aids, not production solutions.

## Proposal

### Portal as Reverse Proxy

The portal backend should act as a reverse proxy for provider micro-frontends. All provider UIs are served under the portal's own domain:

```
https://console.pm.io/providers/cowboys/ui/  -->  backend proxy  -->  cowboys-ui.provider-ns.svc:8080
https://console.pm.io/providers/search/ui/   -->  backend proxy  -->  search-ui.search-system.svc:8080
https://console.pm.io/providers/terminal/ui/ -->  backend proxy  -->  terminal-ui.terminal-system.svc:8080
```

From the browser's perspective, everything is `console.pm.io`. No mixed content, no CORS, no extra domains.

```
┌──────────────────────────────────────────────────────────────────┐
│                         Browser                                  │
│                                                                  │
│   Portal (https://console.pm.io)                                 │
│   ├── Core UI                                                    │
│   ├── /providers/cowboys/ui/  ◄── same origin, no CORS           │
│   ├── /providers/search/ui/   ◄── same origin, no CORS           │
│   └── /providers/terminal/ui/ ◄── same origin, no CORS           │
│                                                                  │
└──────────────────┬───────────────────────────────────────────────┘
                   │ HTTPS (single cert, single domain)
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Portal Backend                                │
│                                                                  │
│   Ingress / Reverse Proxy                                        │
│   ├── /providers/cowboys/ui/*  ──► cowboys-ui.svc (mTLS)        │
│   ├── /providers/search/ui/*   ──► search-ui.svc (mTLS)         │
│   └── /providers/terminal/ui/* ──► terminal-ui.svc (mTLS)       │
│                                                                  │
│   Trust: Platform-issued mTLS certificates                       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Route Registration via ContentConfiguration

The `ContentConfiguration` resource already describes where a micro-frontend lives. Extend it to support backend-proxied URLs:

```yaml
apiVersion: ui.platform-mesh.io/v1alpha1
kind: ContentConfiguration
metadata:
  name: cowboys-ui
  labels:
    ui.platform-mesh.io/entity: core_platform-mesh_io_account
    ui.platform-mesh.io/content-for: wildwest.platform-mesh.io
spec:
  inlineConfiguration:
    content: |-
      {
        "name": "cowboys",
        "luigiConfigFragment": {
          "data": {
            "nodes": [
              {
                "pathSegment": "cowboys",
                "label": "Cowboys",
                "icon": "person-placeholder",
                "url": "/providers/cowboys/ui/index.html",
                "entityType": "main.core_platform-mesh_io_account",
                "loadingIndicator": { "enabled": true }
              }
            ]
          }
        }
      }
    contentType: json
  # Backend proxy target - where the portal routes traffic to
  proxyTarget:
    # In-cluster service
    service:
      name: cowboys-ui
      namespace: cowboys-system
      port: 8080
    # OR external URL (provider runs their own infra)
    # url: "https://ui.cowboys.example.com"
    basePath: /providers/cowboys/ui
```

The portal backend reads `proxyTarget` and sets up the reverse proxy route. The `url` in the Luigi config becomes a relative path—same origin, no CORS, no mixed content.

### Trust Model: Mutual TLS

Trust goes both ways. The platform needs to prove itself to the provider ("this request really comes from PM"), and the provider needs to prove itself to the platform ("this backend is really the cowboys UI, not something impersonating it").

For external providers, this is proper mTLS—both sides present certificates, both sides validate.

#### Platform-to-Provider (client certificate)

When the proxy calls an external provider backend, it presents a client certificate issued by the platform. The provider validates this against the platform's CA to confirm the caller is a trusted PM installation.

The flow:

1. **Provider registers** their UI service (via `ContentConfiguration` with `proxyTarget`)
2. **Platform issues** a client TLS key pair (cert + key), stores it in a Secret
3. **ContentConfiguration status** gets a reference to the Secret and a SHA256 of the certificate
4. **Proxy presents** this client certificate on every request to the provider backend
5. **Provider validates** the certificate against the platform's CA
6. **On rotation**, the platform updates the Secret, bumps the SHA in status, proxy picks up the new cert

#### Provider-to-Platform (server certificate)

The other direction: the provider's backend also needs a certificate that the proxy can validate. This proves the proxy is actually talking to the legitimate provider backend, not something sitting in between.

The provider has two options:

1. **Platform-issued server cert** — The platform issues a server key pair for the provider too. Provider uses it to serve TLS. Platform's proxy trusts only certs signed by its own CA. This is the simplest model—the platform controls the full trust chain.
2. **Provider-managed server cert** — The provider uses their own certificate (e.g., from Let's Encrypt or their corporate CA). The provider registers their CA's public cert in the `ContentConfiguration` spec so the proxy knows what to trust. We do not want to suppor this in the initial implementation due to complexity, but it could be added later for providers who have strict requirements around their own certificate management.

```yaml
spec:
  proxyTarget:
    url: "https://ui.cowboys.example.com"
    basePath: /providers/cowboys/ui
    # Option 1: platform issues both sides (default)
    tls:
      mode: platform-managed
    # Option 2: provider manages their own server cert
    # tls:
    #   mode: provider-managed
    #   trustedCA:
    #     secretRef:
    #       name: cowboys-ca-cert
    #       namespace: platform-mesh-system
```

Default should be `platform-managed`—the platform issues both client and server certs. Lower barrier for providers, single trust chain, the platform controls rotation for both sides.

#### Status Tracking

The `ContentConfiguration` status tracks both certificate sets:

```yaml
apiVersion: ui.platform-mesh.io/v1alpha1
kind: ContentConfiguration
metadata:
  name: cowboys-ui
spec:
  proxyTarget:
    url: "https://ui.cowboys.example.com"
    basePath: /providers/cowboys/ui
status:
  proxyTLS:
    # Client cert: platform presents this to the provider
    clientCertificate:
      secretRef:
        name: cowboys-ui-client-mtls
        namespace: platform-mesh-system
      certificateSHA256: "a1b2c3d4e5..."
      expiresAt: "2026-06-06T00:00:00Z"
    # Server cert: provider presents this to the platform
    serverCertificate:
      secretRef:
        name: cowboys-ui-server-mtls
        namespace: platform-mesh-system
      certificateSHA256: "f6g7h8i9j0..."
      expiresAt: "2026-06-06T00:00:00Z"
  conditions:
    - type: ProxyReady
      status: "True"
    - type: ClientCertificateValid
      status: "True"
    - type: ServerCertificateValid
      status: "True"
```

When either certificate rotates, its SHA changes. The proxy watches for this and reloads. Providers can watch the client cert SHA to track when the platform refreshes its credentials.

The provider doesn't care about the portal's public domain. They serve their UI on a port, the platform handles the rest. Whether the backend is `cowboys-ui.svc:8080` or `https://1.2.3.4:8443`—doesn't matter. Trust lives in the certificates, not the domain.

### Internal vs External Providers

Both cases go through the proxy, but the trust requirements differ:

| Scenario | Provider UI Location | Proxy Target | mTLS Required | Why |
|----------|---------------------|--------------|---------------|-----|
| Co-deployed (in-cluster) | `*.svc`, `*.cluster.local`, `localhost` | Kubernetes service | No | Traffic stays in the cluster network, already trusted |
| External (provider's infra) | `https://ui.provider.com` | External URL | Yes | Traffic leaves the cluster, needs mutual authentication |

For co-deployed providers the proxy connects directly without mTLS. The cluster network is the trust boundary—no need to add overhead for traffic that never leaves.

For external providers mTLS is mandatory in both directions. Each PM installation issues its own client key pair, and either issues or validates the provider's server cert. If a provider serves 5 PM installations, it has 5 client certs to accept and can tell which installation is calling by inspecting the client certificate subject.

### Local Development Story

For local dev, the existing `npm start` flow stays the same. Developers point `url` at `http://localhost:4200` and use the Luigi local-nodes mechanism. No proxy needed—browsers trust localhost.

The proxy only matters for in-cluster and production deployments. The `ContentConfiguration` can carry both:

```yaml
spec:
  # Used in-cluster (proxy route)
  proxyTarget:
    service:
      name: cowboys-ui
      namespace: cowboys-system
      port: 8080
    basePath: /providers/cowboys/ui
  # The inlineConfiguration url uses the proxied path
  inlineConfiguration:
    content: |-
      {
        "luigiConfigFragment": {
          "data": {
            "nodes": [{
              "url": "/providers/cowboys/ui/index.html"
            }]
          }
        }
      }
```

During local dev, the developer overrides `url` to `http://localhost:4200/index.html` via the Luigi console config. No CRD changes needed.

## Alternatives Considered

### Provider-Managed Subdomains

Give each provider a subdomain (`cowboys.console.pm.io`). Provider manages DNS and TLS.

This is what we want to avoid. It doesn't scale, creates certificate management headaches, and gets blocked by enterprise proxy rules. Every new provider means DNS changes, cert issuance, and possibly firewall rule updates.

### Backend-for-Frontend (BFF) per Provider

Each provider deploys a BFF pod that also serves their UI, and the platform routes to it.

This works but pushes complexity to every provider. Each one needs to understand the platform's networking, get the right certs, handle TLS termination. The proxy approach centralizes this once.

### Shared Wildcard Certificate

Use `*.console.pm.io` wildcard cert and give providers subdomains.

Better than per-provider certs, but still requires DNS management per provider and enterprise allowlisting per subdomain. Doesn't solve the fundamental "new domain per provider" problem.

## Drawbacks

- **Man-in-the-Middle** - The proxy terminates and re-initiates TLS. For external providers this means the platform sees all traffic. Needs to be clearly documented from a security/compliance perspective.
- **Latency** - Extra hop through the proxy adds latency to micro-frontend loads. Should be negligible for static assets but worth measuring.
- **Proxy Complexity** - The portal backend takes on routing responsibility. Needs health checks, timeouts, circuit breaking for misbehaving provider UIs.
- **Path Conflicts** - Need clear namespacing (`/providers/{name}/ui/`) to avoid collisions between provider routes and portal routes.

## Open Questions

- How does the proxy interact with the existing virtual-workspaces service that gathers `ContentConfiguration` resources?
- Should the proxy be part of the portal backend or a separate component (dedicated ingress controller)?
- What's the fallback when a provider UI is unreachable? Show an error in the micro-frontend slot, or hide the nav entry entirely?
- How do we handle websocket connections from provider UIs (e.g., terminal provider)?

## References

- [RFC 001: API Provider Metadata and UI Discovery](001_api-providers-and-ui-discovery.md)
- [RFC 004: Core Platform Extendability](004_core-platform-extendability.md)
- [MDN: Mixed Content](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Mixed_content)
- [Luigi Project: Navigation Configuration](https://docs.luigi-project.io/docs/navigation-configuration)
