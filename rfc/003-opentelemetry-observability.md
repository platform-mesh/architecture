# RFC 003: OpenTelemetry Observability

| Status  | Draft                                                                              |
|---------|------------------------------------------------------------------------------------|
| Author  | @nexus49                                                                           |
| Created | 2026-03-27                                                                         |
| Updated | 2026-03-27                                                                         |
| RFC PR  | TBD                                                                                |

## Motivation

Platform-mesh operators and services already produce telemetry — metrics via controller-runtime's Prometheus registry (`/metrics`), traces via OpenTelemetry spans (RFC 002), and structured logs via `logr`. However, there is no unified collection layer. Each installation must manually configure scraping, trace ingestion, and log aggregation. This creates friction for both local development and production deployments.

This RFC introduces a built-in OpenTelemetry Collector pipeline that ships with every platform-mesh installation. The goal is zero-config observability: install platform-mesh, get telemetry collection out of the box. Backends (where data goes) are intentionally out of scope — this RFC defines only how telemetry is collected and forwarded.

## Scope

### In Scope

- Telemetry collection for all three signals: metrics, traces, and logs
- OTel Collector deployment topology for local development and production
- Collector pipeline configuration (receivers, processors, exporters)
- OTel Operator integration for lifecycle management
- Target Allocator for production-scale metric scraping

### Out of Scope

- Observability backends (Prometheus, Jaeger, Tempo, Loki, Grafana, etc.)
- Dashboards, alerting rules, or notification routing
- Application-level instrumentation — RFC 002 already defines subroutine-level tracing and metrics
- Backend storage, retention, or high-availability configuration

## Design Principles

1. **Gateway-first** — A centralized OTel Collector gateway handles metrics scraping, trace ingestion, log processing, and export. A lightweight DaemonSet log agent supplements the gateway for collecting logs from all components via the node filesystem. No sidecars.

2. **Pull for metrics, push for traces, tail for logs** — Metrics are scraped from existing `/metrics` endpoints (no code changes). Traces are pushed via OTLP by the applications themselves. Logs are tailed from the node filesystem by the DaemonSet agent and forwarded to the gateway — uniformly for all components, whether platform-mesh operators or third-party tools like KCP, cert-manager, or Keycloak.

3. **One tool, one protocol** — Both the gateway and the DaemonSet agent are OTel Collector instances. One config language, one upgrade cycle, one set of docs. The DaemonSet agent replaces what would otherwise be a separate tool like Fluentbit or Fluentd.

4. **One config, scaling knobs** — The collector configuration is identical across environments. The only difference between local and production is the replica count.

5. **Operator-managed** — The OpenTelemetry Operator manages both collector instances via `OpenTelemetryCollector` custom resources. No manual Deployment/ConfigMap management.

6. **Collection, not consumption** — This RFC defines the collection pipeline. What consumes the collected telemetry (which backend, how it's stored, how it's queried) is a deployment-time decision, not an architectural one.

---

## Architecture

### Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        All Components                           │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   Operator A  │  │   Operator B  │  │  Service C (NestJS)  │  │
│  │  /metrics     │  │  /metrics     │  │  /metrics            │  │
│  │  OTLP traces  │  │  OTLP traces  │  │  OTLP traces         │  │
│  │  stdout logs  │  │  stdout logs  │  │  stdout logs         │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                  │                      │              │
│  ┌──────────────┐  ┌──────────────┐                             │
│  │  KCP          │  │ cert-manager  │  ...third-party tools     │
│  │  stdout logs  │  │  stdout logs  │  /metrics (where avail.)  │
│  └──────┬───────┘  └──────┬───────┘                             │
│         │                  │                                     │
└─────────┼──────────────────┼─────────────────────────────────────┘
          │                  │
          ▼                  ▼
    ┌─────────────────────────────────────────────────────┐
    │              OTel Collector — Gateway                │
    │              (Deployment / StatefulSet)              │
    │                                                     │
    │  Receivers:                                         │
    │    prometheus  ← scrapes /metrics (pull)            │
    │    otlp        ← traces from apps (push)            │
    │                  + logs from DaemonSet agent (push)  │
    │                                                     │
    │  Processors:                                        │
    │    memory_limiter, resource, batch                   │
    │                                                     │
    │  Exporters:                                         │
    │    otlp/metrics    → metrics backend                │
    │    otlp/traces     → traces backend                 │
    │    otlp/logs       → logs backend                   │
    └──────────────────────────▲──────────────────────────┘
                               │
                               │ OTLP push (logs)
                               │
    ┌──────────────────────────┴──────────────────────────┐
    │          OTel Collector — DaemonSet Log Agent        │
    │          (one pod per node)                          │
    │                                                     │
    │  Receivers:                                         │
    │    filelog    ← tails /var/log/pods/ (all pods)     │
    │                                                     │
    │  Processors:                                        │
    │    k8sattributes (pod, namespace, container)         │
    │                                                     │
    │  Exporters:                                         │
    │    otlp       → forwards to gateway                 │
    └─────────────────────────────────────────────────────┘
