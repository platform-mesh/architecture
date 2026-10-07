# ADR 015: Platform Mesh Installation

| Status          | Proposed          |
|-----------------|-------------------|
| Date            | 2026-10-02        |
| Decision-makers | Platform Mesh TSC |

## Context and Problem Statement

Until recently, Platform Mesh provided only a local installation path (local-setup) with 2 flavors — a PINNED installation and PRERELEASE (local build mode). A production setup was added later. Both turned out to be problematic for new users and adopters, leading to confusion and a sub-par user experience. The following problems were identified:

1. **Complex and error-prone local-setup.** The local-setup is a set of bash scripts, kustomize overlays and YAML with multiple dependencies and 2 distinct modes (PINNED and PRERELEASE), where PRERELEASE is the default. Because of this inherent complexity, when the install fails it is hard for the user to understand what happened and how to fix it. A script-based imperative installation is also not a preferred practice and doesn't translate to other environments like PRODUCTION, air-gapped or GitOps-based installations.
2. **Huge resource requirements.** Platform Mesh requires 12 GB of memory and 6 CPUs allocated to Docker. This exceeds the defaults of most workstations and laptops, and the install fails if the user doesn't manually increase the limits. The resource footprint is unfit for most developer machines.
3. **Poor documentation.** The 3 different installation paths (PRODUCTION, local PINNED, and local PRERELEASE) were confusing and badly communicated. Users often started the wrong mode and tried to make it work on a topology where it was not intended. Issues on the website with wrong versions, the requirement to check out Git repositories before install, and duplication of docs across the website and multiple repositories all contributed to the bad user experience.
4. **No declarative GitOps installation.** A gap between user expectations and Platform Mesh documentation — this was simply missing.
5. **Oversized software stack.** The prerequisites list was large — KRO, OCM, Flux, openssl, mkcert, specific versions of jq, yq, and base64. If anything was missing or at the wrong version, installation often failed. The purpose of each tool was not clearly communicated, so users made changes in the wrong place (editing Flux HelmReleases instead of the PM profile), which made things worse. The larger and more complex the list of technologies, the heavier the mental burden and the worse the experience.

The goal is a Helm-first installation that uses only standard Kubernetes primitives, ships production-safe chart defaults, and lets GitOps controllers manage the lifecycle.

## Decision Drivers

- **Reduced resource footprint**: the installation should reduce resource usage so Platform Mesh can run in constrained environments. Locally, the current requirement of 12 GB memory and 6 CPUs exceeds the defaults of most workstations. In production, resource requests are dominated by the observability stack — measured at over 110 GB memory requests on a GKE Autopilot cluster, most of it unused at low workspace counts. Two levers address this: disabling OpenTelemetry by default (near-term), and a CORE-only installation that disables non-core functionality (gated on the modularization RFC).
- **A single installation path**: provide only one installation path — one document and one set of parts to install — tailored for local, production, and air-gapped targets via configuration only.
- **No image building during local install**: use published components for the install.
- **Declarative-first**: every part of the installation must be expressible as a Kubernetes resource so GitOps controllers can manage and reconcile it.
- **Separation of production and development values**: the published chart ships production-safe defaults; environment-specific overrides live in files outside the chart.
- **Minimal prerequisites**: external dependencies are moved out of the Platform Mesh installation and listed clearly as requirements — Flux, the OCM Kubernetes controller, cert-manager, Gateway API, an ingress/Gateway controller, CloudNativePG, and the Keycloak Operator. Remove as many internal dependencies as possible, starting with KRO.
- **GatewayClass agnosticism**: the chart should not hard-code Traefik; it should accept a configurable GatewayClass name so users can bring their own ingress controller.
- **Global configuration via `PlatformMesh` resource**: settings that appear in multiple Helm charts (e.g. `userIdClaim`, IdP configuration) should be set once in the `PlatformMesh` resource and propagated by the operator, avoiding partial-configuration drift.
- **Improved user documentation**: the website, repositories, and readme's should all be consistent and straightforward. The user should be able to follow obvious installation instructions without alternatives, duplicates, or unclear installation modes.
- **No git checkout**: users should not have to git clone or checkout resources in order to install Platform Mesh.

## Considered Options

### A. Improved status quo — bootstrap scripts, production guide

Iterate on the existing shell scripts and production guide. Teams that need GitOps compatibility must create it by themselves.

- Good, because almost all scenarios are covered: local pinned, local prerelease, production.
- Good, because the existing platform-mesh-operator already does 100% of the installation declaratively in a GitOps-friendly way.
- Good, because it is fully automated.
- Bad, because of heavy installation scope — PM installs external dependencies which are not core PM.
- Bad, because users run into many issues and have a bad experience.
- Bad, because the correct path is hard to identify and follow.
- Bad, because of unrealistic local requirements for resources and dependency tooling.
- Bad, because host-tool differences (base64, jq) cause silent failures on developer machines.
- Bad, because when an error occurs, debugging is hard for a newcomer without experience with the whole stack.

### B. New declarative installation based on the platform-mesh-operator

