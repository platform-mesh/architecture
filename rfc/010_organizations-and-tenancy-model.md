# RFC 010: Lightweight Tenancy Model — Tenants, Projects and Identity

| Status  | Implemented in part, see [§Status](#status) |
|---------|------------|
| Author  | @mjudeikis |
| Created | 2026-07-27 |
| Updated | 2026-08-05 |

> Drafted with an LLM from a running proof-of-concept, then rewritten against the working code in
> [`platform-mesh/platform-mesh`](https://github.com/platform-mesh/platform-mesh) under
> `operators/tenancy-operator`. Where the code and the design disagree, this document follows the
> code.

## Summary

Tenancy is two kcp workspace tiers and four API objects. Nothing sits in the request path. kcp is
exposed directly and kcp RBAC is the only thing that authorizes.

- **`Tenant`** is the membership and billing boundary. It is a kcp workspace holding Memberships and
  Projects. No tenant work happens in it.
- **`Project`** is where work happens. One kcp workspace per Project, flat under its Tenant.
- **`Membership`** is (user, scope, role). The controller turns it into kcp RBAC.
- **`User`** is the record of an identity that has signed in. Each User has a membership index.

Two components are added to the platform, and nothing else:

| | what it is | what it writes | why |
|---|---|---|---|
| **tenancy VW** | one virtual workspace, platform-wide, at `/services/tenancy/…` | intent: `User`, `Tenant`, `Project`, `Membership` | Optional context based api. Can be replaced by a GraphQL or REST service. |
| **tenancy controller** | five reconcilers reading the fleet through four APIExport VWs | workspaces, RBAC, index | Ensures the desired state of the tenancy model is reflected in kcp. |

Identity federation happens outside the platform, in a broker such as Dex or Keycloak. kcp trusts one
issuer, so no component has to verify more than one token format.

## Motivation

Today tenancy is `Account` and `AccountInfo` from account-operator, with authorization in a separate
stack of Keycloak, OpenFGA and `rebac-authz-webhook`. See
[RFC 007](./007_platform-simplification.md). Three problems:

- **Two authorization systems.** OpenFGA tuples and kcp RBAC both decide access and have to be kept
  in sync. Every new surface has to pick one to trust.
- **No boundary object.** `Account` is both the billing boundary and the place work happens. There is
  no way to give someone one team space without giving them the whole account, except by making a
  second account.
- **Identity coupling.** Terminating several identity providers inside the platform means every
  component has to verify several token formats.

## Terminology

| Term | Meaning here |
|---|---|
| Tenant | Membership and billing boundary. A kcp workspace of type `tenant`. Holds no tenant work. |
| Project | A team space. The tenant-facing handle for a kcp workspace of type `project`. Flat under its Tenant. |
| Folder | Reserved, not built. A grouping tier below Project. See [§Folder](#folder-reserved). |
| Workspace | kcp's own `tenancy.kcp.io/Workspace`. Never exposed. Clients name Projects instead. |
| User | An object whose `metadata.name` is `sha256(issuer + "\n" + sub)`. Carries `rbacIdentity`. |
| Membership | (user, scope, role), where scope is `tenant` or `project` and role is `admin`, `member` or `viewer`. |
| `rbacIdentity` | The username kcp sees after OIDC extraction. It joins this model to kcp RBAC. |
| tenancy VW | The single virtual workspace serving the tenancy APIs. |

Capitalised **Tenant** is the object. A **member** is a person holding a Membership. Lowercase
"tenant" as an adjective (tenant work, tenant tier) means the customer side of the platform.

In examples, `pm:` is the configured OIDC username prefix and `platform-mesh.io` is the API group.
Both are deployment settings. Nothing in the model depends on the strings.

## The tree

```
root                                    ← install root, configurable
├── providers/<provider>/
├── tenants/                            the tenant fleet
│                                        binds tenancy-provisioner
│   └── <tenant-uuid>/                  WorkspaceType: tenant
│       │                                binds tenancy + tenancy-provisioner
│       │                                does NOT bind tenancy-access, so no RBAC is written here
│       │                                holds: Membership, Project
│       ├── <project-uuid>/             WorkspaceType: project
│       │                                binds tenancy-access (claims only, no schemas)
│       │                                holds: provider APIBindings, namespaces, RBAC, all work
│       └── <project-uuid>/             flat siblings
└── system/
    ├── controllers/                    defines all four APIExports and their schemas.
    │                                     Stores nothing.
    ├── providers/                      binds the provider export
    └── directory/                      binds tenancy-platform
                                          holds: User, Tenant, UserMembershipIndex
```

`root:system:controllers` is the only workspace that defines APIs. Every workspace that stores those
objects is an ordinary consumer with an `APIBinding`, including `root:system:directory`.

### Four exports

An APIBinding does two separate things. It imports the export's **schemas**, so the consumer can
store those objects. It also grants the export's **permission claims**, so the provider can reach
into the consumer. Different tiers need different halves. A binding imports all of an export's
resources, and there is no way to import only some. So there are four exports:

| APIExport | schemas | claims | bound in |
|---|---|---|---|
| `tenancy` | `Membership`, `Project` | none | every Tenant workspace |
| `tenancy-platform` | `User`, `Tenant`, `UserMembershipIndex` | none | `system:directory` only |
| `tenancy-provisioner` | none | `tenancy.kcp.io/workspaces` | `root:tenants` and every Tenant |
| `tenancy-access` | none | namespaces, serviceaccounts, clusterroles, clusterrolebindings | every Project, never a Tenant |

The two exports with schemas are split by audience. Binding `tenancy` in a Tenant must not make
`User` servable there. The two without schemas are split by capability: creating a workspace versus
reaching inside one. Splitting them means the two most dangerous powers are named, granted and
revoked separately.

`tenancy-access` is not bound in the Tenant tier. It is the only export that claims
`clusterrolebindings`, so the operator cannot write RBAC into a Tenant workspace even if a future
reconciler asked it to. The guarantee is a missing claim, not code that declines.

`Project` is in the `tenancy` export next to `Membership`, so it lives in the Tenant's own cluster.
Listing a Tenant's projects is one list in one cluster. Deleting a Tenant deletes its Projects,
because they are stored inside it.

### Paths are configuration

| option | default | controls |
|---|---|---|
| install root | `root` | the prefix everything hangs off |
| tenant fleet root | `<root>:tenants` | where Tenants are created |
| directory workspace | `<root>:system:directory` | where User, Tenant and index live |
| exports workspace | `<root>:system:controllers` | where exports are defined, and what bindings reference |
| VW path prefix | `/services/tenancy` | where the singleton is mounted |

The platform should not claim `root:`, because kcp's root may belong to someone else's tree. Two
installs on one kcp need separate subtrees and separate export workspaces, since a binding references
its export by path. The VW mount prefix has to differ too, or both installs answer on
`/services/tenancy`.

The shape is fixed: a Tenant is always a child of the fleet root, and a Project is always a child of
a Tenant.

### WorkspaceTypes

| Type | `extend` | parents | children | default bindings |
|---|---|---|---|---|
| `tenant` | `root:universal` | — | `project` | `tenancy`, `tenancy-provisioner` |
| `project` | none | `tenant` | `none: true` | `tenancy-access` |

Two things are left out on purpose:

- `project` does not extend `root:universal`. Universal would add `tenancy.kcp.io` and
  `topology.kcp.io` to the tenant's API surface, letting a tenant create workspaces and read cluster
  topology. The cost is that kcp no longer creates the `default` namespace, so the controller creates
  it through the `tenancy-access` claim.
- `project` adds no tenant-facing API. `tenancy-access` has no schemas, so `kubectl api-resources` in
  a new Project shows only kcp built-ins. A tenant gets APIs by enabling providers.

`limitAllowedChildren: {none: true}` keeps Projects flat. Nesting them would put a `Workspace` object
inside a tenant's own workspace, which would mean granting the `create workspaces` claim in every
Project.

### Folder (reserved)

When two levels are not enough, a third kind is reserved for future use:

```
Tenant → Project -> Folder → Folder → …
```

Folder nests. Project stays a parent. Nothing implements Folder and no API reserves the name. Two
things have to be true now for it to be added later without a migration, and both are true:
`Project` has to have `tenancy.kcp.io` api (it does not have it now), and `Membership.spec.scope` 
is an enum that can take a new value.

## Objects

All objects are cluster-scoped. `metadata.name` is assigned by the server on Tenant and Project.
`displayName` is metadata and is not unique.

**In `root:system:directory`, through `tenancy-platform`:**

| object | spec | status |
|---|---|---|
| `User` | `email`, `name`, `issuer`, `subject`, `rbacIdentity`, `tenantQuota`, `tenancy{seedTenant, seedProject}` | `active`, `lastLogin`, `defaultTenant`, `defaultProject`, `conditions` |
| `Tenant` | `displayName`, `personal`, `projectCreation`, `providersMetadataCreation`, `projectQuota` | `workspacePath`, `clusterID`, `firstAdmin`, `conditions` |
| `UserMembershipIndex` | `entries[]` | `entryCount`, `observedGeneration` |

**In each Tenant's workspace, through `tenancy`:**

| object | spec | status |
|---|---|---|
| `Membership` | `user`, `scope`, `project`, `role` | `clusterID`, `conditions` |
| `Project` | `displayName` | `clusterID`, `workspacePath`, `conditions` |

Details the field lists do not show:

- **A User's name is the identity.** It is the full 64-character hex of
  `sha256(issuer + "\n" + sub)`, not a generated name with the hash in a label. This makes
  self-provisioning safe to repeat: two concurrent creates for one identity collide on
  `AlreadyExists`. The separator is a newline because both an issuer URL and a `sub` can contain
  slashes.
- **The default on `User.spec.tenancy` is on the object, not on its fields.** Structural defaulting
  only descends into objects that are present. Defaults on `seedTenant` and `seedProject` alone never
  fire when a client omits `tenancy`. The fields arrive as `false` and the User gets nothing.
- **`status.defaultTenant` and `defaultProject` record that seeding happened.** They are set once and
  never cleared or acted on again. If the User deletes that Tenant, it stays deleted. They are not
  access-control inputs and not request-scoping defaults. There is no default workspace in this
  model.
- **Every Membership lives in the Tenant workspace,** whatever its scope. A project-scope Membership
  names its target in `spec.project` but is stored one tier up. The `project` WorkspaceType does not
  bind the tenancy group, so there is nowhere in a Project to put it. The Tenant tier is also the one
  the controller can watch fleet-wide. The cost is that deleting a Project does not remove its
  Memberships, so the Project reconciler deletes them and waits for them to go before the workspace
  is torn down.
- **A Membership's name is derived** from (user, scope, project) as a UUIDv5. Granting the same
  access twice collides instead of writing a second object. Two Memberships for one grant would mean
  two role bindings, and revoking one would leave the other in place.
- **`UserMembershipIndex` is a read model, not a gate.** There is one per User, sharing its name. It
  lets a client list its tenants and projects without querying every workspace. It is not consulted
  when a tenant talks to their own workspace. kcp RBAC decides that. If the two disagree, RBAC is
  right and the index needs rebuilding. Each row carries both cluster IDs, so a client can render a
  switcher and build a kubeconfig from one read. The key is `(tenantUUID, projectUUID)`, and an empty
  `projectUUID` is a key, not a wildcard.

### Naming

A client can never supply the name. `metadata.name` on a Tenant or Project is also the kcp workspace
name, so picking it means picking a path. Paths are where collisions, renames and leaked tenant
strings cause trouble.

Which server-assigned name to use is a setting, `--tenancy-naming-strategy`:

| strategy | example | unique by |
|---|---|---|
| `uuid` (default) | `7f3a91d2-…` | 122 bits |
| `base36` | `k3f9q2m1x7t0b` | 64 bits, kcp's own shape |
| `words` | `ruby-lunar-plateau` | about 258k triples, then a suffix |
| `displayname` | `acme-co` | the slug, then a suffix |

The creator handles collisions, not the strategy: create, retry on `AlreadyExists`, stop when the
strategy says it is out of options. A strategy is a plain function that never calls an API server.

Two things to know before changing away from `uuid`. It does not apply to existing objects, because
renaming would move a workspace. And `displayname` puts tenant input into workspace paths, kubeconfig
URLs and logs. That is reasonable for Projects. It is worse for Tenants, where the first tenant to
create `platform` holds that name for the whole platform.

It's recommended to use `uuid`. But one can choose a different one for readability. It's recommended to
set strategy at install time and never change it, because a new strategy does not rename existing
objects. Migration is possible, but it is a separate problem from this RFC.

## Access

### Two planes, both enforced by kcp

**No member can reach the Tenant tier, because no member has RBAC there.** A member can address
`root:tenants:<uuid>` and kcp will refuse, because nothing ever created a role binding for their
identity in that cluster. The only identity with access is the tenancy VW's. This holds for a new
client, a forgotten code path or a misconfigured route, because there is no check to miss.

**The Project tier is ordinary kcp RBAC.** A Membership makes the controller write a role binding for
the user's `rbacIdentity`. Removing the Membership removes the binding. The workspaces a user can
reach are exactly the workspaces they have bindings in. There is no allow-list, no index on the
request path, and no cache that can go stale after a revocation.

### Roles

| Scope and role | Through the tenancy VW | Direct kcp (Project) |
|---|---|---|
| Tenant admin | create Projects, manage all Memberships, edit `Tenant.spec` | admin in every Project |
| Tenant member | list Projects, create them if `projectCreation=members` | member in every Project |
| Tenant viewer | list Projects | viewer in every Project |
| Project admin | — | full access plus `escalate` and `bind` in that Project |
| Project member | — | create, edit and delete objects, no RBAC escalation |
| Project viewer | — | read-only |

The three tiers differ in their verb sets, mainly in the two verbs that control escalation.
`escalate` lets a caller write a Role granting more than they hold. `bind` lets a caller reference a
role they do not hold. `admin` has both through `*`. `member` has a listed set that leaves out both,
and also leaves out `impersonate`. `viewer` has `get`, `list` and `watch`, and not `create`, because
`create` on `*/*` covers `TokenRequest` and `SelfSubjectAccessReview`.

A member can still write ordinary RBAC objects inside their own Project. The Membership reconciler
owns the bindings it wrote and rewrites them, so a member who deletes a colleague's access only
breaks it until the next reconcile.

**Roles are ranked and the strongest wins.** A user can hold both a tenant-scope and a project-scope
Membership in the same Tenant. An unknown role ranks below viewer, so an old binary that meets a role
added by a newer one gives too little access rather than too much.

**A Tenant admin is admin in every Project in that Tenant.** No Project is private from its Tenant
admin. The privacy boundary is the Tenant. A team that needs isolation uses a separate Tenant. This
belongs in the onboarding docs.

The implicit access is written out, not inferred. kcp only asks whether a binding exists in the
cluster being addressed, so a tenant-scope Membership becomes one ClusterRoleBinding per Project
cluster. The Membership controller watches `Project` for this reason: a set computed once would miss
Projects created later. A tenant-scope Membership in a Tenant with no Projects writes no bindings.
That is the normal state at bootstrap, and it is `Ready`, not pending.

### Context is in the path

There is no server-side "active tenant". Scope is the cluster segment of the URL:
`/services/tenancy/clusters/{tenant}/…` for tenancy, `/clusters/{id}/…` for a workspace. This is how
every other kcp surface expresses scope. Display names never travel on the wire.

## Bootstrap

Creating a `User` starts a chain that is safe to repeat. It is not one state machine. It is five
controllers, each owning one object, each handing off by creating the next object rather than by
calling the next step.

```
User appears                                                  [UserReconciler]
 ├─ RBACIdentityCurrent    recompute spec.rbacIdentity from live OIDC config
 ├─ SeedTenantReady        create the personal Tenant          (once)
 └─ SeedProjectReady       create the first Project in it      (once)

Tenant appears                                              [TenantReconciler]
 ├─ WorkspaceReady         create <fleet-root>/<tenant>, resolve its cluster ID
 ├─ OwnerMembershipReady   Membership{user, scope=tenant, role=admin}
 └─ IndexSynced            holds the finalizer that prunes this tenant's rows

Membership appears                                      [MembershipReconciler]
 ├─ RBACApplied            ClusterRole + ClusterRoleBinding in every target
 └─ IndexSynced            the subject's row, never before the binding exists

Project appears                                            [ProjectReconciler]
 ├─ WorkspaceReady         create <tenant>/<project>, resolve its cluster ID
 └─ MembershipsPruned      on delete only: revoke grants, then release the workspace

Workspace becomes Ready                                  [WorkspaceReconciler]
 └─ create `default`       kcp will not, since `project` omits universal

ClusterRoleBinding changes                                 [BindingReconciler]
 └─ nudge its Membership   repair deleted or edited platform RBAC
```

**Each step hands off by creating an object, not by calling the next step.** `SeedTenantReady`
creates a Tenant and stops. It does not wait for the workspace and never learns whether it worked.
Each controller's `Ready` covers only its own work, so a stuck Project does not hold up its Tenant,
and a restarted operator picks up from whatever objects exist.

**Seeding runs once.** It happens while `status.defaultTenant` is empty and never again. Turning
`spec.tenancy.seedTenant` off and on does not re-run it. Otherwise the platform would recreate
objects a tenant chose to delete.

Steps can be enabled and disabled individually with `--reconcilers-*-enabled` and
`--controllers-*-enabled`. One rule: a chain with no enabled steps must not report `Ready=True`. An
empty chain that succeeds looks exactly like one that did the work.

Cold start takes 10 to 25 seconds, so a new user is authenticated before they have any membership.
Nothing blocks on it. The VW answers with what exists and the client polls. A watch is the intended
shape and is not built.

Every user gets a personal Tenant with `spec.personal: true`, which does not count against the quota,
plus one Project. A platform admin can turn this off fleet-wide with
`--tenancy-personal-tenants-enabled=false`, and a single User can opt out through `spec.tenancy`.

## The tenancy virtual workspace

One instance for the whole platform, built on
[`virtual-workspace-framework`](https://github.com/kcp-dev/kcp/tree/main/staging/src/github.com/kcp-dev/virtual-workspace-framework).
It replaces the JSON REST service in the reference implementation with a Kubernetes API. Clients use
client-go instead of a hand-written SDK, `watch` instead of polling, an authorizer and admission
plugins instead of checks in HTTP handlers, and GraphQL generated from discovery instead of
hand-written resolvers.

It is a singleton because the tenant tree crosses workspaces. No single workspace can answer "which
tenants am I in", so the per-workspace and per-export VW patterns do not fit. One instance also means
one endpoint to publish and one component holding the privileged identity for the tenant tier. It is
stateless and scales horizontally.

### URL and resources

```
/services/tenancy/clusters/{cluster}/apis/tenancy.platform-mesh.io/v1alpha1/{resource}[/{name}]
```

`{cluster}` is either `*`, meaning across the whole fleet and filtered to the caller's memberships,
or one Tenant's cluster ID. The wildcard covers the case where no tenant has been selected yet.

| resource | scope | verbs | notes |
|---|---|---|---|
| `users` | `*` | get, create | self only, `~` means the caller. `create` with an empty spec provisions the caller. `get` returns 404 until then |
| `tenants` | `*` | get, list, create | filtered to the caller's memberships |
| `projects` | `*` or a tenant | get, list, create | `create` refused for a viewer, and for a member when `projectCreation=admin` |
| `memberships` | `*` or a tenant | get, list, create, update, delete | every role can read. Writes are admin-only, except leaving on your own row |

The VW is not per-export. `tenants` and `users` come from `tenancy-platform`, `projects` and
`memberships` from `tenancy`. The export split is a storage boundary and clients do not see it.

`memberships` is the only way a tenant manages access. No tenant identity has a binding in the Tenant
workspace, so there is no `kubectl` route around this storage, and its rules hold:

| rule | reason |
|---|---|
| only an admin may `create` or `update` | a member who could add members could promote themselves through a second account |
| `update` has no self-service form, unlike `delete` | leaving is your decision. Editing your own grant is self-promotion |
| `update` accepts only `spec.role` | the name comes from (user, scope, project), so changing one of those describes a different grant stored under this one's name |
| demoting the last admin is refused, like deleting them | the risk is the grant disappearing, not which verb removed it |
| the subject must already have a `User` | otherwise the Membership names nobody. Invitations are a separate object |
| the last tenant admin cannot be removed | a Tenant with no admin cannot be administered again |

The last-admin check reads and then writes, so it can race. Two admins leaving at the same time can
both succeed. This is accepted: the check makes an ownerless Tenant rare rather than impossible. A
reconciler that reports ownerless Tenants is the longer-term fix.

### Every response is scoped to the caller

The same URL returns a different body to different callers. This is what makes one shared instance
safe.

| operation | behaviour |
|---|---|
| `list` over `*` | returns what the caller's memberships cover. Always filtered, never "403 unless you can see everything" |
| `get` outside that set | 404, not 403. A 403 confirms the object exists, and the API must not tell callers which tenant UUIDs are real |
| `create` in `/clusters/{tenant}` | only where the caller is a member. The wildcard scope accepts one create, a new Tenant |
| `users` | self only. There is no call that lists the platform's people |
| `memberships` in a Tenant | lists co-members, viewers included. A member who cannot read the roster cannot tell whether an admin still exists |

Two consequences:

- **Visibility has to change during a watch, not just when it starts.** When a Membership is added or
  removed, an open wildcard watch has to send `ADDED` or `DELETED` for objects entering and leaving
  the caller's view. Normal kube watch semantics assume the authorization decision holds for the life
  of the stream, and here it does not. This is not built, and it is better left unbuilt than
  half-built: a watch that filtered only at the start would look like it worked while streaming a
  Tenant to someone removed from it minutes ago.
- **Nothing may be cached across callers.** Any cache is keyed by caller. Shared informers sit behind
  the filter, never in front of it. Today there is no cache at all and every request re-reads the
  caller's index, which is a correct start and a poor steady state.

Failures close: an identity that cannot be resolved or an index that cannot be read produces an error
or an empty result, never an unfiltered one.

### Authorization lives in the storages

`Authorize` admits any `system:authenticated` caller and does nothing else. All narrowing happens in
the storage. An `authorizer.Authorizer` receives attributes and returns a boolean, which does not fit
here. The question is not whether the caller may list tenants, which is always yes, but which tenants
to return, and the answer is a filter over rows. So the authorizer establishes that there is a
caller, and the storage decides what that caller sees.

The cost is that there is no single place to audit. There are four storages, and a fifth resource
would mean remembering to filter it. One shared helper, `resolveAccess`, keeps them consistent.

`Mutate` is empty and stays empty. Each storage's `Create` fills in server-owned fields, because
those come from the caller's credential, which admission cannot see, and because the naming strategy
retries against the API server on collision, which admission cannot do.

`Validate` is empty, and that is a gap. Quota and the last-admin check belong there, so that a
`kubectl delete` cannot get past a check written into a storage method.

The storage is a projection, not a database. The VW writes intent and returns. The controllers do all
the reconciling. `create tenants` writes a `Tenant` and returns it. It does not wait for a workspace,
does not poll for `Ready`, and does not fail if the controller is down.

The VW is its own command and its own Deployment, not a goroutine inside the operator. It is
stateless and scales horizontally, while the controller is a leader-elected singleton. They share a
binary so they cannot disagree about the workspace layout, the `rbacIdentity` convention or the API
types.

## The controller plane

The singleton VW is where intent is written. Acting on that intent across the fleet is a different
virtual workspace: the standard kcp APIExport VW, one per export, so four of them.

```
 clients (kubectl / CLI / UI)                    tenancy controller
         │ write intent                                  │ watch fleet-wide, reconcile
         ▼                                               ▼
┌──────────────────────────┐   ┌─────────────────────────────────────────────────┐
│  tenancy VW (singleton)  │   │  APIExport VWs (standard kcp, one per export)   │
│  /services/tenancy/…     │   │   · tenancy-platform     → the directory        │
│  fixed GVs, per-caller   │   │   · tenancy              → every Tenant          │
│  storage-side filtering  │   │   · tenancy-provisioner  → fleet root + Tenants  │
└───────────┬──────────────┘   │   · tenancy-access       → Projects only         │
            │                  └──────────────────┬──────────────────────────────┘
            ▼                                     ▼
   <root>:system:controllers  ── defines all four exports, stores nothing
   <root>:system:directory    ◄─ tenancy-platform ──►  Users, Tenants, index
   <root>:tenants             ◄─ tenancy-provisioner►  the Tenant Workspace objects
   <root>:tenants:<tenant>    ◄─ tenancy ───────────►  Memberships, Projects
                              ◄─ tenancy-provisioner►  the Project Workspace objects
                              ── no tenancy-access: no RBAC is written here ──
     └── <project>            ◄─ tenancy-access ────►  namespaces, all tenant RBAC
```

A controller cannot watch across logical clusters on its own. kcp gives it one wildcard endpoint, the
APIExport virtual workspace, which spans every workspace bound to that export. Binding is therefore
how a workspace becomes visible to the control plane. This is why Memberships live in the Tenant
tier: it is the tier the controller can watch fleet-wide.

Four managers in one process is the cost of splitting the exports. It is paid in informers, not in
credentials. Each manager has exactly the reach its claims declare, and a bug in one cannot write
through another. `kubectl get apibindings` in a Project shows what the platform can touch there.

### No component holds kcp admin at runtime

| operation | goes through |
|---|---|
| read and write `User`, `Tenant`, index | `tenancy-platform` export VW |
| read and write `Membership`, `Project` | `tenancy` export VW |
| create Tenant and Project workspaces | `tenancy-provisioner` claim on `tenancy.kcp.io/workspaces` |
| create `default`, write RBAC in a Project | `tenancy-access` claims |
| mint a ServiceAccount token | unverified, see [§Open questions](#open-questions) |

The operator's runtime kubeconfig reaches the fleet only through those four VWs. The one credential
that needs kcp admin belongs to `tenancy-operator init`, which runs as an init container and exits,
so the credential goes with it. Installing the platform, which means creating the workspaces,
WorkspaceTypes, exports, schemas and the two hand-written bindings, is an admin action done once. If
the controller's kubeconfig grants anything beyond its export VWs, something has regressed.

## kcp traps

Each of these cost real debugging time. They are kcp behaviours, not decisions in this design.

**Getting into a workspace needs a non-resource grant.** kcp's workspace content authorizer runs
before resource RBAC and requires the verb `access` on the non-resource path `/`. A ClusterRole
granting `*` on `*/*` does not cover it, so every request returns 403 with
`no verb=access permission on /`, which reads like a broken grant rather than a missing one. Two
rules go in front of every role this platform writes, the same for admin, member and viewer:

```yaml
- nonResourceURLs: ["/"]
  verbs: ["access"]
- nonResourceURLs: ["/api", "/api/*", "/apis", "/apis/*", "/openapi", "/openapi/*", "/version"]
  verbs: ["get"]                # discovery: kubectl asks for this before anything else
```

They are the same in all three roles because entering a workspace is not a privilege. It is what has
to happen before any privilege applies.

**A permission claim is matched by (group, resource, identityHash).** An accepted claim without the
hash is a different claim and matches nothing. The binding still reports `Bound` and
`kubectl get apibindings` looks fine. The first symptom is `access denied` on the first write. Every
claim on a `tenancy.kcp.io` resource has to carry that export's identity hash, resolved at install
time and substituted into the manifest.

**`defaultAPIBindings` does carry accepted claims.** kcp's default-apibinding controller copies every
claim from the referenced export into the binding, with `state: Accepted`, the `identityHash`, and
`selector: matchAll`. An earlier draft assumed the opposite and built two WorkspaceType initializers
on top of it. Both are gone and no initializer exists in this design. That is better in operation: an
initializer only this operator can clear means every new workspace gets stuck in `Initializing`
whenever the operator is missing. The cost is a short window where a Project workspace is `Ready`
before `default` exists. A missing operator should delay provisioning, not block it.

**`--oidc-username-prefix` defaults to nothing when the claim is `email`.** For every other claim
kube defaults it to `<issuer>#`. So `email`, the most common setting, is the one where forgetting the
prefix quietly produces a bare username, and two identity sources asserting the same address collide.
Set a prefix explicitly, including for `email`. This bit us from both sides: the operator had `pm:`
and kcp had none, so every ClusterRoleBinding named `pm:dex@pm.localhost` while kcp authenticated
`dex@pm.localhost`. Both objects looked right and the user got 403 in a workspace they owned.

**`APIResourceSchema` names have to be content-hashed.** Schemas are immutable. A name that does not
change with the content means a regenerated schema keeps serving the old one, which shows up as a
field the CRD clearly declares being rejected as unknown.

**Every type the VW serves needs a generated OpenAPI model.** The delegated API server resolves one
per type at startup and exits with `cannot find model definition for …` without it. Clients see the
connection close mid-request, because there is no server left to write an API error. Nothing in
`go build` or the unit tests catches this.

**`limitAllowedParents` matches a canonical path, not a cluster ID.** An export reference accepts
either. This does not. A cluster ID is accepted at install time and fails when the first child
workspace is created.

**`defaultAPIBindings` apply only when a workspace is created.** Changing them does not re-bind
existing workspaces. Rolling out a change is a migration.

**`RunFinalize` walks a reconciler chain backwards.** A step that has to clean up before another one
must be registered after it. This is how deleting a Project revokes its grants before its workspace
is destroyed: the prune step is last in the chain, so it finalizes first.

**A finalizer waiting on a cluster that is being deleted will deadlock.** A Membership holds a
finalizer until it has removed its ClusterRoleBindings. If the workspace holding those bindings is
being deleted too, neither side finishes. The fix is to release the finalizer without acting when no
live Project still claims the target cluster. An error while checking does not count as gone: if the
list fails, keep holding the finalizer.

**Repair has to release the finalizer before nudging.** The binding-repair controller nudges a
Membership when its ClusterRoleBinding is deleted. If it nudges while the finalizer still holds the
object, the rebuild finds the binding present, changes nothing and reports success. The finalizer is
then released and the binding disappears. The first version did this, and bindings stayed deleted for
a full minute while every reconcile reported success. Swapping the order took repair from never to
under a second.

## RBAC self-repair

Granting and revoking are the same reconciler over `Membership`, so they cannot drift apart. Adding a
Membership writes the ClusterRole and ClusterRoleBinding. Deleting it removes them. Patching
`spec.role` rewrites them.

Drift is repaired by a watch, with a resync as backup. A second controller watches
`ClusterRoleBinding` through `tenancy-access`, filtered by the `platform:membership:` name prefix.
That prefix is why platform-owned bindings are named rather than opaque. Every binding carries labels
naming its Membership and Tenant, and a finalizer. The finalizer does not block deletion. It makes
the deletion observable with the labels still attached, because a tombstone with no labels does not
say what to rebuild. The Membership controller also resyncs every 5 minutes, for events missed while
the operator was down.

The Membership reconciler computes the RBAC subject from the live username convention instead of
reading `User.spec.rbacIdentity`. A fleet-wide prefix change then repairs itself on the next resync
instead of needing a migration job.

## Identity

kcp is configured with one issuer:

```
--idp-issuer-url=https://auth.example.com
--idp-client-id=platform
--idp-ca-file=/idp-ca/ca.crt     # needed for private-CA issuers
```

Those values go into kcp's own OIDC authenticator, with `UsernameClaim=email`, `UsernamePrefix=pm:`,
`GroupsClaim=groups` and `GroupsPrefix=pm:`, and into the tenancy VW's verifier. kcp authenticates
the end user's token itself, so RBAC inside a workspace is evaluated against the real user, never
against an impersonating service account. Several upstream identity providers are the broker's
problem, not the platform's.

**Login involves no platform component.** A client is a public OAuth2 client and runs PKCE directly
against the identity provider. There is no client secret anywhere in the platform and no component
holds session state. Login produces credentials, not a workspace. Nothing in the login path knows the
tenant tree, so nothing can hand out a default cluster.

**Provisioning is an explicit call, not a side effect of a read.** A new identity creates its own
`User` with an empty spec, and the server fills every field from the verified token. Doing it on the
first authenticated request would turn `GET` into a write, so a monitoring probe listing tenants
would create projects. It would also hide failures inside unrelated calls and leave nothing to
rate-limit. Doing it in an OIDC callback would only work for interactive browser logins, not for a
CLI with a cached refresh token or a CI job, so provisioning would depend on how someone
authenticated.

**`rbacIdentity` mirrors kcp's configuration, and mirrors drift.** It has to be computed from the
config kcp is running, never hardcoded. If the two differ, every ClusterRoleBinding names a subject
that never authenticates, and the user is denied with a Membership and a binding that both look
correct. Changing the claim convention is an identity migration, not a config change. A
`RBACIdentityCurrent` condition and a reconciler recompute it on every pass, because the condition
can go False without anything about the User changing.

**Platform admin is not a Membership.** It is an operator-level capability, such as registering a
provider or listing every user, and it must never appear in the membership index. Only the way a
caller is recognised is pluggable, behind an `AdminChecker` seam: a static `--admin-users` allowlist
today, `--admin-groups` matched against the OIDC groups kcp already extracts next, and kcp RBAC in a
system workspace as the end state.

## Kubeconfigs

All of them point at the front-proxy, never at a shard, and address a workspace by cluster ID.

**Built by the client after login.** This is built, as `tenancyctl kubeconfig`. It uses an
exec-credential plugin, so tokens refresh without a browser. The cluster is whichever workspace the
client picked from its memberships. Nothing on the server side picks one.

```yaml
clusters:
- name: platform
  cluster: { server: https://front-proxy.example.com/clusters/<cluster-id> }
users:
- name: user-xxxx
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: <credential-plugin>
      args: [ get-token, --oidc-issuer-url=…, --oidc-client-id=… ]
```

There is no client secret, because PKCE refresh needs only the issuer and client ID. Two details the
example does not show. The exec plugin has to be a stable path: a `go run` binary lives in a temp
directory that is deleted on exit, so the kubeconfig works once. And the issuer URL, client ID and CA
have to be written into the file. An empty `--oidc-issuer-url=` produces a file that passes
`kubectl config view` and fails on first use.

Per-workspace kubeconfigs and ServiceAccounts for bots are designed and not built. See
[§Status](#status).

Switching context without logging in again is a client-side operation: resolve the target cluster ID
through the VW and point the server URL at it. A client that accepts display names has to treat
ambiguity as an error, because display names are not unique.

## Status

The whole chain works end to end. An OIDC login through dex produces a `User`, which seeds a
`Tenant`, its workspace, an owner `Membership`, the RBAC that makes it reachable, a `Project`, that
Project's workspace and the membership index. All of it is watched, self-healing and reachable from a
CLI.

| area | state |
|---|---|
| self-provisioning through `create users` | built |
| one issuer, `rbacIdentity` mirror and repair | built |
| tenant tree, bootstrap chain, two access planes | built |
| tenancy VW | built: four resources, verbs as listed above |
| Membership as a CR for both scopes, revocation, three real roles | built |
| binding-repair watch | built |
| kubeconfigs | client-side only (`tenancyctl`). No subresource, no ServiceAccounts |
| quotas | not built |
| last-admin check | built |
| `watch` on the VW | not built |
| `delete` and `update` on `tenants` and `projects` | not built |
| soft delete | not built, and no fields remain on the API |
| per-tenant identity providers, group-to-Membership sync, group-based platform admin | not built |

The gaps that matter:

- **No `watch`.** Every storage implements `get`, `list` and `create` and nothing else. Visibility
  cannot change mid-stream, and first run is a poll.
- **No `delete` except on `memberships`.** Removing a Tenant or Project needs a kcp credential a
  tenant does not have. This is the biggest gap, and it is one-sided: access can be revoked through
  this API, but the things access points at cannot be removed through it.
- **No `update` on `tenants` or `projects`.** Renaming is impossible through this API, even though
  `displayName` was designed as the field you patch.
- **Quotas do nothing.** `--tenancy-tenant-quota`, `--tenancy-project-quota`,
  `User.spec.tenantQuota` and `Tenant.spec.projectQuota` all exist and nothing reads them.
- **`Tenant.spec.providersMetadataCreation` is stored and never checked.** Its sibling
  `projectCreation` is checked in the `projects` storage.
- **Dead condition constants.** `apis/tenancy/v1alpha1/types_tenant.go` declares
  `TenantConditionMembershipReady`, `WorkspaceCreated`, `NamespaceReady`, `WorkspaceAdminReady` and
  `ProfileReady`. Nothing sets or reads them, and the Tenant chain has three steps. They should be
  deleted.

### What building it changed

| the draft said | what building it showed |
|---|---|
| the tiers are `Organization` and `Account` | renamed to `Tenant` and `Project`. `Account` collided with account-operator's own `Account`, and `Organization` implied a legal entity |
| the Tenant tier binds `tenancy-access` | reverted. That would give a tenant RBAC in the tier that decides access. Tenant-scope Memberships bind only in Project workspaces |
| `defaultAPIBindings` cannot carry accepted claims, so initializers are needed | wrong. Both WorkspaceTypes declare their bindings and no initializer exists |
| `root:tenants` binds `tenancy-access` | it binds `tenancy-provisioner`, the export that exists for that claim |
| a permission claim is matched by (group, resource) | it is (group, resource, identityHash) |
| granting `*/*/*` in a workspace grants access to it | it does not. See [§kcp traps](#kcp-traps) |
| names are UUIDs | a client can never supply the name, but which server-assigned name to use is a setting |
| soft delete lives on `User` fields | the fields were removed. Soft delete is deferred with nothing left behind |

## Alternatives considered

**External ReBAC (OpenFGA).** Rejected for this model. Not because ReBAC is wrong, but because two
authorization systems over the same objects have to be kept in sync, and kcp already enforces RBAC on
the request path with the real user identity. If fine-grained cross-object relations become a
requirement, the place to add them is a kcp authorization webhook, not a second decision plane.

**One flat `Account` covering both tiers.** Rejected. It makes "a second account" the only answer to
"a team that should not see what the other team is running", which duplicates billing, membership and
provider enablement.

**Human-readable workspace paths.** Rejected. Display-name collisions become path collisions,
renames become moves, and tenant names end up in URLs and logs.

**Direct tenant access to Tenant workspaces, restricted by admission.** Rejected. It needs an
admission webhook or MaximalPermissionPolicy scoping to enforce "no tenant work here", and not
writing a role binding gets the same result for free.

**Keep the bespoke REST service.** It works and is less machinery. But it is a second protocol with a
second auth path, a second error convention, hand-written GraphQL resolvers and no watch. Checks that
should be admission end up as handler code that is easy to forget on a new endpoint.

**An APIExport bound into every Tenant instead of a VW.** It would put the tenancy CRDs into the
Tenant's own API surface, need an APIBinding per tenant, and could not answer the cross-tenant
question at all.

**A VW per Tenant.** Same cross-tenant problem, plus one endpoint and one cache per tenant for a view
whose value is being global.

**Nested workspaces in v1.** Deferred. See [§Folder](#folder-reserved).

**Soft delete with an undo window.** Deferred. It cannot use the normal delete path, because
`metadata.deletionTimestamp` is one-way: an object on that path can be delayed by a finalizer but
never recovered. A recoverable window has to be its own state, with `DELETE` meaning "mark". Adding
it later is additive, which is why leaving it out now is safe.

**Invitations.** Adding a member resolves against existing `User` objects, and a `User` exists only
after its owner signs in once, so a teammate has to authenticate before being granted anything. An
`Invitation` object reconciled against `User`s would close that gap.

## Open questions

1. **Cross-shard singleton.** Can one VW serve a `*`-scoped, membership-filtered list and watch over
   a fleet spread across shards, backed by the cache server? This was built against a single shard,
   where the question does not come up. Nothing assumes one shard, and nothing proves it works with
   several. If the answer is no, the singleton serves only the caller-scoped index and proxies
   tenant-scoped reads to the owning shard.
2. **Per-caller watch cost.** One wildcard watch per active client, re-filtered when memberships
   change. This cannot be answered until `watch` exists. Today's cost is one index read per request,
   which is worse in a different way.
3. **Do permission claims cover subresources?** Specifically `serviceaccounts/token`. If not, minting
   bot tokens is the one operation that cannot go through an export, and either the controller needs
   a narrow credential for that call or platform-issued SA tokens have to go.
4. **Should SA tokens reach the tenancy VW?** Default answer is no, since bots act inside a
   workspace rather than on the tree. CI that provisions workspaces is a real use case though.
5. **kcp multi-issuer support.** Does `AuthenticationConfiguration` with several JWT authenticators
   work in the embedded kcp, and can `usernamePrefix` be set per issuer? This blocks per-tenant
   identity providers.
6. **Group sync semantics.** When a group is removed in the identity provider, revoke immediately or
   keep the grant for a grace period?
7. **Cross-tenant read-only access.** `viewer` is a real role with strictly less access, but it does
   not span Tenants. A Membership is scoped to one Tenant or one Project, so read-only access to
   another team's Project still means a Membership in it.
8. **Membership index scale.** One object per user, read on every request, with no informer and no
   cache. Is an informer-backed lister enough, or does the index need sharding?
9. **Relationship to account-operator.** Is this a replacement, a layer above it, or a parallel model
   for a different deployment? The question is sharper now that this RFC's child tier is also called
   `Project`: `tenancy.platform-mesh.io/Project` versus `core.platform-mesh.io/Account`. Two kinds
   with the same name is tolerable while one is a proposal and a problem once both ship.

## Appendix A — objects

Shortened. Read the Go types in `apis/tenancy/v1alpha1` for the full field set and validation.

### A.1 `root:system:directory`, through `tenancy-platform`

```yaml
apiVersion: tenancy.platform-mesh.io/v1alpha1
kind: User
metadata:
  # the name is the identity: sha256(issuer + "\n" + sub)
  name: 9f2c4a7e1b8d3f05c6a9e2b7d4f18c30a5e7b91d6c2f4a08e3b5d7c9f1a2e4b6
spec:
  email: alice@acme.example
  name: Alice Doe
  issuer: https://auth.example.com
  subject: 9c4b8e1f-…
  rbacIdentity: "pm:alice@acme.example"   # prefix + username claim, as configured on kcp
  tenantQuota: 0                          # 0 means platform default. Not enforced
  tenancy:                                # the default is on this object, not on the fields
    seedTenant: true
    seedProject: true
status:
  active: true
  lastLogin: 2026-07-27T09:12:04Z
  defaultTenant: 7f3a91d2-…               # a record that seeding happened, never acted on again
  defaultProject: 9c4b8e1f-…
  conditions:
    - {type: RBACIdentityCurrent, status: "True"}
    - {type: SeedTenantReady,     status: "True"}
    - {type: SeedProjectReady,    status: "True"}
    - {type: Ready,               status: "True"}
```

```yaml
apiVersion: tenancy.platform-mesh.io/v1alpha1
kind: Tenant
metadata:
  name: 7f3a91d2-…                        # server-assigned, and also the workspace name
spec:
  displayName: ACME Corp                  # metadata only, not unique
  personal: false                         # immutable. When true, excluded from the quota
  projectCreation: members                # members | admin
  providersMetadataCreation: members      # members | admin. Not enforced
  projectQuota: 0                         # 0 means platform default. Not enforced
status:
  workspacePath: root:tenants:7f3a91d2-…  # the string kcp's logs and errors use
  clusterID: 2cs1kksarttqbr95             # what clients address
  firstAdmin: 9f2c4a7e…
  conditions:
    - {type: WorkspaceReady,       status: "True"}
    - {type: OwnerMembershipReady, status: "True"}
    - {type: IndexSynced,          status: "True"}
    - {type: Ready,                status: "True"}
```

```yaml
apiVersion: tenancy.platform-mesh.io/v1alpha1
kind: UserMembershipIndex
metadata:
  name: 9f2c4a7e…                         # one per User, with the same name as that User
spec:
  entries:
    - tenantUUID: 7f3a91d2-…              # tenant-scope row, so no projectUUID
      tenantDisplayName: ACME Corp
      tenantClusterID: 2cs1kksarttqbr95
      tenantFirstAdmin: 9f2c4a7e…
      role: admin
      personal: true
    - tenantUUID: 7f3a91d2-…              # project-scope row
      tenantClusterID: 2cs1kksarttqbr95
      projectUUID: 9c4b8e1f-…
      projectDisplayName: platform
      projectClusterID: 1xk9wq3ftz0m4a7c  # what a kubeconfig points at
      role: member
```

### A.2 `root:tenants:<tenant>`, through `tenancy`

```yaml
apiVersion: tenancy.platform-mesh.io/v1alpha1
kind: Membership
metadata:
  name: <uuid>                            # UUIDv5 over (user, scope, project)
spec:
  user: 9f2c4a7e…                         # the User's name
  scope: project                          # tenant | project
  project: 9c4b8e1f-…                     # required when scope is project
  role: member                            # admin | member | viewer
status:
  clusterID: 1xk9wq3ftz0m4a7c             # set only when it names one place, so scope=project.
                                          #   empty for scope=tenant, which binds in every Project
  conditions:
    - {type: RBACApplied,  status: "True"}
    - {type: IndexSynced,  status: "True"}
    - {type: Ready,        status: "True"}
```

```yaml
apiVersion: tenancy.platform-mesh.io/v1alpha1
kind: Project
metadata:
  name: 9c4b8e1f-…
spec:
  displayName: platform                   # the only field a client sets
status:
  clusterID: 1xk9wq3ftz0m4a7c
  workspacePath: root:tenants:7f3a91d2-…:9c4b8e1f-…
  conditions:
    - {type: WorkspaceReady, status: "True"}
    - {type: Ready,          status: "True"}
```

### A.3 What a Membership becomes in the target workspace

The controller writes this through the `tenancy-access` claims. A tenant never writes it. There is
one per target cluster: for `scope: tenant` that is every Project in the Tenant, and for
`scope: project` the one it names. Never the Tenant workspace itself.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  # named after the Membership, not the user, so it can be found again on update and delete.
  # the prefix marks it as platform-owned and is what the repair watch filters on.
  name: platform:membership:395529d0-…
  labels:
    # a back-reference, so a delete event says what to rebuild
    tenancy.platform-mesh.io/membership: 395529d0-…
    tenancy.platform-mesh.io/tenant: 2cs1kksarttqbr95
  finalizers:
    # not to block deletion. It keeps the labels attached long enough for the delete to be seen
    - membership.tenancy.platform-mesh.io/binding
subjects:
  # computed from the live username convention, never read back from User.spec.rbacIdentity
  - {kind: User, name: "pm:alice@acme.example", apiGroup: rbac.authorization.k8s.io}
roleRef:
  kind: ClusterRole
  name: platform:project:member           # or :admin / :viewer
  apiGroup: rbac.authorization.k8s.io
```

The same reconciler writes the `ClusterRole` into the same cluster. It always starts with the two
non-resource rules from [§kcp traps](#kcp-traps).

### A.4 The exports and the two hand-written bindings

```yaml
# root:system:controllers defines all four exports and stores nothing
apiVersion: apis.kcp.io/v1alpha2
kind: APIExport
metadata: {name: tenancy}                 # tenant-facing, bound in every Tenant
spec:
  resources:
    - {name: memberships, group: tenancy.platform-mesh.io, schema: v27d7f33661b6.memberships.…}
    - {name: projects,    group: tenancy.platform-mesh.io, schema: vf2b3ab05d930.projects.…}
---
apiVersion: apis.kcp.io/v1alpha2
kind: APIExport
metadata: {name: tenancy-platform}        # platform-global, never bound in a tenant tier
spec:
  resources:
    - {name: users,                 group: tenancy.platform-mesh.io, schema: vfcbdb9c95a67.users.…}
    - {name: tenants,               group: tenancy.platform-mesh.io, schema: va9d11835eb94.tenants.…}
    - {name: usermembershipindices, group: tenancy.platform-mesh.io, schema: vad3a0315d204.usermembershipindices.…}
---
apiVersion: apis.kcp.io/v1alpha2
kind: APIExport
metadata: {name: tenancy-provisioner}     # capability: create workspaces. No resources.
spec:
  permissionClaims:
    - {group: tenancy.kcp.io, resource: workspaces, identityHash: <tenancy.kcp.io identity>,
       verbs: [get, list, watch, create, delete]}
---
apiVersion: apis.kcp.io/v1alpha2
kind: APIExport
metadata: {name: tenancy-access}          # capability: write inside a workspace. No resources.
spec:
  permissionClaims:
    - {group: "",                        resource: namespaces,          verbs: [get, list, watch, create]}
    - {group: "",                        resource: serviceaccounts,     verbs: [get, list, watch, create, update, delete]}
    - {group: rbac.authorization.k8s.io, resource: clusterroles,        verbs: [get, list, watch, create, update, delete]}
    - {group: rbac.authorization.k8s.io, resource: clusterrolebindings, verbs: [get, list, watch, create, update, delete]}
```

Only two bindings are written by hand, both at install. kcp creates every binding in a tenant
workspace from the WorkspaceType's `defaultAPIBindings`, claims included. There is no initializer and
no manifest per Tenant.

```yaml
# install #1, in root:system:directory, so the global objects are servable there
apiVersion: apis.kcp.io/v1alpha2
kind: APIBinding
metadata: {name: tenancy-platform}
spec:
  reference: {export: {path: root:system:controllers, name: tenancy-platform}}
---
# install #2, in root:tenants (the parent), so Tenant workspaces can be created without admin
apiVersion: apis.kcp.io/v1alpha2
kind: APIBinding
metadata: {name: tenancy-provisioner}
spec:
  reference: {export: {path: root:system:controllers, name: tenancy-provisioner}}
  permissionClaims:
    - group: tenancy.kcp.io
      resource: workspaces
      verbs: [get, list, watch, create, delete]
      identityHash: <tenancy.kcp.io identity>   # empty here means no claim, silently
      state: Accepted
      # v1alpha2 requires an accepted claim to say which objects it covers. matchAll, because the
      # platform has to see every Workspace in this tier. Narrowing by label would let a workspace
      # opt out of being managed by the thing that creates it.
      selector: {matchAll: true}
```
