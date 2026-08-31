# RFC 010: Platform-Mesh JIT/JEA Support Requirements

| Status  | Proposed      |
|---------|---------------|
| Author  | @BergCyrill   |
| Created | 2026-04-06    |
| Updated | -             |

## 1. Purpose & Audience

This document defines the requirements the platform-mesh product must satisfy **so that a privileged access management integration (JIT/JEA) is enforceable inside platform-mesh's authorization layer**.

## 2. Context & Problem Statement

Platform-mesh authorizes requests through ReBAC: OpenFGA stores identity-based tuples (`user:<sub>` assigned to role objects, accounts, and resources), and two kinds of callers evaluate them:

- The **ReBAC authorization webhook** answers every kcp `SubjectAccessReview` by checking OpenFGA with `user:<subject>` — it ignores `spec.groups` entirely.
- The **IAM service** enforces its GraphQL `authorized` directive by checking OpenFGA with the caller's identity only.

Meanwhile, a PAM integration could deliver a user's effective entitlements — standing workforce roles and time-capped, 4-eye-approved elevations alike — as the JWT `roles` claim (either by using an authentication webhook `TokenReview` adapter or by using an OIDC-based identity provider) prefixed with `oidc:`. Because every OpenFGA decision is keyed on identity alone, those entitlements are invisible exactly where ReBAC is authoritative:

- On the **portal path**, the IAM service consults only OpenFGA; there is no RBAC fallback at all.
- In the **orgs workspace** and **tenant workspaces**, the contextual handler returns NoOpinion on deny, so claim-based access today works only where a parallel kcp RBAC binding happens to exist — leaving list filtering, IAM visibility, and any FGA-only decision blind to elevations.

Writing tuples when an elevation is granted is not an acceptable fix: it introduces the out-of-band entitlement mutation, requires an event transport plus cleanup machinery, and decouples revocation from token TTL. It also shifts the authority from the PAM plane to the platform-mesh product itself. The requirements below therefore mandate **check-time evaluation** of group claims.

## 3. Scope

### In scope

- **All platform-mesh OpenFGA check paths**: the ReBAC authorization webhook (orgs handler and contextual handler) and the IAM service `authorized` directive. Any future component that calls OpenFGA `Check`/`ListObjects` on behalf of an end-user inherits these requirements.
- **OpenFGA model and check-request changes** needed to evaluate group claims.
- **The role catalogue** — the canonical mapping from claim role names to ReBAC semantics.
- **Tenant-scoped claims** (JEA) — confining a claim-derived entitlement to an account subtree.
- **Decision caching and audit behavior** on the affected paths.

### Out of scope

- **The PAM plane itself** — token minting, approval workflows, elevation TTLs.
- **The TokenReview adapter and `oidc:` prefix wiring** — external component/adapter in context of the [modularization RFC](./008-platform-mesh-modularization.md).
- **kcp native RBAC bindings** — kcp's own RBAC continues to evaluate `oidc:`-prefixed groups independently; nothing here changes that layer.
- **Machine identities and break-glass access** — handled by a separate mechanism, not the user-oriented JIT/JEA elevation specified here.

## 4. Glossary

- **PAM** — Privileged Access Management. The external plane and discipline that grants time-bounded, approved, audited privileged access, delivering a user's effective entitlements — standing workforce roles and elevations alike — to platform-mesh as token claims. The PAM plane itself (token minting, approval workflows, elevation TTLs) is out of scope (§3).
- **JIT** — Just-In-Time. Temporary, time-bounded entitlement grants or elevations that reach platform-mesh only via the next-issued token's `roles` claim and expire with it.
- **JEA** — Just Enough Access. Confining a granted entitlement to the narrowest scope needed; here, a claim-derived entitlement bounded to a single account and its child-account subtree (§6.2), as opposed to a platform-global grant.
- **Group claim** — one entry of the user's effective entitlement set as it reaches a platform-mesh check path: an `oidc:`-prefixed group on the SubjectAccessReview path, or an entry of the verified JWT `roles` claim on the GraphQL path.
- **Catalogue role** — a canonical role name (e.g. `viewer`) defined in the platform-mesh-owned role catalogue (§6.3), with defined ReBAC semantics.
- **Claim-derived entitlement** — access granted because a catalogue role appears in the requester's group claims, as opposed to access granted by persisted tuples.
- **Scoped claim** — a group claim carrying a scope component that confines it to an account subtree (§6.2).
- **Check path** — any platform-mesh code path that issues an OpenFGA `Check` or `ListObjects` call to authorize an end-user request.
- **Contextual tuple** — an OpenFGA feature: a tuple supplied with a single check request, evaluated as if stored, but never persisted.

