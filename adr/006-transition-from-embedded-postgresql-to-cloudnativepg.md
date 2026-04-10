# ADR 006: Transition from Embedded Bitnami PostgreSQL to a Kubernetes-Native PostgreSQL Operator

| Status  | Draft      |
|---------|------------|
| Date    | 2026-04-10 |

## Context and Problem Statement

Platform Mesh deploys PostgreSQL for Keycloak and OpenFGA using **Bitnami PostgreSQL subcharts** — two independent single-instance StatefulSets with no replication, failover, read scaling, or unified management. Additionally, Bitnami has [ended free-tier security updates](https://github.com/bitnami/charts/issues/35164) (August 2025), making continued use untenable without paid licensing or forked images.

As Platform Mesh matures beyond alpha, a more reliable database layer is needed — particularly for production environments.

## Decision Drivers

* Automatic failover without manual intervention.
* Unified PostgreSQL management via Kubernetes-native APIs.
* Read replica support for services that benefit from it (OpenFGA).
* The transition must be **optional** — operators can use their own database infrastructure.
* CNCF or Linux Foundation affiliation preferred (not required) for vendor-neutral governance.
* Permissive license compatible with Apache 2.0 (Platform Mesh's license).
* Preference for European open-source projects where possible (digital sovereignty).
* Cross-cluster replication support for future multi-site deployments.

## Considered Options

1. **Keep embedded Bitnami PostgreSQL subcharts** — status quo.
2. **CloudNativePG (CNPG)** — CNCF Sandbox; Kubernetes-native PostgreSQL operator (Italy).
3. **Zalando Postgres Operator** — Patroni-based HA operator (Germany).
4. **Crunchy PGO** — Enterprise-grade Patroni-based operator (USA, acquired by Snowflake 2025).
5. **StackGres** — PostgreSQL operator with web UI and sharding (Spain).

## Decision Outcome

Chosen option: **CloudNativePG (CNPG)**, because it is the only CNCF-affiliated PostgreSQL operator, originates from a European company (EDB/2ndQuadrant, Italy), uses Apache 2.0 licensing, and provides automatic failover, declarative database/role CRDs, native read replica services, and cross-cluster replica clusters. The comparison matrix below details how the alternatives compare across all decision drivers.

### Architecture

CNPG `Cluster` resources provision PostgreSQL in the `platform-mesh-system` namespace. Whether to use a single shared cluster or separate clusters per service is a deployment-time decision. Databases and roles are managed declaratively via `Database` CRDs and `managed.roles`. CNPG automatically creates services per cluster: `-rw` (primary), `-ro` (replicas), `-r` (any instance).

| Service | Read Replica Support | Connection Strategy |
|---------|---------------------|-------------------|
| **Keycloak** | No — all operations require write access | Primary (`-rw`) only |
| **OpenFGA** | Yes — via `OPENFGA_DATASTORE_SECONDARY_URI` | Primary (`-rw`) + replica (`-ro`) |

For production, a 2+ instance cluster enables failover and read scaling. For local development, a single instance suffices. Cross-cluster deployments can use CNPG Replica Clusters with streaming replication or WAL archiving, with GitOps-driven switchover for failover.

### Consequences

* Good, because a managed cluster replaces unmanaged StatefulSets with automatic failover.
* Good, because databases, roles, and credentials are managed declaratively via CRDs.
* Good, because OpenFGA can leverage read replicas for improved throughput.
* Good, because CNPG is CNCF Sandbox with vendor-neutral governance and Apache 2.0 licensing.
* Good, because cross-cluster replica clusters provide a path to multi-site deployments.
* Bad, because CNPG introduces an additional operator dependency.

## Operator Comparison Matrix

| Criterion | CloudNativePG | Zalando | Crunchy PGO | StackGres |
|-----------|:---:|:---:|:---:|:---:|
| **Origin** | Italy (EDB) | Germany (Zalando SE) | USA (Crunchy Data / Snowflake) | Spain (OnGres) |
| **CNCF/LF Affiliation** | CNCF Sandbox (entry tier) | None | None | None |
| **License** | Apache 2.0 | MIT | Apache 2.0 | AGPLv3 |
| **HA Mechanism** | Native streaming replication | Patroni | Patroni | Patroni |
| **Database-Level CRD** | Yes (`Database`) | No (cluster-level only) | No (cluster-level only) | No (cluster-level only) |
| **Read/Write Splitting** | Native `-rw`/`-ro` services | Via replica pooler | Via routing | Via `-replicas` service |
| **Cross-Cluster** | Replica Clusters + switchover | Standby clusters (S3 WAL) | Active-Standby (pgBackRest) | Not supported |
| **Backup/PITR** | Barman Cloud (S3/GCS) | WAL-E/S3 | pgBackRest | Kubernetes-native |
| **Official Helm Chart** | Yes (OCI) | Yes | Yes | Yes |
