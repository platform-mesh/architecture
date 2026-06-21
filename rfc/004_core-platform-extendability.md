# RFC 004: Core Platform Extendability - Everything is a Provider

| Status  | Proposed                                                     |
|---------|--------------------------------------------------------------|
| Author  | @mjudeikis                                                   |
| Created | 2026-03-03                                                   |
| Updated | 2026-03-06                                                   |

## Summary

Platform Mesh should treat all capabilities—including core features like Search and Terminal—as providers. This moves us from thinking "Core vs. Provider" to "Core vs. Builtin Providers vs Third-Party Providers", making the platform more modular and preventing core bloat.

## Motivation

We keep adding features directly to core, which creates maintenance burden and tight coupling. By building core features as providers we:

- Force ourselves to improve the provider framework (dogfooding)
- Allow teams to own their scope (Search team maintains search-provider repo)
- Give adopters the modularity they require
- Support regional deployments that need to disable or swap capabilities

## Context and Problem Statement

Right now we don't clearly separate "core" from "provider" functionality. Features like Search and Terminal are being designed with tight integration into `WorkspaceType` initializers. While this works, it's not what WorkspaceTypes were designed for and creates problems:

1. **Single Point of Failure** - If one initializer fails, the whole account provisioning blocks
2. **No Day-2 Operations** - Can't enable/disable features on existing accounts
3. **Blocking over Eventual Consistency** - We block account access waiting for things that could happen async
4. **System Internals Leak** - Risk of exposing system resources to user RBAC manipulation

### How Cloud Platforms Do It

Azure, AWS, GCP all use eventual consistency. A fresh Azure account might show compute unavailable for a few minutes after login. Google Search autocomplete degrades gracefully when backends are slow. Users get access while provisioning continues in the background.

We should do the same—give users UX signals about what's still provisioning rather than blocking them entirely.

## Proposal

### Everything is a Provider

Move away from a monolithic core. Instead, categorize features into tiers:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Platform Mesh                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    The Core (Minimal)                     │  │
│  │  • Workspace Management                                   │  │
│  │  • RBAC                                                   │  │
│  │  • APIExport/APIBinding                                   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                  │
│  ┌───────────────────────────▼───────────────────────────────┐  │
│  │              Core Providers (1st Party)                   │  │
│  │  • Search Provider                                        │  │
│  │  • Terminal Provider                                      │  │
│  │  • Monitoring Provider                                    │  │
│  │  • ... other platform experience features                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                  │
│  ┌───────────────────────────▼───────────────────────────────┐  │
│  │              External Providers (3rd Party)               │  │
│  │  • External providers integrating into system             │  │
│  │  • User-developed providers                               │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Provider Tiers

| Tier | Name | What it is | User Control |
|------|------|------------|--------------|
| 0 | **The Core** | Minimum to run the platform—RBAC, workspace management | None |
| 1 | **Core Providers** | Platform experience features (Terminal, Search, Monitoring). Built by us, but following provider patterns. | Limited via Deny RBAC |
| 2 | **External Providers** | APIs added by users or partners | Full |

This mirrors what Azure did with 1st/2nd/3rd party providers. Everything is "just a provider" but with different trust levels. In some deployments (regulated environments, specific regions) we disabled or swapped 1st party providers as needed.

### The Onion Model

Extensions should be pluggable, not hardcoded:

1. **Toggleable** - Platform admins can enable/disable Search or Terminal per org/account
2. **Separate Repos** - Search team owns `platform-mesh/search-provider`, not core
3. **Dogfooding** - We use the same provider framework we ask 3rd parties to use

### Non-Blocking Account Provisioning

Account creation should hand over immediately. Background work happens async:

```
┌──────────────────────────────────────────────────────────────────┐
│                    Account Creation Flow                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   [User Request] ──► [Create Workspace] ──► [RBAC Setup]         │
│                              │                                   │
│                              ▼                                   │
│                    [Account Ready ✓]  ◄── User can access now    │
│                              │                                   │
│              ┌───────────────┼───────────────┐                   │
│              ▼               ▼               ▼                   │
│       [Search Index]  [Terminal Setup]  [Monitoring]             │
│       (async, 3min)   (async, 1min)     (async, 2min)            │
│              │               │               │                   │
│              ▼               ▼               ▼                   │
│       [Index Ready]   [Terminal Ready] [Monitoring Ready]        │
│                                                                  │
│   UI shows: "Search index building..." with graceful degradation │
└──────────────────────────────────────────────────────────────────┘
```