## 5. Division of Authority: Tuples vs. Claims

This split is normative and is the core of the design:

| Source | Carries | Lifecycle |
| ------ | ------- | --------- |
| **Persisted identity tuples** | Standing tenant/workspace **membership**: account owner/member, parent chains, role assignments made through the portal IAM flow | Written by account operator and IAM service; removed by offboarding |
| **Token group claims** | **Catalogue-role entitlements**: standing workforce roles and all JIT/JEA elevations, for any persona | Exist only inside the token; expire with the token; never persisted |

Authorization decisions take the **union**: a request is allowed if persisted tuples grant it, or claim-derived entitlements grant it, or both. Claims never subtract access granted by tuples.

## 6. Functional Requirements

Each requirement uses `REQ-PM-<DOMAIN>-NNN` numbering. RFC 2119 keywords (MUST, SHOULD, MAY) are normative. Verification pointers (`AC-PM-...`) refer to §10.

### 6.1 Claim Evaluation (`REQ-PM-CLAIM-*`)

- **REQ-PM-CLAIM-001** — Every authorization decision platform-mesh makes against OpenFGA MUST evaluate the requester's group claims in addition to persisted tuples, with union semantics (§5). A user holding zero tuples but a sufficient claim is allowed; a user holding sufficient tuples but no claims keeps today's behavior. *Verification: AC-PM-01, AC-PM-07.*
- **REQ-PM-CLAIM-002** — The group set MUST be taken exclusively from the verified authentication context of the request: `SubjectAccessReview.spec.groups` on the kcp path, the verified bearer token on the GraphQL path. Group information from unverified headers, query parameters, or client-asserted fields MUST NOT be used. *Verification: AC-PM-08.*
- **REQ-PM-CLAIM-003** — Token contents MUST NOT cause tuple writes. No component may persist a tuple because a claim was observed; claim-derived access exists only for the lifetime of the token that carries it. *Verification: AC-PM-03, AC-PM-04.*
- **REQ-PM-CLAIM-004** — Platform-mesh MUST apply one canonical normalization before evaluation: the `oidc:` prefix present on the SubjectAccessReview path is handled consistently, and only catalogue roles (§6.3) are evaluated. Non-catalogue groups (e.g. `system:authenticated`), unknown role names, and malformed entries MUST be ignored — no access granted, no request failed. *Verification: AC-PM-08.*
- **REQ-PM-CLAIM-005** — Claim parsing, normalization, and scope resolution MUST be implemented once and shared by all check paths, so that the same token yields the same entitlements on every surface. Per-caller re-implementations are not acceptable. *Verification: AC-PM-06.*

### 6.2 Tenant Scoping — JEA (`REQ-PM-SCOPE-*`)

- **REQ-PM-SCOPE-001** — Platform-mesh MUST support a scope component on group claims that confines the entitlement to a single account and its child-account subtree. Claims without a scope component apply platform-globally. The wire encoding is a joint design decision (see §8.3); the scoping capability itself is normative. *Verification: AC-PM-01.*
- **REQ-PM-SCOPE-002** — A scoped claim MUST have no effect outside its subtree: a check targeting a sibling or ancestor account MUST NOT be influenced by it. *Verification: AC-PM-02.*
- **REQ-PM-SCOPE-003** — A claim whose scope component cannot be parsed, or names an account unknown to the evaluating store, MUST be ignored (fail-closed): no access granted, no request failed. *Verification: AC-PM-08.*

### 6.3 Role Catalogue (`REQ-PM-CAT-*`)