Provide a single installation path — one document and one set of parts to install — which is declarative and uses the `platform-mesh-operator` to fully bootstrap a working environment. It targets different topologies (local, production, air-gapped) via configuration only. External dependencies are moved out of the PM installation path and listed clearly as prerequisites. Chart defaults target production; local needs configuration overrides. Disabling the observability stack by default reduces the footprint in the near term; once the modularization RFC is implemented, a stripped-down CORE-only PM installation can be enabled, turning off non-core functionality to reduce the resource footprint significantly.

- Good, because a `helm install` command produces a complete, reconciling installation without scripts.
- Good, because the rendered Kubernetes resources are managed by platform-mesh-operator.
- Good, because we keep OCM benefits like the chart can be pinned to an exact Platform Mesh OCM component version, making installations reproducible and air-gap-capable.
- Good, because it removes confusion regarding the appropriate installation path.
- Good, because it reuses the `platform-mesh-operator` to bootstrap the environment, which keeps the existing structure in place and features like operator templating.
- Good, because it enables the removal of many dependencies like KRO, jq, yq, mkcert, openssl.
- Good, because it doesn't force users to 'git checkout/clone' before install.
- Bad, because the user might still get confused by unfamiliar technologies — OCM, FluxCD.
- Bad, because the user still needs to understand how the installation process works and where the user-facing configuration toggles live.
- Bad, because useful features like local-setup and PRERELEASE modes are removed.
- Bad, because it is less automated compared to the script-heavy installation.

### C. Flat Flux `HelmRelease` manifests applied directly

