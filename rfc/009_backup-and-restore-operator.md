# RFC 009: Platform Mesh Backup and Restore Operator (POC/MVP)

| Status  | Proposed                                                                |
|---------|-------------------------------------------------------------------------|
| Author  | @mjudeikis                                                              |
| Created | 2026-05-20                                                              |
| Updated | 2026-05-20                                                              |
| RFC PR  | TBD                                                                     |
| ADR     | [ADR 008](../adr/008-platform-mesh-backup-and-restore.md)               |

## Summary

[ADR 008](../adr/008-platform-mesh-backup-and-restore.md) chose to build a thin Platform Mesh backup and restore operator that orchestrates [Velero](https://velero.io), [CloudNativePG](https://cloudnative-pg.io), and [etcd-druid](https://github.com/gardener/etcd-druid) under a single platform-level API, with a `topology.json` manifest for architecture-pinned restore and a cross-tier reconciler closing consistency gaps.

This RFC scopes the **POC/MVP implementation** of that operator. The goal is the smallest end-to-end loop that proves a Platform Mesh cluster can be backed up to S3 and restored onto an identically-shaped target — independent captures, no platform-wide synchronisation, multi-shard kcp topology covered from day one (Platform Mesh always runs with at least two shards), no selective workspace restore. Productionalisation follows only what the POC proves.

## Motivation

Nothing in ADR 008's cross-component story is exercised end-to-end yet. Each upstream (Velero, CNPG, etcd-druid) works in isolation, but composing them under one CRD-driven API with a tiered restore reconciler is novel for this project. The risks worth retiring fast:

1. Can independent etcd-druid + CNPG captures be reconciled into a usable post-restore state without a platform-wide admission freeze?
2. Does the `topology.json` "manifest-in-bucket" pattern catch the architecture mismatches we actually care about?
3. How brittle is the restore sequence when the target cluster's logical-cluster IDs are preserved (the assumption made in ADR 008 for v1)?
4. What does the surface area of cross-tier repair look like in practice — orphan tuples, missing account-derived tuples, security-operator re-derivation lag?

An RFC-scoped POC answers these before the operator's public API hardens.

## Scope

### In Scope (POC/MVP)

- A new repository `platform-mesh/backup-operator` running on the host cluster.
- `PlatformBackup` and `PlatformRestore` CRDs (group `backup.platform-mesh.io/v1alpha1`).
- **Velero lifecycle management** — the operator installs Velero CRDs, the Velero server `Deployment`, the node-agent `DaemonSet`, and reconciles `BackupStorageLocation` from `PlatformBackup.spec.storage`. Adopters never `helm install` Velero separately.
- **Multi-shard kcp etcd** via etcd-druid `Etcd` CR snapshot triggers — every shard discovered in the platform namespace is captured and restored in the same `PlatformBackup`/`PlatformRestore` lifecycle.
- One backup component: **CNPG `Cluster`s** via CNPG on-demand `Backup` CR.
- One backup component: **Velero `Backup`** for arbitrary KRM resources (account CRs, security-operator `Store`/`AuthorizationModel`, miscellaneous CRDs).
- `topology.json` writer/validator with multi-shard schema (shard list with per-shard digests).
- A restore reconciler that issues etcd-druid restore, CNPG `Cluster` recovery, Velero `Restore`, and a basic cross-tier repair pass (orphan-tuple sweep, security-operator nudge).
- End-to-end happy-path test on a Kind-based Platform Mesh quick-start cluster.

### Out of Scope (deferred)

- Selective workspace / tenant restore.
- **Per-object backup and restore** — the operator does not back up or restore individual KRM objects (a single `Workspace`, `Account`, CR instance, OpenFGA tuple, Keycloak user, etc.). Granularity is the platform; recovering a single object is a user-level concern handled outside this operator.
- Cluster-ID remapping (ADR 008 assumes IDs preserved; prior art exists if proven needed later).
- Encryption / KMS surface (CNPG and Velero handle it natively when adopters need it).
- Backup of the host cluster's own control plane (operators, RBAC, Helm releases).
- Coordinated admission freeze / shared logical capture marker — explicitly tested *without*, per ADR 008.
- Production-grade scheduling, retention, GC policies. The POC takes on-demand backups only.
- Multi-tenant backup configuration (per-org `PlatformBackup` CRs).
- UI integration.

## Design Principles

1. **Orchestrate, don't reimplement.** Every storage layer is backed up by its upstream tool. The operator only reconciles the upstream's CRs and reports their status.
2. **Independent captures by default.** No platform-wide synchronisation. The cross-tier reconciler at restore time is the consistency mechanism. If the POC proves this insufficient, that finding drives a follow-up RFC — not preemptive freeze plumbing.
3. **One CRD, one verb.** `PlatformBackup` triggers a snapshot; `PlatformRestore` triggers a restore. No mixed-mode resources.
4. **Topology manifest is authoritative.** The presence and shape of `topology.json` in the backup prefix decides whether a restore can proceed. The operator never restores a backup it cannot validate.
5. **Subroutine-based reconciliation** ([RFC 002](002-runtime-lifecycle-v2.md)). Each external system the operator drives — etcd-druid, CNPG, Velero, topology, repair — is one subroutine. Failure isolation, status, and retries follow the platform's lifecycle conventions.
6. **POC code is throwaway-friendly.** Hard-code component selection, accept a fixed S3 bucket layout, no migrations. Stable API ergonomics are a post-POC concern.

## Architecture Overview

The operator runs as a `Deployment` on the host cluster — the same cluster where etcd-druid and CNPG already run as Platform Mesh prerequisites. **Velero is owned by the backup-operator itself**, not assumed to be pre-installed: it is purely a backup-side concern and the operator manages its lifecycle (CRDs, server `Deployment`, `BackupStorageLocation`, node-agent `DaemonSet`) so adopters do not have to install or version-track it separately. The operator watches its own CRDs and reacts by creating/annotating CRs owned by etcd-druid, CNPG, and Velero.

```
                        ┌────────────────────────────────────┐
                        │   backup-operator (host cluster)   │
                        │                                    │
   PlatformBackup ──────┤  PlatformBackupReconciler          │
                        │     ├── topology subroutine        │
                        │     ├── etcd-druid subroutine ─────┼──► Etcd CR (snapshot trigger)
                        │     ├── cnpg subroutine ───────────┼──► CNPG Backup CR
                        │     ├── velero subroutine ─────────┼──► Velero Backup CR
                        │     └── manifest writer ───────────┼──► S3 (topology.json)
                        │                                    │
   PlatformRestore ─────┤  PlatformRestoreReconciler         │
                        │     ├── topology validator         │
                        │     ├── etcd restore subroutine ───┼──► Etcd CR (restore mode)
                        │     ├── cnpg restore subroutine ───┼──► CNPG Cluster (bootstrap.recovery)
                        │     ├── velero restore subroutine ─┼──► Velero Restore CR
                        │     └── repair subroutine ─────────┼──► OpenFGA tuple sweep
                        └────────────────────────────────────┘
                                          │
                                          ▼
                              ┌──────────────────────┐
                              │  S3-compatible bucket │
                              │  /<backup-id>/         │
                              │    topology.json       │
                              │    etcd/...            │
                              │    cnpg/...            │
                              │    velero/...          │
                              └──────────────────────┘
```

All snapshot/restore data is written by the upstream tools to their existing S3 prefixes (etcd-druid → `etcd/`, CNPG Barman → `cnpg/`, Velero → `velero/`). The operator's only direct S3 write is `topology.json` at the backup root, plus a small `index.json` cross-referencing the upstream artefacts so a restore can find them.

## API Scaffolding

CRDs live under `backup.platform-mesh.io/v1alpha1`. Group name reserved; not bound to a specific Go module path.

### `PlatformBackup`

Triggers a single backup. POC has no scheduling — a `CronJob` or controller-driven schedule is layered later.

```yaml
apiVersion: backup.platform-mesh.io/v1alpha1
kind: PlatformBackup
metadata:
  name: poc-2026-05-20-001
spec:
  storage:
    s3:
      endpoint: s3.example.com
      bucket: platform-mesh-backups
      region: eu-west-1
      credentialsRef:
        name: backup-s3-creds       # Secret with accessKey/secretKey
  components:
    etcd: { enabled: true }         # POC: drives all etcd-druid Etcd CRs in the platform namespace
    cnpg: { enabled: true }         # POC: drives all CNPG Clusters in the platform namespace
    velero: { enabled: true }       # POC: a single fixed include list
status:
  phase: Succeeded                  # Pending | Capturing | WritingManifest | Succeeded | Failed
  backupID: poc-2026-05-20-001-0a3f
  topologyDigest: sha256:...
  artefacts:
    etcd:    { snapshotID: full-2026-05-20T13-04-22Z, delta: ... }
    cnpg:    { backupName: cnpg-on-demand-... }
    velero:  { backupName: velero-... }
  conditions:
    - type: TopologyCaptured
    - type: EtcdSnapshotted
    - type: CNPGSnapshotted
    - type: VeleroBackedUp
    - type: ManifestWritten
```

### `PlatformRestore`

```yaml
apiVersion: backup.platform-mesh.io/v1alpha1
kind: PlatformRestore
metadata:
  name: restore-poc-2026-05-20
spec:
  source:
    storage:
      s3:
        endpoint: s3.example.com
        bucket: platform-mesh-backups
        credentialsRef: { name: backup-s3-creds }
    backupID: poc-2026-05-20-001-0a3f
  topologyValidation: Strict        # POC supports Strict only; AllowMismatch deferred
status:
  phase: ValidatingTopology         # ValidatingTopology | RestoringEtcd | RestoringCNPG | RestoringVelero | Repairing | Succeeded | Failed
  topologyValidation:
    sourceDigest: sha256:...
    targetDigest: sha256:...
    matches: true
  conditions:
    - type: TopologyValid
    - type: EtcdRestored
    - type: CNPGRestored
    - type: VeleroRestored
    - type: RepairCompleted
```

### `topology.json` (schema sketch)

```json
{
  "schemaVersion": "v1alpha1",
  "capturedAt": "2026-05-20T13:04:22Z",
  "hostCluster": {
    "kubernetesVersion": "v1.32.4",
    "namespace": "platform-mesh"
  },
  "kcp": {
    "shardCount": 2,
    "shards": [
      { "name": "root",  "etcdRef": "etcd/root",  "logicalClusterIDsDigest": "sha256:..." },
      { "name": "shard-a", "etcdRef": "etcd/shard-a", "logicalClusterIDsDigest": "sha256:..." }
    ]
  },
  "cnpg": {
    "clusters": [
      { "name": "openfga-db", "specDigest": "sha256:...", "majorVersion": 16 },
      { "name": "keycloak-db", "specDigest": "sha256:...", "majorVersion": 16 }
    ]
  },
  "openfga": {
    "stores": [ { "name": "orgs", "modelDigest": "sha256:..." } ]
  },
  "operatorVersion": "0.1.0-poc"
}
```

The digests are content hashes of the source-cluster CR specs at capture time. `Strict` validation requires every digest to match the target's current digest; the POC stops there.

#### Schema storage

The JSON Schema for `topology.json` is shipped with the backup-operator and projected into a `ConfigMap` (`backup-topology-schemas`) in the operator's namespace at startup, keyed by `schemaVersion`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backup-topology-schemas
  namespace: platform-mesh-backup-operator
data:
  v1alpha1.json: |
    { "$schema": "https://json-schema.org/draft/2020-12/schema", ... }
```

The operator reads schemas from this ConfigMap when writing or validating `topology.json`. Schema upgrades flow through operator releases — a new operator version reconciles new keys into the ConfigMap and continues to honour older versions for backups taken by earlier operator versions. The ConfigMap is also a useful debugging surface for operators (`kubectl get cm backup-topology-schemas -o yaml`).

## Component Orchestration

The operator owns no backup data. It drives upstream CRs and watches them to completion. POC behaviour for each:

### etcd (etcd-druid) — multi-shard

- **Backup capture** — the operator enumerates every `Etcd` CR in the platform namespace (one per kcp shard) and triggers an on-demand full snapshot on each in parallel. etcd-druid's etcdbr sidecar exposes a snapshot trigger via the [`Etcd` resource's snapshot operation annotation](https://github.com/gardener/etcd-druid). Continuous delta snapshots are already running independently per shard at the configured `delta-snapshot-period`; the operator does not interrupt them. The operator records the resulting per-shard full-snapshot keys and writes them to `index.json` keyed by shard name.
- **Restore** — for each shard recorded in `topology.json`, the operator recreates the `Etcd` CR with a restore directive pointing at that shard's recorded snapshot key. Restore proceeds shard-by-shard; a failure on any shard fails the whole `PlatformRestore` (no partial restore in the POC). Target topology must include the same set of shard names — enforced by topology validation.
- **Capture ordering** — shards are captured concurrently. Cross-shard logical consistency is not synchronised, consistent with the operator's no-coordinated-capture stance; the restore reconciler is the consistency mechanism if any cross-shard drift surfaces.

### CNPG `Cluster`s

- **Backup capture** — for each `Cluster` in the platform namespace, the operator creates a [CNPG `Backup`](https://cloudnative-pg.io/documentation/current/backup/) CR (on-demand base backup). CNPG continues continuous WAL archiving in parallel; the operator records the `Backup` CR name and the WAL position at capture time.
- **Restore** — POC restore creates a new `Cluster` with `spec.bootstrap.recovery.source` pointing at the existing `barmanObjectStore`, optionally with `recoveryTarget.targetTime`. CNPG handles the rest (validate, fetch base backup, replay WAL).
- **POC simplification** — fixed list of `Cluster`s (`openfga-db`, `keycloak-db`). Per-cluster overrides deferred.

### Velero — lifecycle plus orchestration

Velero is treated as an internal dependency of the backup-operator, not a separately-installed Platform Mesh component.

- **Lifecycle (owned by backup-operator)** — at startup the operator ensures Velero CRDs, server `Deployment`, node-agent `DaemonSet`, and a default `BackupStorageLocation` derived from `PlatformBackup.spec.storage` exist in the platform namespace. Versioning is pinned to whatever Velero release the backup-operator vendors; upgrades flow through backup-operator releases, not user Helm values.
- **Backup capture** — the operator creates a Velero `Backup` CR with a hard-coded `includedResources` list covering Platform Mesh CRDs that are not in etcd-druid's coverage. POC scope: `Store`, `AuthorizationModel`, `Account` and a few well-known platform CRDs. Velero handles the actual snapshot.
- **Restore** — creates a Velero `Restore` CR pointing at the captured backup. Apply order is **after** etcd and CNPG so that CRs land on a usable kcp/Postgres.
- **POC simplification** — no kcp-aware Velero plugin yet; that work is its own follow-up RFC. The POC backs up what Velero understands natively and lets the security-operator re-derive what it can.

### Cross-tier repair (the "reconciler")

After Velero restore completes, a final subroutine runs:

1. **Wait for the security-operator** to come up and re-reconcile `Store` / `AuthorizationModel` CRs. The operator just waits on those CRs' status — no direct OpenFGA calls.
2. **OpenFGA orphan tuple sweep** — list tuples whose `originClusterID/name` no longer resolves to a kcp resource on the target cluster. POC deletes them. Behaviour is `dry-run` by default — a flag on `PlatformRestore` opts into deletion.
3. **Report** — write outcome counts to `PlatformRestore.status.repair`.

There is intentionally **no** logic for Keycloak sessions, in-flight controller state, or anything else outside this surface. Sessions die on restore — accepted.

## Reconciliation Flow

### Backup

```
PlatformBackup created
  ↓
validate spec, resolve storage Secret
  ↓
topology_capture: read Etcd, CNPG Cluster, Store/AuthorizationModel CRs → build topology.json
  ↓
parallel:
  etcd_capture:   annotate each Etcd CR with snapshot trigger, watch until snapshot key reported
  cnpg_capture:   create CNPG Backup CR per Cluster, watch until Succeeded
  velero_capture: create Velero Backup CR, watch until Completed
  ↓
manifest_writer: PUT topology.json + index.json to S3 at <bucket>/<backupID>/
  ↓
status.phase = Succeeded
```

### Restore

```
PlatformRestore created
  ↓
fetch <bucket>/<backupID>/topology.json
  ↓
topology_validator: compute current target digests, compare → status.conditions[TopologyValid]
  ↓  (refuse if mismatch and topologyValidation=Strict)
sequential:
  etcd_restore:   recreate Etcd CR pointing at recorded snapshot key
  cnpg_restore:   create new Cluster with bootstrap.recovery referencing barmanObjectStore
  velero_restore: create Velero Restore CR
  ↓
repair: wait for security-operator readiness, run orphan-tuple sweep
  ↓
status.phase = Succeeded
```

Each subroutine emits a `Condition` and updates `status.phase`. Failures stop the chain and leave the resource in a `Failed` phase with a structured reason.

## Test Plan

POC success criteria — all running on the [helm-charts `local-setup`](https://github.com/platform-mesh/helm-charts) Kind environment:

1. **Backup**: a `PlatformBackup` on a freshly-installed Platform Mesh produces a complete bucket layout with `topology.json`, etcd full + delta snapshots, CNPG base backup + WAL, Velero artefacts. Manual inspection confirms every artefact is reachable.
2. **Round-trip restore**: tear down the cluster, recreate it from the same Helm install, apply a `PlatformRestore` referencing the prior backup. Confirm:
   - kcp workspaces, accounts, and stores return identically.
   - OpenFGA tuples (including iam-service-authored ones) are intact.
   - Keycloak realms exist and users can authenticate (sessions are expected to be lost).
3. **Topology mismatch**: change one CNPG `Cluster`'s name on the target before restore; confirm `PlatformRestore` refuses with a `TopologyValid=False` condition.
4. **No-coordination consistency**: run continuous traffic against the platform during backup capture; restore; run the repair subroutine; quantify orphan tuple count and any other observed divergence. **This is the gating result for whether ADR 008's "no admission freeze" assumption holds.**
5. **Multi-shard restore**: take a backup of a ≥2-shard platform, tear down, restore. Confirm workspaces partitioned across shards return to their correct shards, APIBindings still resolve, and cross-shard references survive. Failure on any single shard during restore must fail the `PlatformRestore` cleanly (no partial state).

## Implementation Tickets

Twelve tickets, organised into four tracks. After Track 1 lands (foundation), Track 2 runs **fully parallel** (seven independent component drivers). Track 3 stitches them together and depends on most of Track 2. Track 4 validates.

```
                ┌───── T3  Velero lifecycle ─────────────┐
                ├───── T4  etcd-druid drivers ───────────┤
T1 ──┐          ├───── T5  CNPG drivers ─────────────────┤
     ├─►  ──────┼───── T6  Velero CR drivers ────────────┼───►  T10 PlatformBackup ctrl ──┐
T2 ──┘          ├───── T7  Topology capture + S3 writer ─┤    T11 PlatformRestore ctrl ──┼─► T12 E2E
                ├───── T8  Topology validator ───────────┤                               │
                └───── T9  Cross-tier repair ────────────┘                               │
                                                                                         ▼
                                                                                  Track 4 validation
```

### Track 1 — Foundation (parallel, must land first)

- **T1: Bootstrap `platform-mesh/backup-operator` + API types.** New repo. Go module, controller-runtime + [`platform-mesh/subroutines`](https://github.com/platform-mesh/subroutines) scaffolding, CI, Dockerfile, minimal Helm chart, CODEOWNERS. `PlatformBackup` / `PlatformRestore` Go types under `api/v1alpha1/`, CRD generation via `controller-gen`, sample manifests under `config/samples/`. No reconcile logic. **Size: M.**
- **T2: `topology.json` schema v1alpha1 + ConfigMap projection + Go validator library.** JSON Schema document(s) embedded in the operator binary, projected into `backup-topology-schemas` ConfigMap at startup, Go library that reads schemas from the ConfigMap and validates/marshals topology documents. Unit tests on schema round-trip. **Size: S.**

### Track 2 — Component drivers (fully parallel after Track 1)

Each ticket is independently shippable, owns its own envtest setup with the relevant upstream CRDs installed, and exposes a `Subroutine` interface from `platform-mesh/subroutines`.

- **T3: Velero lifecycle subsystem.** `internal/velero/` reconciler: ensure Velero CRDs, server `Deployment`, node-agent `DaemonSet`, and a `BackupStorageLocation` derived from `PlatformBackup.spec.storage`. Pin Velero version via Go vendor. Upgrade tests against two Velero versions. **Depends on: T1. Size: M.**
- **T4: etcd-druid capture + restore drivers.** Discover all `Etcd` CRs in the platform namespace, trigger on-demand full snapshot via the etcdbr annotation, watch until snapshot key reported. Restore-side: recreate `Etcd` CR with restore directive pointing at a recorded key. Per-shard fan-out logic. **Depends on: T1. Size: M.**
- **T5: CNPG capture + restore drivers.** Create on-demand `Backup` CR per CNPG `Cluster`, watch until `Succeeded`. Restore-side: create new `Cluster` with `spec.bootstrap.recovery` referencing the source `barmanObjectStore`. **Depends on: T1. Size: M.**
- **T6: Velero Backup + Restore CR drivers.** Build the `includedResources` list (hard-coded POC inventory), create Velero `Backup` / `Restore` CRs, watch to terminal state. **Depends on: T1 (also T3 for the BSL contract). Size: S.**
- **T7: Topology capture subroutine + S3 manifest writer.** Read `Etcd`, CNPG `Cluster`, security-operator `Store`/`AuthorizationModel` CRs, compute digests, assemble `topology.json` and `index.json`, PUT to S3 via `minio-go`. **Depends on: T1, T2. Size: M.**
- **T8: Topology validator subroutine.** Fetch `topology.json` from S3, compute target-cluster digests, compare under `Strict`, surface mismatches as a structured condition. **Depends on: T1, T2. Size: S.**
- **T9: Cross-tier repair subroutine.** Wait for security-operator readiness, list OpenFGA tuples whose `originClusterID/name` no longer resolves on the target cluster, sweep (dry-run by default; opt-in deletion via `PlatformRestore.spec.repair.delete`). Report counts to status. **Depends on: T1. Size: M.**

### Track 3 — Controller wiring (parallel pair after Track 2 components land)

- **T10: `PlatformBackup` controller wiring.** Chain T7 → (T4 + T5 + T6 parallel) → T7 manifest write, status aggregation, conditions per subroutine, failure semantics. **Depends on: T3, T4, T5, T6, T7. Size: M.**
- **T11: `PlatformRestore` controller wiring.** Sequential chain: T8 (validate) → T4 (etcd) → T5 (cnpg) → T6 (velero) → T9 (repair). All-or-nothing failure semantics. **Depends on: T4, T5, T6, T8, T9. Size: M.**

### Track 4 — Validation (last)

- **T12: End-to-end test suite on Kind quick-start.** Covers all five scenarios in the Test Plan, including the gating no-coordination consistency run and multi-shard round-trip. Lives in `test/e2e/` with a `task e2e` entrypoint. **Depends on: T10, T11. Size: L.**

### Suggested initial assignment

Three people can run the POC in parallel after T1+T2 (≈1–2 days of foundation) lands:

- **Person A** picks up T3 + T6 (Velero, both lifecycle and CR drivers — keeps Velero knowledge in one head) → T10.
- **Person B** picks up T4 + T5 (etcd-druid and CNPG drivers — both are "drive upstream CRs and watch") → T11.
- **Person C** picks up T2 → T7 + T8 + T9 (topology and repair — the genuinely novel surface) → T12.

This gets seven Track-2 tickets in flight at once with minimal cross-blocking.

## Open Questions

- What is the minimum useful set of CRDs the Velero subroutine should include in the POC? Driven by what the security-operator cannot re-derive — needs a concrete inventory.
- How does the POC handle the case where the target cluster's `Etcd` CR shape doesn't exactly match the source (e.g., resource limits drift in Helm values between source and target)? Strict topology may reject too aggressively; needs a defined tolerance.
- What is the unit-test seam for the upstream CR drivers (etcd-druid, CNPG, Velero)? Each has a CRD client; mocking those vs. running envtest with the actual CRDs installed is a tooling choice.

## Follow-up RFCs

These are the items the POC will likely surface as needing their own design:

1. **kcp-aware Velero plugin** — `BackupItemAction` / `RestoreItemAction` v2 for logical-cluster-level filtering and selective workspace restore.
2. **Scheduling, retention, GC** — `PlatformBackupSchedule` CRD or controller-managed cron.
3. **Partial restore / per-shard recovery** — if the POC's all-or-nothing multi-shard restore proves too coarse in operations.
4. **Coordinated capture** — only if the POC's "no coordination" test produces an unacceptable divergence; then a cross-platform admission freeze and shared logical marker are designed.
5. **Cluster-ID remapping** — only if the assumption that logical-cluster IDs are preserved on restore turns out to be wrong in practice.
6. **Encryption / KMS surfaces** — a single key reference on `PlatformBackup` that wires through to CNPG and Velero.