The key points:

- Don't use `WorkspaceType` as a lifecycle engine for every feature
- If search indexing takes 3 minutes, let users log in anyway and show "Index building..."
- Define shared status conditions so the UI can surface provisioning state consistently
- Providers configure themselves post-creation

### Protecting System Internals

We still need to prevent users from messing with system resources:

| Concern | How we handle it |
|---------|------------------|
| Protect system resources | Deny RBAC on critical 1st party resources |
| Visible but protected | Users can see resources exist, can't delete them |
| Admin override | Org admins can disable providers if needed (e.g., drop a 1TB index to save costs) |

The point is: it should be *possible* for admins to disable things, just guarded. If I'm a startup and don't care about monitoring this week, why shouldn't I be able to turn it off?

### Provider Migration on Existing Workspaces

A key benefit of the provider model is that services can be replaced on existing workspaces without requiring workspace re-creation. For example, when introducing a new observability stack:

1. **Bind** the new provider's API to the workspace
2. **Migrate** data and configuration from old to new provider
3. **Unbind** the old provider's API

This only works if providers do not assume the workspace is in an initializing state to perform their setup. Providers must be designed to configure themselves against workspaces in any lifecycle phase—not just during initial creation. This is another reason to move away from `WorkspaceType` initializers: they only run at creation time, making day-2 service replacement impossible.

### Config vs Instance Bindings

Handle global vs local settings with two bindings per provider:

```yaml
# Global config at :root:orgs:<account>
apiVersion: apis.kcp.io/v1alpha1
kind: APIBinding
metadata:
  name: search-provider-config
spec:
  reference:
    export:
      path: "root:providers:search"
      name: search-config.platform-mesh.io
---
# Instance binding at sub-accounts
apiVersion: apis.kcp.io/v1alpha1
kind: APIBinding
metadata:
  name: search-provider
spec:
  reference:
    export:
      path: "root:providers:search"
      name: search.platform-mesh.io
```

This gives us:
- Singleton config objects at org level (guarded by RBAC)
- Sub-accounts consume the service without duplicating config
- Day-2 enable/disable on existing accounts

### Initializers on Config Objects (not WorkspaceTypes)

Instead of `WorkspaceType` initializers, put initializers on provider config objects:

```yaml
apiVersion: search.platform-mesh.io/v1alpha1
kind: SearchConfig
metadata:
  name: cluster  # Singleton, unique in workspace
  finalizers:
    - search.platform-mesh.io/index-initializer
spec:
  indexingEnabled: true
  retentionDays: 30
status:
  phase: Initializing
  conditions:
    - type: IndexReady
      status: "False"
      reason: "Building"
      message: "Search index is being created"
```

Any object can have an initializer. Make it a singleton and you get the same guarantees as `WorkspaceType` initializers, but:
- Account provisioning isn't blocked
- Can enable/disable day-2
- Clear ownership by the provider

## Alternatives Considered

### Keep Features in Core with WorkspaceType Initializers

Simpler to implement, and blocking guarantees completion. But it couples everything tightly, blocks provisioning when any component fails, and has no day-2 story.

### Embedded Core Features with Provider Interface

Single codebase, easier deployment. But still causes core bloat and doesn't push the provider framework forward.

## Drawbacks

- More repos and deployments to manage—needs good automation
- Two-binding pattern adds complexity
- Existing features need refactoring
- UI needs work for graceful degradation

## References

- [RFC 001: API Provider Metadata and UI Discovery](001_api-providers-and-ui-discovery.md)
- [RFC 002: Terminal Controller Manager](https://github.com/platform-mesh/architecture/pull/8)
- [RFC 003: Search Architecture](https://github.com/platform-mesh/architecture/pull/3)
- [kcp Documentation](https://github.com/kcp-dev/kcp)
