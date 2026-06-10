# ADR 008: Platform Mesh Backup and Restore

| Status  | Proposed   |
|---------|------------|
| Date    | 2026-04-27 |

## Context and Problem Statement

A Platform Mesh deployment runs a heterogeneous control plane on a host Kubernetes cluster: one or more [kcp](https://github.com/kcp-dev/kcp) shards backed by their own etcd, [CloudNativePG](https://cloudnative-pg.io) (per [ADR 006 — pending](https://github.com/platform-mesh/architecture/pull/31)) hosting the application databases for [OpenFGA](https://openfga.dev) and [Keycloak](https://www.keycloak.org) (per [ADR 007 — pending](https://github.com/platform-mesh/architecture/pull/32)), provider operators, and arbitrary tenant CRs distributed across kcp workspaces. Adopting organizations expect:

* **Continuous backup** with low RPO (Recovery Point Objective) — seconds for transactional state, minutes for the control plane — not nightly snapshots only.
* **Selective restore** at the granularity of a single workspace, tenant, or component; not just whole-platform recovery.
* A **declarative, KRM-driven** backup operator running on the host cluster, configurable entirely through platform-level CRs reconciled by an in-cluster controller. Platform Mesh does not assume a Git source of truth — backup configuration is cluster-resident state, not user-supplied repository contents.
* **Architecture-pinned restores**. A backup taken from a platform with N kcp shards, in-cluster Postgres, and OpenFGA must only restore onto a target with the same topology. The topology must travel inside the backup, so the restore controller can validate before mutating any state.
* **S3-compatible object storage** as the only required storage backend (AWS S3, MinIO, Cloudflare R2, etc.).

No single existing tool covers all of this. kcp has no upstream backup design; etcd has no production-grade continuous WAL streaming; arbitrary KRM resources need a generic backup engine; Postgres already has best-in-class continuous backup tooling that should not be reimplemented. The decision is how to compose these layers under one platform-level API.

## Decision Drivers

* Reuse upstream, production-grade tooling for each storage layer rather than building a parallel backup stack.
* Keep the public API surface small — operators should configure platform backups, not tool-specific knobs per component.
* Make architecture mismatches a hard, early failure at restore time, not a silent partial success.
* Permissive open-source licensing for everything in the default install.
* KRM-native: backup configuration is CRs reconciled in-cluster, status reflected on those CRs, no out-of-band CLI required for routine operation. The platform does not depend on Git or any external configuration source.

## Considered Options

### A. General-purpose Kubernetes backup operators (orchestration layer)

| Tool | License | Continuous | Selective restore | kcp-aware | Notes |
|---|---|---|---|---|---|
| [Velero](https://velero.io) (1.18+) | Apache-2.0 | Schedule-driven; Kopia-backed data mover; CSI volume-group snapshots | Namespace, label, resource type, cross-namespace mapping | No — extensible via [`BackupItemAction`/`RestoreItemAction` plugins](https://velero.io/docs/v1.18/custom-plugins/) | Mature, broadly deployed, plugin SDK is the standard extension point |
| [Kanister](https://kanister.io) | Apache-2.0 (CNCF Sandbox) | Per-action; Blueprints can chain WAL/incremental commands | Per ActionSet | No | Strong as a building block underneath an operator, weak as the top-level controller |
| [KubeStash](https://kubestash.com) (Stash 2.0) | Source-available; DB add-ons require commercial license | Schedule-driven | Per snapshot | No | Licensing makes it awkward for a fully open default |
| [K8up](https://k8up.io) | Apache-2.0 | Schedule-driven, Restic-only | PVC-level | No | PV-only; does not handle API objects or databases |
| Kasten K10, TrilioVault, CloudCasa | Commercial | Yes | Yes | No | Comparison only; license precludes embedding |

### B. Component-native continuous backup (storage layer)

* **PostgreSQL** — CloudNativePG natively performs continuous WAL archiving to any [Barman-Cloud](https://cloudnative-pg.io/docs/devel/appendixes/backup_barmanobjectstore)-supported object store and supports PITR to any second between the first base backup and the latest archived WAL. The [pgBackRest CNPG-I plugin](https://github.com/operasoftware/cnpg-plugin-pgbackrest) is an alternative for shops standardising on pgBackRest. This is the right primitive for OpenFGA, Keycloak, and any other CNPG-backed component.
* **etcd** — The strongest open-source option is [etcd-druid](https://github.com/gardener/etcd-druid) (Gardener), which provisions etcd clusters as `Etcd` CRs and pairs each member with the [etcd-backup-restore](https://github.com/gardener/etcd-backup-restore) sidecar. The sidecar takes scheduled **full snapshots** plus **delta snapshots driven by the etcd Watch API** (default `delta-snapshot-period` 20 s) directly to S3/GCS/ABS/Swift/OSS/ECS/OCS/local; restore replays the latest full snapshot followed by sequential deltas. This delivers continuous, seconds-RPO etcd backup as a first-class capability — it is the only mature open-source path that does. Alternatives — `etcdctl snapshot save` cron jobs, the dead [CoreOS etcd-operator](https://github.com/coreos/etcd-operator), [Giant Swarm's etcd-backup-operator](https://github.com/giantswarm/etcd-backup-operator) — all cap RPO at the snapshot interval (5–60 min).
* **kcp** — No upstream backup proposal. Workspaces are reconstructed by snapshotting the underlying shard etcd plus the root workspace metadata; per-workspace (logical-cluster) selectivity is achieved at restore time by filtering the etcd snapshot via a kcp-aware Velero plugin (see Decision Outcome §6). The alternative — iterating logical clusters and dumping `Workspace`/`APIBinding`/`APIExport`/CR YAML through the front-proxy — is rejected: it is a per-object approach (out of scope, see *Scope and Non-Goals*) and cannot match the seconds-RPO contract etcd-druid delivers.
* **OpenFGA** — Authoritative state is split. The OpenFGA **model** and **seed tuples** are not held inside OpenFGA at rest; they live on `Store` and `AuthorizationModel` CRs in kcp ([platform-mesh/security-operator](https://github.com/platform-mesh/security-operator)) and are reconciled into OpenFGA by the security-operator via `CreateStore`/`WriteAuthorizationModel`. **Account/role tuples** are derived from kcp `LogicalCluster` watches by the same operator. Only **user-grant tuples** written by [iam-service](https://github.com/platform-mesh/iam-service) are uniquely authoritative in OpenFGA's Postgres. Tuples embed `originClusterID` strings, so kcp logical-cluster IDs must be preserved or remapped on restore. This has been done before in the 0.3 migration ([migrate-openfga-tuples.sh](https://github.com/platform-mesh/helm-charts/blob/main/docs/migration-0.3/migrate-openfga-tuples.sh)), so the approach is proven feasible if it turns out to be needed at restore time. `storeId` and `authorizationModelId` live only in CR status and are recreated by the operator on restore, not preserved.
* **Keycloak** — The Keycloak Operator's `KeycloakRealmImport` is creation-only and does not manage stateful data (users, sessions). Authoritative backup is the Postgres database; realm-import CRs are a useful declarative complement, not a backup.

### C. Architecture-pinning prior art

* [Velero](https://velero.io/docs/v1.18/how-velero-works/) stamps every backup with `velero.io/source-cluster-k8s-gitversion` annotations and an API group/version inventory in `metadata.json`; the restore controller validates API availability before applying, but there is no first-class topology object.
* [CloudNativePG](https://cloudnative-pg.io/documentation/current/backup/) writes the full `Cluster` manifest into the Barman object store with each base backup; recovery requires a new `Cluster` referencing the bucket and CNPG validates Postgres major-version compatibility before bootstrapping. The "manifest-in-bucket" pattern is the cleanest precedent.

## Decision Outcome

Chosen option: **a thin Platform Mesh backup operator that orchestrates Velero, CloudNativePG, and per-shard etcd snapshots, with a `topology.json` manifest written into the same S3 prefix as the data.**

Concretely:

1. **Public API** — a `PlatformBackup` and `PlatformRestore` CRD owned by a new `platform-mesh-backup-operator` running on the host cluster. The CRs carry the S3 endpoint/credentials reference, schedule and retention, the component selector for selective restore, and a target topology reference for restore-time validation. This is the only surface most operators will touch.
2. **Generic KRM resources** — delegated to [Velero 1.18+](https://velero.io) with the [Kopia](https://kopia.io) data mover for object-storage-backed volume backups. A custom `platform-mesh-velero-plugin` adds kcp logical-cluster awareness (`BackupItemAction`/`RestoreItemAction` v2) so backups can be filtered and restored at the workspace level.
3. **Postgres / OpenFGA / Keycloak data** — delegated to CloudNativePG's native continuous WAL archiving to S3. The operator does not reimplement Postgres backup; it reconciles the `barmanObjectStore` configuration on each `Cluster`. PITR is a CNPG primitive.
4. **etcd per kcp shard** — delegated to [etcd-druid](https://github.com/gardener/etcd-druid). Each kcp shard's etcd is provisioned as an `Etcd` CR with backup configured against the same S3 bucket: full snapshots on a schedule (default 24 h) plus Watch-driven delta snapshots (default 20 s `delta-snapshot-period`). The platform operator only reconciles the shard-to-`Etcd`-CR mapping and pushes on-demand full-snapshot triggers driven by `PlatformBackup` events; etcd-druid owns the snapshot lifecycle and restore. This gives the control plane a seconds-RPO contract aligned with the Postgres/CNPG layer.
5. **Architecture pinning** — every backup writes a `topology.json` alongside the data: host-cluster Kubernetes version, kcp shard count and shard identities, CNPG `Cluster` spec digests, OpenFGA store identifiers, Keycloak realm list, and the operator's own version. `PlatformRestore` refuses to proceed unless the target topology matches. Override is possible only via an explicit `spec.allowTopologyMismatch` field with a written reason recorded in status — same spirit as `kubectl --force`.
6. **Selective restore** — implemented at two levels: (a) component (e.g. "restore only the OpenFGA database to T-1h") via CNPG PITR plus the Velero plugin's resource selector, and (b) workspace (e.g. "restore workspace `:root:tenants:acme` to yesterday") via the Velero plugin's logical-cluster filter applied against the etcd snapshot.
7. **Cross-component consistency via tiered sources of truth** — there is no single source of truth across kcp, OpenFGA, and Keycloak, so the operator does not attempt to take cross-system point-in-time snapshots. Instead, each piece of authoritative state is assigned to exactly one backup tier, and a restore-time reconciler closes the boundary gaps:
   * **kcp etcd** is the source of truth for platform topology and any state held on CRs in kcp — including the OpenFGA `Store` / `AuthorizationModel` CRs (model + seed tuples) and all `Account` / `LogicalCluster` resources from which tuples are derived. Restoring kcp etcd is sufficient to drive the security-operator to rebuild the model and the derivable tuple set.
   * **CNPG Postgres** is the source of truth only for state that is uniquely authored against the application, not against kcp — chiefly iam-service user-grant tuples in OpenFGA and Keycloak users/sessions/credentials. CNPG continuous WAL backup covers this.
   * **Synchronised capture is deliberately deferred.** A coordinated admission freeze across kcp, CNPG, and provider controllers would require cross-cutting changes at every layer of the platform; the POC must first establish whether the restore reconciler alone — operating on independently-captured etcd-druid deltas and CNPG WAL — produces an acceptable post-restore state. Only if it does not is a coordinated capture path (admission webhook block, CNPG checkpoint nudge, shared logical marker in `topology.json`) added in a later iteration. Until then, captures are independent and the cross-tier gap is closed entirely by reconciliation.
   * **A `PlatformRestore` reconciler** runs after restore to close the cross-tier boundary: it remaps `originClusterID` strings in OpenFGA tuples if shard cluster IDs changed (existing prior art), deletes orphan tuples whose kcp peer is gone, and re-runs the security-operator to re-derive any account tuples missing in OpenFGA. The reconciler does not attempt to recover Keycloak sessions or any in-flight controller state — both are explicitly out of scope.
8. **Storage** — S3-compatible only. The operator validates bucket reachability and write permission as part of `PlatformBackup` admission.

### Consequences

* Good, because each storage layer is backed up by the upstream tool that already does it best, and the operator stays small.
* Good, because a single `PlatformBackup` CR replaces a half-dozen tool-specific configurations.
* Good, because architecture-pinned restore turns silent topology mismatches into early, explicit failures.
* Good, because Velero's plugin SDK is a stable, well-trodden extension point for the kcp-specific work — the only genuinely novel code we ship.
* Good, because S3 is the single storage contract; no additional infrastructure is required for adopters.
* Bad, because cross-component point-in-time consistency is not guaranteed; the restore reconciler closes boundary gaps (orphan tuples, cluster-ID remapping) but cannot recover in-flight controller state, Keycloak sessions, or any uniquely-Postgres state authored after the most recent CNPG WAL flush.
* Bad, because the operator must track three upstreams (Velero, CNPG, etcd-druid) and absorb their breaking changes.
* Bad, because adopting etcd-druid implies provisioning kcp shard etcd through it, which is a deployment-shape constraint (vs. raw StatefulSets) that should be reflected in the kcp shard installer.
* Bad, because skipping a coordinated capture means the restore reconciler is on the critical path for cross-tier correctness; if its repair logic is incomplete for some boundary case, post-restore state may be subtly wrong rather than loudly broken.

## Scope and Non-Goals (v1)

* `topology.json` is versioned and owned **alongside the backup & restore operator**
* etcd provisioning is via **etcd-druid** — already in use in Platform Mesh. Other shapes (raw StatefulSets, external etcd) are out of scope.
* `originClusterID` **remapping is not implemented in v1**. The operating assumption is that kcp logical-cluster IDs are preserved by the restore process. Remapping remains feasible if later required ([migrate-openfga-tuples.sh](https://github.com/platform-mesh/helm-charts/blob/main/docs/migration-0.3/migrate-openfga-tuples.sh) is prior art).
* Backup of the host cluster's own control-plane resources (operators, RBAC, Helm releases) is **out of scope**; the target is Platform Mesh state only. This can be extended later if needed.
* Backup encryption at rest and KMS integration is **out of scope** for v1. CNPG and Velero both support SSE-KMS natively when adopters need it; the operator surfacing a single key reference is a follow-up.
* **Per-object backup and restore is out of scope.** The operator does not back up or restore individual KRM objects (a single `Workspace`, `Account`, CR instance, OpenFGA tuple, Keycloak user, etc.). Backup and restore granularity is the platform — and, as a follow-up, the component or workspace tier described in the selective-restore section. Recovering an individual object is a user-level concern handled outside this operator (e.g. re-applying a YAML manifest, re-issuing a tuple write).

## Risks and Iteration Plan

The biggest risk in this ADR is that **none of the cross-component story is proven end-to-end yet**. Each upstream tool (Velero, CNPG, etcd-druid) works in isolation; composing them under a single `PlatformBackup`/`PlatformRestore` API with a tiered-source-of-truth restore reconciler is novel work for this project.

The intended path is therefore **POC-first, productionalisation later**:

1. **WIP/POC** — build the minimum end-to-end loop that proves a Platform Mesh cluster can be backed up and restored onto an identically-shaped target. Stub the operator surface, hard-code component selection, use a single shard. The goal is to validate the tiered-restore + reconcile model and the `topology.json` flow, not to ship a stable API. **Explicitly test without any coordinated capture** — independent etcd-druid and CNPG backups plus the restore reconciler — because adding cross-platform admission/checkpoint synchronisation later is a large, cross-cutting change that should only be undertaken if the POC proves the un-coordinated path leaves boundary states the reconciler cannot close.
2. **Productionalise only what the POC proves.** Anything the POC cannot validate (selective workspace restore, multi-shard fan-out, cluster-ID remapping, encryption) is deferred until there is operational evidence the v1 core works.
3. **Iterate fast on the POC** rather than expanding the ADR's scope before the loop closes. Course-correct the ADR from real failure modes encountered during the POC, not from speculative ones.
