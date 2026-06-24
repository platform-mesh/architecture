---
title: Platform Mesh Repository Consolidation — Proposed Repo Disposition

---

# Platform Mesh Repository Consolidation — Proposed Repo Disposition

> Companion working document to [RFC 009: Platform Mesh Repository Consolidation](https://github.com/platform-mesh/architecture/pull/41).
> Scope: **Go-only consolidation** into two repositories — `platform-mesh-operator` (standalone) and
> `platform-mesh` (Go monorepo). Frontend/UI consolidation is **deferred**.
>
> Baseline: all 73 repositories in the `platform-mesh` GitHub organization, verified 2026-06-11
> (`gh repo list`, `go.mod` inspection per repo).

---

## Target Topology

| Target repository | Role |
|---|---|
| **`platform-mesh-operator`** | Standalone installer/lifecycle operator. Deploys the platform; sits *above* the release train. Nothing merges into it. |
| **`platform-mesh`** | Go monorepo. All first-party Go operators, services, libraries, and CLIs. Single release train + independently SemVer'd library submodules. |

> ⚠️ **Naming catch:** the `platform-mesh/platform-mesh` repository name **already exists and is
> archived**. It must be unarchived and repurposed for the monorepo. Upside: the org-level name is
> already reserved, and GitHub redirects from merged repos resolve cleanly.

### Why `platform-mesh-operator` stays standalone

- **Chicken-and-egg:** the operator that rolls out platform version `vX.Y.Z` should not itself be
  versioned *inside* `vX.Y.Z`. It needs an independent lifecycle to orchestrate upgrades of the train.
- **No inbound dependencies:** no repo in the org imports it as a Go module — keeping it out costs
  nothing on the dependency side.
- **Growing responsibility:** per RFC 009 Goal 7, it will deploy resources directly (instead of
  layered Helm charts), strengthening the case for an independent release cadence.
- It should consume `golang-commons` / `subroutines` as **tagged SemVer releases**, like an external
  consumer — this continuously proves the published-module contract works.

---

## `platform-mesh` Monorepo — Repos to Merge In (21)

Proposed layout:

```text
platform-mesh/
├── go.work                  # resolves all sibling modules at HEAD
├── operators/<name>/        # internal "platform" module
├── services/<name>/         # internal "platform" module
├── cmd/<tool>/              # internal "platform" module
├── golang-commons/          # sibling module · independent SemVer
├── subroutines/             # sibling module · independent SemVer
├── apis/                    # sibling module · independent SemVer (CRD types)
└── charts/                  # co-located Helm charts (Wave 3)
```

### Operators → `operators/<name>/` (8)

| # | Repo | Internal deps today | Migration notes |
|---|---|---|---|
| 1 | `account-operator` | commons `v0.17.6`, subroutines `v0.3.3` | Core grouping entity (Account/Workspace). |
| 2 | `security-operator` | commons `v0.17.6`, subroutines `v0.4.3` | |
| 3 | `extension-manager-operator` | commons `v0.17.6`, subroutines `v0.3.3` | Has an `api/` Go submodule pinned externally by `virtual-workspaces` — **extract into the shared `apis/` module**. |
| 4 | `search-operator` | commons `v0.16.11` *(skewed)* | |
| 5 | `resource-sharding-operator` | commons `v0.17.0` *(skewed)* | |
| 6 | `kcp-migration-operator` | commons `v0.13.0` *(most skewed in org)* | |
| 7 | `terminal-controller-manager` | commons `v0.13.23`, subroutines `v0.2.6` *(skewed)* | Ships **2 images** — affects image-component accounting. |
| 8 | `backup-operator` | Contains only `README.md` + `CODEOWNERS` — an empty scaffold. **Do not migrate.** Archive/delete; start as `operators/backup-operator/` in the monorepo when work actually begins. |

### Services → `services/<name>/` (5)

| # | Repo | Internal deps today | Migration notes |
|---|---|---|---|
| 1 | `iam-service` | commons `v0.17.6`, subroutines `v0.3.3` | |
| 2 | `search` | commons `v0.13.24-0.2026…` *(pseudo-version pin — skew poster child)* | |
| 3 | `kubernetes-graphql-gateway` | commons `v0.17.6` | WOULD BE GOOD TO SYNC OUT ON DAY 1 - stagging repo but It has a lot of value to be in monorepo due to tight coupling |
| 4 | `rebac-authz-webhook` | — | No internal deps, but core security path. |
| 5 | `virtual-workspaces` | `extension-manager-operator/api v0.4.9` | Must be its own module (not repository) due to the import of kcp virtual-workspace-framework and kube fork |

### Libraries → sibling SemVer modules at repo root (2 + 1 extracted)

| Module | Source | Why independent SemVer |
|---|---|---|
| `golang-commons` | `golang-commons` repo | The skew bottleneck — **5 live versions** across the org today (`v0.13.0` → `v0.17.6`). Externally importable. |
| `subroutines` | `subroutines` repo | Same pattern — **3 live versions** (`v0.2.6` → `v0.4.3`). Externally importable. |
| `apis` | extracted from `extension-manager-operator/api` (+ other CRD types) | External providers bind against these CRD types; needs a stable, independently tagged contract. |

Tagging: `<module>/vX.Y.Z` (e.g. `golang-commons/v0.18.0`) per Go submodule rules; published
`go.mod` files carry **no `replace` directives**; CI builds each public module with `GOWORK=off`
to prove the tagged graph compiles for external consumers.

### CLIs / tools → `cmd/<tool>/` (2)

| Repo | Migration notes |
|---|---|
| `jl` | Module path still `github.com/openmfp/jl` — fix on the way in. |
| `qbrtool` | No internal deps; trivial move. |

---

## Everything Else (does not enter either target)

### UI — deferred to a future frontend workspace (6)

`portal` · `portal-server-lib` · `portal-ui-lib` · `iam-ui` · `marketplace-ui` · `generic-resource-ui`

### Keep standalone (~17)

| Category | Repos |
|---|---|
| Org / governance | `.github`, `community`, `architecture`, `handbook`, `coderabbit` |
| Docs / website | `platform-mesh.github.io` |
| Build / release infra | `helm-charts` (thins out as app charts co-locate in Wave 3), `ocm`, `external-ocm-components`, `custom-images`, `upstream-images` |
| Admin / trackers | `admin`, `ai-configurations`, `backlog`, `backlog-internal`, `reporting`, `loadtest-internals` |

### Move to `contrib-` per ADR 009 (~10)

`contrib-examples` (already prefixed) · `example-httpbin-operator` · `example-mongodb-multiclusterruntime` ·
`provider-quickstart` · `poc-kcp-intro` · `mongodb-demo` · `samples-opendesk-ocm-landscaper` ·
`poc-maestro` · `onboarding-kcp-intro` · `opendesk`

Some repositories are already planned to be moved into `contrib-examples`, these will be migrated instead of renamed.

### Archive (1 new + 18 already archived)

- **New:** `gardener-workspace` — loose YAML scratchpad (committed `.DS_Store`, sample manifests, no releasable code).
- **Already archived (no action):** `binder-poc`, `demo-collab-hub-app`, `ecofed-demo`,
  `helm-charts-priv`, `httpbin-operator`, `httpbin-ui`, `httpbin-ui2`, `init-agent`,
  `platform-mesh` *(unarchive → becomes the monorepo)*, `poc-api-download`,
  `poc-kcp-observability`, `poc-kcp-scheduler`, `poc-ord-reference-application`,
  `poc-pm-workload-moving`, `poc-pm-workload-selector`, `repository-template`,
  `samples-opendesk-ocm-k8s-toolkit`, `showroom-mcp-demo`.

---

## Disposition Summary

| Disposition | Count | Repos |
|---|---:|---|
| Merge → `platform-mesh` monorepo | **21** | 8 operators + 5     services + 2 libraries + 2 CLIs + 1 extracted `apis` module |
| Keep → `platform-mesh-operator` | **1** | the installer/lifecycle operator |
| Defer (frontend) | 6 | UI repos, future frontend workspace |
| Keep standalone | ~17 | org, docs, build/release infra, trackers |
| Move to `contrib-` | ~10 | examples, samples, POCs, demos |
| Archive (new) | 2 | `backup-operator` (empty scaffold), `gardener-workspace` |
| Already archived | 18 | no action (except unarchiving `platform-mesh` itself) |

**Net effect:** 22 active first-party Go repos → **2**. The `golang-commons` version skew
(5 live versions) and `subroutines` skew (3 live versions) are eliminated the moment Wave 1 lands —
inside the monorepo every component resolves both libraries at HEAD via `go.work`.

---

## Open Questions

1. **`apis` module scope** — only `extension-manager-operator/api`, or consolidate all CRD types
   that external providers bind against?
2. **Unarchive timing** for `platform-mesh/platform-mesh` — Wave 0 scaffold target.

## References

- [RFC 009: Platform Mesh Repository Consolidation (PR #41)](https://github.com/platform-mesh/architecture/pull/41)
- ADR 001: SBOM Generation and OCM Component Restructuring
- ADR 004: Shared CI Workflow Strategy
- ADR 009: Repository Tiers and `contrib-` Prefix
- [Go Modules Reference — workspaces & submodule tagging](https://go.dev/ref/mod)
