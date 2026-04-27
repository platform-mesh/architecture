# ADR 008: Platform Mesh Backup and Restore

| Status  | Proposed   |
|---------|------------|
| Date    | 2026-04-27 |

## Context and Problem Statement

A Platform Mesh deployment runs a heterogeneous control plane on a host Kubernetes cluster: one or more [kcp](https://github.com/kcp-dev/kcp) shards backed by their own etcd, [CloudNativePG](https://cloudnative-pg.io) (per [ADR 006 — pending](https://github.com/platform-mesh/architecture/pull/31)) hosting the application databases for [OpenFGA](https://openfga.dev) and [Keycloak](https://www.keycloak.org) (per [ADR 007 — pending](https://github.com/platform-mesh/architecture/pull/32)), provider operators, and arbitrary tenant CRs distributed across kcp workspaces. Adopting organizations expect:

* **Continuous backup** with low RPO (Recovery Point Objective) — seconds for transactional state, minutes for the control plane — not nightly snapshots only.
* **Selective restore** at the granularity of a single workspace, tenant, or component; not just whole-platform recovery.
* A **declarative, KRM-driven** backup operator running on the host cluster, configurable through platform-level CRs that fit a GitOps workflow.
* **Architecture-pinned restores**. A backup taken from a platform with N kcp shards, in-cluster Postgres, and OpenFGA must only restore onto a target with the same topology. The topology must travel inside the backup, so the restore controller can validate before mutating any state.
* **S3-compatible object storage** as the only required storage backend (AWS S3, MinIO, Cloudflare R2, etc.).

No single existing tool covers all of this. kcp has no upstream backup design; etcd has no production-grade continuous WAL streaming; arbitrary KRM resources need a generic backup engine; Postgres already has best-in-class continuous backup tooling that should not be reimplemented. The decision is how to compose these layers under one platform-level API.

## Decision Drivers

* Reuse upstream, production-grade tooling for each storage layer rather than building a parallel backup stack.
* Keep the public API surface small — operators should configure platform backups, not tool-specific knobs per component.
* Make architecture mismatches a hard, early failure at restore time, not a silent partial success.
* Permissive open-source licensing for everything in the default install.
* GitOps-friendly: backup configuration as KRM, status reflected on CRs, no out-of-band CLI required for routine operation.

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
* **kcp** — No upstream backup proposal. Workspaces can be reconstructed by either snapshotting the underlying shard etcd plus the root workspace metadata, or by iterating logical clusters and dumping `Workspace`/`APIBinding`/`APIExport`/CR YAML through the front-proxy. The latter is the only path that supports per-workspace selective restore.
* **OpenFGA** — State lives entirely in its database; backing up Postgres is sufficient.
* **Keycloak** — The Keycloak Operator's `KeycloakRealmImport` is creation-only and does not manage stateful data (users, sessions). Authoritative backup is the Postgres database; realm imports are a useful GitOps complement, not a backup.

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
6. **Selective restore** — implemented at two levels: (a) component (e.g. "restore only the OpenFGA database to T-1h") via CNPG PITR plus the Velero plugin's resource selector, and (b) workspace (e.g. "restore workspace `:root:tenants:acme` to yesterday") via the Velero plugin's logical-cluster filter applied against the etcd snapshot. Cross-component point-in-time consistency is best-effort in v1 — operators choosing PITR across etcd and Postgres accept bounded inconsistency reconciled at restore time.
7. **Storage** — S3-compatible only. The operator validates bucket reachability and write permission as part of `PlatformBackup` admission.

### Consequences

* Good, because each storage layer is backed up by the upstream tool that already does it best, and the operator stays small.
* Good, because a single `PlatformBackup` CR replaces a half-dozen tool-specific configurations and fits GitOps workflows.
* Good, because architecture-pinned restore turns silent topology mismatches into early, explicit failures.
* Good, because Velero's plugin SDK is a stable, well-trodden extension point for the kcp-specific work — the only genuinely novel code we ship.
* Good, because S3 is the single storage contract; no additional infrastructure is required for adopters.
* Bad, because cross-component point-in-time consistency is not guaranteed in v1; tenants requiring transactional consistency across kcp and Postgres will need a future iteration.
* Bad, because the operator must track three upstreams (Velero, CNPG, etcd-druid) and absorb their breaking changes.
* Bad, because adopting etcd-druid implies provisioning kcp shard etcd through it, which is a deployment-shape constraint (vs. raw StatefulSets) that should be reflected in the kcp shard installer.
* Bad, because GitOps round-trips need a documented precedence rule when a restored CR conflicts with what Argo/Flux will reconcile next; this must be addressed before GA.

## Open Questions

* Should `topology.json` be a versioned schema owned by this repository, or live alongside the operator's CRDs?
* Whether the kcp shard installer should adopt etcd-druid as the default etcd provisioner, or keep raw StatefulSets and treat etcd-druid as opt-in for backup-enabled deployments.
* Backup of the host cluster's own controller-plane resources (operators, RBAC) — in scope for `PlatformBackup`, or assumed to be GitOps-reconciled?
* Encryption at rest and key management for backups in S3 — likely SSE-KMS via CNPG and Velero's native settings, but the operator should expose a single key reference.