- **REQ-PM-CAT-001** — platform-mesh MUST own and publish the canonical role catalogue: the closed list of claim role names (e.g. `viewer`, `editor`) and, for each, the exact ReBAC semantics (which relations on which object types the role grants). The catalogue MUST be published in a form the PAM team and platform-mesh operations team can reference (role names appear in PAM approval workflows and in kcp RBAC bindings). Changes to the catalogue are coordinated changes. *Verification: AC-PM-09.*
- **REQ-PM-CAT-002** — The catalogue MUST be able to express at minimum a **read-only** level and a **read-write** level distinct from full owner. Note that in the current model the `member` relation grants `update`, `delete`, and `patch` — a viewer role mapped onto `member` would not be read-only. If the current relation granularity cannot express the catalogue, the model MUST evolve; the catalogue is the contract, not the current relations. *Verification: AC-PM-09.*

### 6.4 Surface Parity (`REQ-PM-SURF-*`)

- **REQ-PM-SURF-001** — All authorization webhook handlers MUST evaluate claims. A user whose entitlements exist only as claims MUST NOT be hard-denied at any workspace (e.g. orgs & tenant workspaces). *Verification: AC-PM-05.*
- **REQ-PM-SURF-002** — The IAM service `authorized` directive MUST evaluate claims derived from the same verified bearer token the gateway forwards. This is stated as a requirement on the contract, independent of where the IAM service obtains identity today. *Verification: AC-PM-06.*
- **REQ-PM-SURF-003** — List and filter operations MUST honor claim-derived entitlements equivalently to point checks: `list`/`watch` verb checks, and any object filtering built on `ListObjects` or equivalent, return the same result set for a claim-holder as for a user holding equivalent tuples. A JIT viewer must see in a list what a point `get` would allow. *Verification: AC-PM-09.*

### 6.5 Revocation Latency (`REQ-PM-REV-*`)

- **REQ-PM-REV-001** — Any decision caching on a platform-mesh check path MUST be bounded such that, once a claim no longer appears in the presented token, claim-derived access stops within a prior defined end-to-end budget (e.g. `≤ token-TTL + 60 s`). This explicitly includes the Kubernetes webhook-authorizer allow-cache in front of the ReBAC webhook and any OpenFGA-side or in-process caches. *Verification: AC-PM-03.*

### 6.6 Audit (`REQ-PM-AUDIT-*`)

- **REQ-PM-AUDIT-001** — An allow decision that depends on a claim-derived entitlement MUST be attributable: the decision log carries the user's stable `sub`, the checked relation and object, the specific claim(s) — including scope — that produced the allow, and a high-resolution timestamp, so the event joins with the PAM-plane and kcp audit streams on `sub` and time. This makes a "privileged actions MUST be fully auditable" requirement possible on the platform-mesh side. *Verification: AC-PM-10.*

## 7. Non-Functional Requirements (`REQ-PM-NFR-*`)

Currently numeric latency and throughput targets are intentionally absent until baseline measurements exist; claim evaluation sits on the SubjectAccessReview hot path. Until then the only normative NFR-adjacent constraint is the revocation-cache bound in REQ-PM-REV-001.

## 8. Integration Contracts

### 8.1 What platform-mesh can rely on

- **SubjectAccessReview path.** Groups arrive in `spec.groups` exactly as the TokenReview adapter or kcp OIDC integration returns them: every role from the JWT `roles` claim prefixed with `oidc:`, plus Kubernetes-standard groups such as `system:authenticated`. The prefix is fixed.
- **GraphQL path.** The `kubernetes-graphql-gateway` forwards the user's bearer token unchanged. The token is a JWT: stable `sub`, unprefixed `roles`, verifiable against the PAM plane's JWKS.
- **Identity stability.** The `sub` claim is stable across sessions; persisted tuples remain keyed on it.
- **Revocation model.** No online revocation exists; token TTL bounds claim lifetime. platform-mesh is never asked to react to PAM-plane events out-of-band.

### 8.2 What the PAM plane relies on from platform-mesh

- The role catalogue (REQ-PM-CAT-001) as the single source of truth for claim role names and their semantics.
- Union decision semantics (§5) on every check path.
- No tuple writes triggered by token contents (REQ-PM-CLAIM-003).

### 8.3 Scoped-claim encoding (initial proposal — to be finalized jointly)