Ship Platform Mesh as a flat set of per-component Flux `HelmRelease` and `OCIRepository` manifests that are applied directly with `kubectl apply`, deliberately bypassing the convenience layers — no OCM resolution at runtime, no KRO, and no Platform Mesh Operator templating. A small Go installer generates the per-cluster manifests (prompting for base domain, front-proxy and Traefik service IPs) and can mirror the Platform Mesh OCM component — all sub-components, charts and images — into another OCI registry for air-gapped environments. Flux reconciles the ~24 `HelmReleases` into pods. This is the approach of the [`platform-mesh-from-scratch`](https://github.com/xrstf/platform-mesh-from-scratch) spike (validated against Platform Mesh 0.5.2 on a Gardener shoot).

- Good, because it is the absolute minimum needed to run Platform Mesh — every component is a plain `HelmRelease`, so there is nothing between the user and Flux to reason about.
- Good, because the Go installer handles OCM mirroring and generates matching Helm values, making air-gapped installs reproducible without hidden steps.
- Good, because the installation is fully declarative and GitOps-native once the manifests are generated.
- Bad, because stripping OCM/KRO/PMO templating means the per-cluster values (domain, service IPs, host aliases) must be substituted into every manifest, and the generator becomes a parallel source of truth to the published chart.
- Bad, because there is no composition engine ordering the `HelmReleases`; the rollout relies on Flux retries and raised concurrency, and several steps (kcp webhook Secret, `platform-mesh-profile.yaml`) are still manual in the spike.
- Bad, because it discards the Platform Mesh Operator's bootstrapping role rather than fixing it — the operator already performs 100% of the declarative install, so this duplicates effort instead of building on it.
- Bad, because removing the OCM controller loses features like proof of origin, packaging, and versioning.

## Decision Outcome

Chosen option: **B — New declarative installation based on the platform-mesh-operator**, because it satisfies the declarative and GitOps-compatibility requirements while keeping the initial installation experience simple, and because it builds on the Platform Mesh Operator's existing bootstrapping role rather than discarding it. Option C proves that a bare-manifest install is possible, but strips the convenience layers (OCM at runtime, PMO templating) that make the install reproducible and air-gap-friendly without a parallel generator; its value is as a reference for the minimal runtime dependencies, not as the shipped install path.

Concretely:

1. **`installation` stanza in the Platform Mesh Operator chart** (opt-in, default `false`). When enabled, the chart renders:
   - An OCM `Repository` pointing to `ghcr.io/platform-mesh`.
   - An OCM `Component` referencing the pinned `installation.version` and verifying the component signature.
   - A `PlatformMesh` resource wired to the OCM component.
   - The OCM signing public certificate (Secret), unless `installation.installSigningCertificate=false`.
   - The chart fails to render if `installation.baseDomain` is not set.

2. **One procedure, selected by profile.** Local and production use the same installation procedure; the only difference is configuration. `installation.profile` selects the topology and defaults to `production`, which assumes the infrastructure prerequisites already exist. A `kind` profile carries the local-only values (localhost domains, `hostAliases`, self-signed certificates). The production profile must not contain any localhost or Kind-specific values.

3. **Infrastructure prerequisites are scoped outside the PM installer.** The infrastructure dependencies — Flux, the OCM Kubernetes controller, cert-manager, Gateway API, an ingress/Gateway controller, CloudNativePG, and the Keycloak Operator — are not installed by Platform Mesh. They are listed as requirements that the user satisfies with their own tooling before installing PM; preparing them does not count as a PM installation step. An optional companion chart bundles these prerequisites for local and demo use, so that adopters who do not have an existing stack can get started quickly; production operators are expected to bring and manage their own.

4. **Certificate provisioning.** Two modes are supported. In pre-provisioned mode the user supplies the `domain-certificate` and `domain-certificate-ca` Secrets directly (typical for production with externally managed PKI). In installer-emitted mode the chart emits `Certificate` resources for cert-manager to fulfil; a self-signed issuer is available for local use. The mode is selected by chart values; both modes are first-class.

5. **KRO is removed from the installation flow.** The Platform Mesh Operator renders `HelmRelease` resources directly from OCM components. KRO's template role is absorbed into the operator. The effective chain becomes: `PMO → OCM → HelmReleases → Flux → Pods`.

6. **Global configuration via `PlatformMesh` resource.** The operator already propagates configuration to the Platform Mesh services. This improves its templating capabilities so that configuration duplication is fully removed — settings that currently appear in multiple Helm charts (`userIdClaim`, IdP endpoint, SMTP) are set once on the `PlatformMesh` resource. It also adds flexibility by allowing user-defined templates rather than only the fixed ones built into the operator.

7. **OpenTelemetry is disabled by default.** The observability stack (OTEL collector, Tempo, Grafana) ships disabled by default and is opt-in via the `PlatformMesh` resource. This is the primary near-term lever for reducing the resource footprint on both developer machines and production clusters, ahead of the modularization RFC.

8. **Upgradability.** The declarative installation path naturally supports `helm upgrade` for operator chart updates. Detailed upgrade sequencing — CRD upgrades, HelmRelease ordering, data migrations — is deferred to a follow-up ADR; upgradability is a stated goal and is not addressed by scripts. The operator and the PlatformMesh OCM component should be upgraded separately due to fragmented install.

9. **Documentation is a single install guide**, replacing the previous script-based instructions and the dispersed per-repository docs. It documents the one supported path; alternative modes and duplicated docs are removed.

10. **Configurable ingress.** The chart must not hard-code Traefik. Decoupling the installation from a specific GatewayClass so users can bring their own ingress controller is a follow-up and is not required for the first iteration.

### Scope

The scope for this ADR is:
- improve Installation UX
- fully declarative and GitOps friendly installation path
- OpenTelemetry disabled by default (resource footprint reduction, near-term)

Follow-ups to this ADR:
- installation of PlatformMesh via single command
- full resource footprint reduction (requires modularization RFC)
- per-component toggleability / CORE-only install (requires modularization RFC)
- upgrade sequencing and migration (follow-up ADR)
- refactor operator configuration — rework the profile and `PlatformMesh` CR structure (possibly merging them), templating, and validation (follow-up ADR)
- release process improvements — the release and versioning process is partially connected to installation UX (what the user installs and how versions are pinned) and needs dedicated improvement (follow-up ADR)
- CI/CD and release engineering

### Consequences

- Good, because once the prerequisites are in place, a single `helm install` of the operator chart produces a complete, reconciling Platform Mesh installation with no scripts.
- Good, because the same chart, with different profiles, is used for local development, demo, and production; no separate code paths exist.
- Good, because the OCM component version is the only source of truth for the whole PlatformMesh service landscape.
- Good, because the declarative install path enables `helm upgrade` for day-2 operations; detailed upgrade sequencing is deferred to a follow-up ADR.
- Good, because removing KRO eliminates one reconciliation layer and simplifies the mental model to `PMO → OCM → HelmReleases → Flux → Pods`.
- Good, because global configuration in the `PlatformMesh` resource prevents the partial-configuration drift that has caused production incidents.
- Good, because disabling OpenTelemetry by default eliminates the largest single component of resource requests without requiring the modularization RFC.
- Bad, because the significant resource reduction beyond OTEL depends on the modularization RFC; per-component toggleability (flattening Helm charts) is a hard dependency of the CORE-only install path and is not defined here.
- Bad, because the production profile must be validated on an internet-facing cluster before being recommended beyond Kind; this is tracked as a follow-up task.
- Bad, because the infrastructure prerequisites must be prepared by the user and are not part of the installer.
- Bad, because the technology stack is still complex and hard for all users to understand.
- Bad, because the `platform-mesh-operator` isn't resolved from the OCM component which results in fragmented installation. Relevant especially for airgapped environments.

## Open Questions

- **Resource footprint / CORE-only install**: the significant resource reduction beyond disabling OTEL depends on the modularization RFC. Which functionality is core versus optional, and how it is toggled, is resolved there, not in this ADR.

## References

- [PR #2: PlatformMesh install (akafazov/platform-mesh-helm-charts)](https://github.com/akafazov/platform-mesh-helm-charts/pull/2) — POC implementation that this ADR documents.
- [`platform-mesh-from-scratch`](https://github.com/xrstf/platform-mesh-from-scratch) — the bare-manifest "from scratch" spike documented as option C.
- [Backlog issue #391](https://github.com/platform-mesh/backlog/issues/391) — installation UX refinement tracking issue.