```

### Local Development vs Production

The gateway is deployed as a StatefulSet with the Target Allocator enabled in all environments. The DaemonSet log agent is identical everywhere. The only difference between local development and production is the gateway replica count — `1` locally, `N (≥2)` in production. Everything else — deployment mode, service discovery, pipeline config — is the same.

---

## Signal Collection Strategy

### Metrics — Pull via Prometheus Receiver

Platform-mesh operators expose metrics via controller-runtime's Prometheus registry on `/metrics`. This includes both controller-runtime's built-in metrics (`controller_runtime_reconcile_total`, `controller_runtime_reconcile_time_seconds`) and the subroutine-level metrics introduced in RFC 002 (`lifecycle_subroutine_duration_seconds`, `lifecycle_subroutine_errors_total`, `lifecycle_subroutine_requeue_seconds`).

The OTel Collector's Prometheus receiver scrapes these endpoints, converts the metrics to OTLP format, and forwards them through the pipeline. **No code changes are needed in any operator or service** — the existing `/metrics` endpoints are consumed as-is.

Why pull over push for metrics:
- Operators already expose `/metrics` — reuse what exists
- Pull detects target failure immediately (failed scrape = target down)
- Scrape intervals are centrally controlled, not per-application
- No OTLP metrics SDK integration needed in Go or Node.js services

### Traces — Push via OTLP Receiver

Applications push trace spans to the collector's OTLP endpoint (gRPC on port 4317, HTTP on port 4318). RFC 002 defines the span structure — per-subroutine spans with structured attributes (`subroutine.name`, `subroutine.action`, `subroutine.outcome`, etc.). The collector receives these spans and forwards them to whichever trace backend is configured.

The OTLP endpoint is cluster-internal traffic (exposed via a ClusterIP Service) and is not authenticated by default. This is consistent with how most cluster-internal telemetry endpoints operate (e.g., Prometheus scrape targets, Kubernetes API metrics). If stronger isolation is needed, this can be addressed at the deployment level — for example, via mTLS through a service mesh, `NetworkPolicy` rules restricting which namespaces or pods can reach the OTLP ports, or the collector's built-in [authentication extensions](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/extension) (e.g., `bearertokenauth`, `oidcauth`). These are deployment-time choices and not part of the default collection pipeline.

### Logs — Tailed by DaemonSet Agent

All components — platform-mesh operators, third-party tools (KCP, cert-manager, Keycloak, PostgreSQL), and Node.js services — write structured logs to stdout. Kubernetes captures this output to files on the node filesystem at `/var/log/pods/`.

A DaemonSet OTel Collector agent runs on every node, tails these log files, enriches them with Kubernetes metadata (pod name, namespace, container name, labels), and forwards them via OTLP to the gateway for processing and export.

Why tail-based collection over OTLP push for logs:
- **Uniform collection** — one log path for all components, regardless of whether they can be instrumented with the OTel SDK
- **No code changes** — no SDK integration needed in any service, not even platform-mesh operators
- **No lost logs** — file-based tailing survives collector restarts (file offsets are checkpointed)
- **Same tool** — the DaemonSet agent is an OTel Collector instance, same config language and upgrade cycle as the gateway — replaces what would otherwise be a separate tool like Fluentbit or Fluentd

Trade-off: Tail-based collection loses automatic trace-log correlation (the `traceID`/`spanID` from the Go context is not available in stdout). If trace-log correlation is needed, operators can include `traceID` and `spanID` as structured log fields — this is a logging convention, not an SDK dependency.

---

## Collector Pipeline Overview

Two `OpenTelemetryCollector` CRs are deployed: the **gateway** and the **DaemonSet log agent**.

**Gateway** pipeline:

| Stage | Components |
|-------|-----------|
| Receivers | `prometheus` (scrapes `/metrics`), `otlp` (traces from apps, logs from DaemonSet agent) |
| Processors | `memory_limiter` (OOM protection), `resource` (cluster-level attributes), `batch` (export efficiency) |
| Exporters | `otlp/metrics`, `otlp/traces`, `otlp/logs` — backend endpoints are a deployment-time choice |

**DaemonSet log agent** pipeline:

| Stage | Components |
|-------|-----------|
| Receivers | `filelog` (tails `/var/log/pods/`) |
| Processors | `k8sattributes` (pod, namespace, container metadata), `batch` |
| Exporters | `otlp` (forwards to gateway) |

---

## Deployment Modes

Every environment deploys two `OpenTelemetryCollector` CRs: a **gateway** (StatefulSet) and a **DaemonSet log agent**. The configuration is identical across environments — the only difference is the gateway replica count (1 for local, 3 for production).

When the Target Allocator is enabled, the OTel Operator automatically:
1. Deploys a Target Allocator pod
2. Rewrites the Prometheus receiver's `scrape_configs` with `http_sd_configs` pointing to the TA
3. Distributes scrape targets evenly across collector replicas using consistent hashing
4. Discovers targets via `ServiceMonitor` and `PodMonitor` CRDs (if `prometheusCR.enabled: true`)

Each collector replica asks the TA "what should I scrape?" and receives only its assigned subset — no duplication, no single-replica bottleneck.

---

## OTel Operator Integration

The [OpenTelemetry Operator for Kubernetes](https://opentelemetry.io/docs/platforms/kubernetes/operator/) manages the full collector lifecycle:

- **Installation**: Deployed as a Helm chart dependency of the platform-mesh installation
- **Collector management**: The `OpenTelemetryCollector` CR is the single source of truth for the pipeline — the Operator translates it into Deployments/StatefulSets, Services, ConfigMaps, and ServiceAccounts
- **Target Allocator**: Built-in Operator feature, enabled via `spec.targetAllocator` — no additional installation
- **Upgrades**: Operator handles rolling updates when the CR changes

The platform-mesh Helm chart deploys:
1. The OTel Operator (as a subchart dependency)
2. Two `OpenTelemetryCollector` CRs: the gateway and the DaemonSet log agent
3. `ServiceMonitor` resources for each operator/service (for TA-based discovery in production)

---

## Downstream Integration Points

While backends are out of scope, the collection pipeline is designed to integrate with any OTLP-compatible backend. Common patterns:

| Signal | Export Protocol | Example Backends |
|--------|----------------|-----------------|
| Metrics | OTLP gRPC/HTTP, Prometheus remote write | Prometheus, Mimir, Cortex, Thanos, Datadog |
| Traces | OTLP gRPC/HTTP | Jaeger, Tempo, Zipkin, Datadog |
| Logs | OTLP gRPC/HTTP | Loki, Elasticsearch, Datadog |

**Alerting** is a backend concern. For example, in a Prometheus-based setup:
- Prometheus receives metrics via remote write (or OTLP) from the collector
- Alerting rules are defined in Prometheus (`PrometheusRule` CRDs)
- Prometheus evaluates rules and sends alerts to Alertmanager
- Alertmanager handles routing, deduplication, silencing, and notification delivery

The OTel Collector has no role in alerting — it is purely a collection and forwarding pipeline.

---

## In-Flight Buffering and Data Loss

The OTel Collector is a **forwarding pipeline, not a storage system**. It holds data in memory between receiving and exporting — there is no persistent on-disk buffer. This has different implications for each signal.

### Behavior by Signal

| Signal | Collection | During gateway downtime | During backend downtime |
|--------|-----------|------------------------|------------------------|
| Metrics | Pull (scrape) | Missed scrapes create gaps; next scrape picks up current values — no cumulative loss for gauges, but counter deltas for the missed interval are gone | `memory_limiter` applies backpressure; data is dropped if the export queue fills up |
| Traces | Push (OTLP) | Applications receive gRPC/HTTP errors; spans are dropped unless the app has its own retry/queue (OTel SDKs have a limited in-memory queue with retry, but it is bounded) | Same — `memory_limiter` and batch queue limits apply; excess spans are dropped |
| Logs | Tail (filelog) | The DaemonSet agent checkpoints file offsets (when a [storage extension](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/extension/storage/filestorage/README.md) is configured) — it resumes from the last position after restart, so **no log loss** during agent restarts. If the gateway is down, the agent's export queue fills and applies backpressure | Gateway drops logs if the export queue fills up |

### Key Mechanisms

- **[`memory_limiter` processor](https://github.com/open-telemetry/opentelemetry-collector/blob/main/processor/memorylimiterprocessor/README.md)** — Monitors the collector's memory usage and drops data when a configurable threshold is reached. This prevents OOM kills but means data loss under sustained backpressure. It is a safety valve, not a buffer.
- **[`batch` processor](https://github.com/open-telemetry/opentelemetry-collector/blob/main/processor/batchprocessor/README.md)** — Batches telemetry before export for efficiency. It holds data in memory briefly (default: 200ms or 8192 items, whichever comes first). This is not a retry buffer — if the export fails, the batch is retried a limited number of times, then dropped.
- **[Exporter `sending_queue` and `retry_on_failure`](https://github.com/open-telemetry/opentelemetry-collector/blob/main/exporter/exporterhelper/README.md)** — OTLP exporters support a bounded in-memory queue (`sending_queue`, default size: 5120 batches) with exponential-backoff retry for transient failures. Once the queue is full, new data is dropped. The queue can optionally be backed by persistent storage via a [`file_storage` extension](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/extension/storage/filestorage/README.md).
- **[`filelog` receiver checkpointing](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/filelogreceiver/README.md)** — The filelog receiver tracks file read offsets. When configured with a `storage` extension, offsets are persisted to disk, allowing the agent to resume from where it left off after a restart. Without a storage extension, offsets are in-memory only and lost on restart — a storage extension must be configured for durable checkpoint behavior.

### What This Means in Practice

For **local development**, brief interruptions are irrelevant — telemetry gaps during a collector restart are acceptable.

For **production**, the risk profile is:
- **Logs are the most resilient** — file-based tailing with checkpointing (when backed by a storage extension) survives collector restarts without loss
- **Metrics are self-healing** — missed scrapes create gaps but no drift; the next successful scrape reflects current state
- **Traces are the most fragile** — push-based with bounded in-memory queues on both the application side (OTel SDK) and the collector side; sustained gateway or backend outages cause span loss

If durability guarantees beyond in-memory buffering are needed, the exporter tier can be configured to write to a persistent queue (e.g., via a [Kafka exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/exporter/kafkaexporter/README.md) or the collector's [`file_storage` extension for persistent queues](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/extension/storage/filestorage/README.md)). This is a deployment-time decision and out of scope for this RFC.

---

## Alternatives Considered

### Sidecar Deployment Model

Each pod gets an OTel Collector sidecar that collects telemetry locally and forwards to a central backend.

**Rejected**: Sidecars add resource overhead per pod, increase scheduling complexity, and require injection configuration. The gateway model achieves the same result with a single deployment. Sidecars are warranted for very large clusters where a single gateway cannot keep up — that scale is not the current target.

### Push-Based Metrics via OTLP SDK

Instead of scraping `/metrics`, operators would use the OTel Metrics SDK to push metrics directly to the collector via OTLP.

**Rejected**: Every operator and service already exposes `/metrics` via controller-runtime's Prometheus registry. Switching to push would require integrating the OTel Metrics SDK into every service, replacing or bridging the existing Prometheus instrumentation, with no functional benefit. Pull-based scraping reuses what already exists with zero code changes.

### Log Collection via OTel Logs SDK (OTLP Push)

Instead of tailing logs from the node filesystem, operators would use the OTel Logs SDK with a `logr` bridge to push logs directly to the collector via OTLP.

**Rejected**: While this provides richer metadata (automatic trace-log correlation via context), it only works for components where the OTel SDK can be integrated. Third-party tools (KCP, cert-manager, Keycloak) would still need a separate collection mechanism. Using the DaemonSet `filelog` approach uniformly for all components is simpler — one log collection path, no SDK integration, no split behavior. Trace-log correlation can still be achieved by including `traceID`/`spanID` as structured log fields — a convention, not an SDK dependency.

### Fluentbit / Fluentd for Log Collection

Instead of an OTel Collector DaemonSet, use Fluentbit or Fluentd for log tailing and forwarding.

**Rejected**: Functionally equivalent, but introduces a second tool with a different config language, upgrade cycle, and operational model. Using the OTel Collector for both the gateway and the log agent keeps the stack homogeneous — one tool for all telemetry collection.

### Agent + Gateway Hybrid

DaemonSet agents on each node handle local collection (scraping pods on that node, collecting node-level metrics), then push to a central gateway for processing and export.

**Not rejected, but deferred**: This is a valid evolution for very large clusters where per-node agents reduce cross-network scraping traffic. It adds operational complexity (two collector tiers). The current single-gateway approach is sufficient for the expected scale. If needed, this can be introduced as a non-breaking addition — agents push OTLP to the existing gateway.

---

## Drawbacks and Limitations

1. **Prometheus receiver stability** — The Prometheus receiver is currently at **beta** stability in the OTel Collector. While it is widely used in production and included in core/contrib/k8s distributions, the API surface may change in minor releases.

2. **Unsupported Prometheus features** — The Prometheus receiver does not support `alert_config`, `remote_read`, `remote_write`, or `rule_files`. These are Prometheus server features that do not apply to a scraping receiver. None of these affect the platform-mesh use case.

3. **Gateway as single point of failure** — In single-replica mode (local dev), the collector is a SPOF for telemetry. This is acceptable for local development. In production, multiple replicas with the Target Allocator provide redundancy.

4. **Scraping at scale** — Without the Target Allocator, running multiple collector replicas causes duplicate metric scraping. The TA solves this but adds a dependency on the OTel Operator's TA component. For local development with a single replica, this is not an issue.

---

## References

- [RFC 002: Runtime Lifecycle v2](002-runtime-lifecycle-v2.md) — Subroutine-level tracing, metrics, and logging design
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) — Collector architecture and configuration
- [Prometheus Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/prometheusreceiver) — Prometheus scraping receiver for OTel Collector
- [Target Allocator](https://opentelemetry.io/docs/platforms/kubernetes/operator/target-allocator/) — OTel Operator's built-in scrape target distribution
- [OTel Operator for Kubernetes](https://opentelemetry.io/docs/platforms/kubernetes/operator/) — Operator that manages collector lifecycle
- [Gateway Deployment Pattern](https://opentelemetry.io/docs/collector/deploy/gateway/) — OTel Collector gateway deployment model