The multi-tenant claim encoding remains an open decision. To give that decision a concrete starting point, this document proposes the following initial encoding; the **syntax** below is not yet normative, the **capability** (REQ-PM-SCOPE-001) is.

```text
roles: [
  "viewer",                  // unscoped — applies platform-globally
  "editor@acme/payments"     // scoped — applies to account 'payments'
]                            //   under org 'acme', and its child accounts
```

- A claim is `<role>` or `<role>@<scope>`.
- `<scope>` is the slash-separated account path below the orgs root (`<org>/<account>[/<child-account>...]`), matching the account hierarchy platform-mesh already models via `parent` relations.
- Any final encoding MUST satisfy: unambiguous separator that cannot occur in role names or account names; representable as a flat string (the value transits the JWT `roles` array and kcp group strings); survives `oidc:` prefixing untouched.

### 8.4 Decision-cache disclosure

Platform-mesh MUST document the decision caches present on its check paths (location, key, TTL) so the end-to-end revocation budget of REQ-PM-REV-001 can be verified and, if needed, retuned alongside the PAM plane's token TTL choice.

## 9. Assumptions & Constraints

- There is a clearly defined **interface**. The TokenReview adapter's response shape is fixed.
- **Onboarding/offboarding is unchanged.** Group-claim support adds no new onboarding/offboarding surface, because claims are never persisted.
- **Open items** carried by this document: final scoped-claim encoding (§8.3); catalogue contents (platform-mesh-owned, REQ-PM-CAT-001); numeric NFR budgets (§7, pending baselines).

## 10. Acceptance Criteria

All criteria use Given / When / Then. The platform-mesh team is expected to demonstrate each criterion before group-claim support is considered ready for the PAM integration.

- **AC-PM-01** — *Given* user `alice` holds **no tuples** in account `acme/payments` and presents a token whose claims carry the editor catalogue role scoped to `acme/payments`, *when* she runs `kubectl create` against that workspace, *then* the request succeeds. Covers REQ-PM-CLAIM-001, REQ-PM-SCOPE-001.
- **AC-PM-02** — *Given* the same token, *when* alice issues the same request against sibling account `acme/hr`, *then* the request is denied. Covers REQ-PM-SCOPE-002.
- **AC-PM-03** — *Given* alice's elevation expires in the PAM plane, *when* her next-issued token no longer carries the claim, *then* requests relying on it are denied within `≤ token-TTL + 60 s` of the elevation expiry, including any decision-cache effects. Covers REQ-PM-CLAIM-003, REQ-PM-REV-001.
- **AC-PM-04** — *Given* a tuple-store snapshot taken before alice's elevation, *when* a full elevation cycle completes (grant, active use on both surfaces, expiry), *then* a second snapshot shows **no tuple difference** attributable to the cycle. Covers REQ-PM-CLAIM-003.
- **AC-PM-05** — *Given* a workforce user whose entitlements exist only as group claims (zero tuples), *when* they list workspaces in the orgs workspace, *then* the orgs handler does not hard-deny and the listing reflects their claim-derived entitlements. Covers REQ-PM-SURF-001.
- **AC-PM-06** — *Given* the same elevated session, *when* alice performs the corresponding operation through the portal (GraphQL path, IAM service directive), *then* the decision matches the kubectl path — same token, same entitlements, same outcome. Covers REQ-PM-SURF-002, REQ-PM-CLAIM-005.
- **AC-PM-07** — *Given* a tenant owner whose access is granted by persisted tuples and whose token carries no catalogue-role claims, *when* they exercise their standing access, *then* behavior is unchanged from before group-claim support (regression guard). Covers REQ-PM-CLAIM-001.
- **AC-PM-08** — *Given* a token carrying an unknown role name, a `system:*` group, and a claim with a malformed scope, plus a request attempting to assert additional groups through unverified request metadata (e.g. a client-set header), *when* the request is evaluated, *then* none of these entries grant access and none cause a request failure. Covers REQ-PM-CLAIM-002, REQ-PM-CLAIM-004, REQ-PM-SCOPE-003.
- **AC-PM-09** — *Given* alice holds a read-only catalogue role for an account via claims, *when* she lists resources there, *then* the list contains exactly the resources a point `get` would allow, and write verbs (`create`, `update`, `delete`, `patch`) are denied. Covers REQ-PM-SURF-003, REQ-PM-CAT-001, REQ-PM-CAT-002.
- **AC-PM-10** — *Given* a one-hour window containing claim-based allows, *when* platform-mesh decision logs are joined with PAM-plane and kcp audit on `sub` and timestamp, *then* every claim-based allow identifies the claim (including scope) that produced it. Covers REQ-PM-AUDIT-001.

### Coverage table

| AC ID     | Covers REQs                                          |
| --------- | ---------------------------------------------------- |
| AC-PM-01  | REQ-PM-CLAIM-001, REQ-PM-SCOPE-001                   |
| AC-PM-02  | REQ-PM-SCOPE-002                                     |
| AC-PM-03  | REQ-PM-CLAIM-003, REQ-PM-REV-001                     |
| AC-PM-04  | REQ-PM-CLAIM-003                                     |
| AC-PM-05  | REQ-PM-SURF-001                                      |
| AC-PM-06  | REQ-PM-SURF-002, REQ-PM-CLAIM-005                    |
| AC-PM-07  | REQ-PM-CLAIM-001                                     |
| AC-PM-08  | REQ-PM-CLAIM-002, REQ-PM-CLAIM-004, REQ-PM-SCOPE-003 |
| AC-PM-09  | REQ-PM-SURF-003, REQ-PM-CAT-001, REQ-PM-CAT-002      |
| AC-PM-10  | REQ-PM-AUDIT-001                                     |

## 11. Reference Design (non-normative)

This appendix sketches one way to satisfy the requirements, to demonstrate feasibility. Platform-mesh is free to choose a different design that meets §6.

### 11.1 Contextual tuples, no model change

OpenFGA contextual tuples allow a check path to supply tuples that are evaluated as if stored but never persisted — a direct fit for REQ-PM-CLAIM-003. The minimal variant needs **no model change**, because `role#assignee` already admits `user` subjects and `owner`/`member` already admit `role#assignee`. The shared evaluation component (REQ-PM-CLAIM-005) resolves the requester's claims against the catalogue and the check's target account, and injects the same two-tuple pattern the IAM service persists today — contextually:

```json
{
  "storeId": "<account-store-id>",
  "tupleKey": {
    "object": "apps_deployment:<workspace-cluster-id>/demo",
    "relation": "update",
    "user": "user:alice"
  },
  "contextualTuples": {
    "tupleKeys": [
      {
        "user": "user:alice",
        "relation": "assignee",
        "object": "role:core_platform-mesh_io_account/<origin>/payments/editor"
      },
      {
        "user": "role:core_platform-mesh_io_account/<origin>/payments/editor#assignee",
        "relation": "<catalogue-mapped-relation>",
        "object": "core_platform-mesh_io_account:<origin>/payments"
      }
    ]
  }
}
```

(The `parent` contextual tuples the webhook already injects today are omitted for brevity.) Scope resolution happens in the caller: the claim `editor@acme/payments` produces these contextual tuples only when the check targets the `payments` subtree; for `acme/hr` nothing is injected (REQ-PM-SCOPE-002 by construction). Unscoped claims inject against whichever account the check targets. The orgs handler injects analogous tuples against `tenancy_kcp_io_workspace:orgs`. `ListObjects` accepts the same contextual tuples, covering REQ-PM-SURF-003.

### 11.2 Optional model variant for legibility

A `group` type (`define member: [user]`) with `role#assignee` extended to `[user, group#member]` makes group-derived access first-class in the model and shrinks per-check payloads (membership becomes the only contextual tuple). This is an internal trade-off — payload size versus model migration — that does not change the normative requirements.

### 11.3 Considered and rejected: event-driven persisted tuples

Having the PAM plane emit elevation events that a controller turns into persisted (or OpenFGA-conditional, expiring) tuples was considered and rejected: it violates the propagate-only-via-token rule requirement, adds an event transport and cleanup machinery whose failure modes extend privileged access, and decouples revocation from token TTL.

## 12. References

- [RFC 008: A Modular and Layered Framework for Platform Mesh](./008-platform-mesh-modularization.md)
