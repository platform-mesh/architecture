---
title: 'ADR 010: Consolidate Helm Charts and OCM into a Single Repository'

---

# ADR 010: Consolidate Helm Charts and OCM into a Single Repository

| Status  | Proposed   |
|---------|------------|
| Date    | 2026-06-11 |

## Context and Problem Statement

Platform Mesh uses the [Open Component Model (OCM)](https://ocm.software) to package and distribute software components. Per [ADR 001 - Approved](001-sbom-generation-and-ocm-component-restructuring.md), each service is represented by three OCM components: a root/service component, a chart component, and an image component. Today the artifacts that define a chart release are split across two repositories:

* [`platform-mesh/helm-charts`](https://github.com/platform-mesh/helm-charts) - Helm chart definitions, chart OCI paths, `appVersion`, per-chart release workflows, and some OCM component constructors under `.ocm/`.
* [`platform-mesh/ocm`](https://github.com/platform-mesh/ocm) - additional service component constructor templates, the platform-level root aggregator (`constructor/component-constructor.yaml`), the aggregator workflow (`ocm.yaml`), signing keys, and the reusable `pipeline-service-component.yml` workflow.

The shared workflows in [`platform-mesh/.github`](https://github.com/platform-mesh/.github) wire these together: when an operator repository publishes a new image, a shared pipeline calls `pipeline-service-component.yml` in `platform-mesh/ocm` to build the service OCM component version, then opens a PR in `platform-mesh/ocm` to bump the version reference in the root aggregator.

Any structural change to a chart - renaming a resource, updating an OCI path, changing a component reference - requires a matching update to the OCM descriptor in the other repository. There is no automated link between the two changes and no CI gate that validates them together. The mismatch is only visible downstream: in the aggregator pipeline or at deployment time, potentially long after both PRs are merged.

The problem is bidirectional. Changes in `helm-charts` can silently break the descriptors and released CVs in `ocm`, and changes in `ocm` can silently break what `helm-charts` produces. In both cases the feedback loop is too long for effective review. A concrete example of this failure mode, observed in several threads in Platform Mesh's topics in Zulip:

> Someone PRs a breaking change to e.g. `charts/security-operator/`. PR checks run Helm unit tests - those pass because the chart is syntactically valid. PR merges to helm-charts main. The per-chart workflow publishes the chart to the OCI registry. The `platform-mesh/ocm` repo builds a new component descriptor referencing the broken chart. Renovate sees the newOCM CV and updates `local-setup/kustomize/components/ocm/component.yaml`. The kind-localsetup job triggers, deploys, and fails - but at this point the broken chart is already in helm-charts main and the broken OCM CV prevents merging other features and fixes, and reverting it requires coordinated action across two repositories.

Several incidents in the project's history illustrate how cross-repo coupling produces failures that are only visible downstream:

* **Chart republished under the same version, breaking signed digest chain** ([`platform-mesh/helm-charts` #1865](https://github.com/platform-mesh/helm-charts/issues/1865)): A workflow-only change to `keycloak-operator.yaml` (no chart content changed) re-triggered the chart build pipeline, which repackaged and pushed the chart under the existing version tag. The already-signed aggregator component had recorded the old digest. After the overwrite, every service referencing this chart failed OCM signature verification with a `digest mismatch` error, blocking all deployments. The fix required a coordinated sequence of patches across both repositories ([`platform-mesh/ocm` #93](https://github.com/platform-mesh/ocm/pull/93), [#95–98](https://github.com/platform-mesh/ocm/pull/95), [`helm-charts` #1866](https://github.com/platform-mesh/helm-charts/pull/1866)) before the platform was deployable again.

* **OCM pipeline for a component went silently stale**: The `marketplace-ui` OCM component version in the registry (`0.5.13`) fell far behind the actual release (`v0.9.10`), breaking `main` for contributors with no visible indication in either repository that the pipeline had disconnected.

* **Chart removed, OCM constructor orphaned** ([`platform-mesh/helm-charts` #1798](https://github.com/platform-mesh/helm-charts/pull/1798), [`platform-mesh/ocm` #122](https://github.com/platform-mesh/ocm/pull/122)): Charts for `platform-mesh-operator-components` and `platform-mesh-operator-infra-components` were deleted from `helm-charts`, but the corresponding OCM constructor entries in `platform-mesh/ocm` were not removed until a separate, later PR. During the gap, the aggregator continued referencing components whose source charts no longer existed.

A secondary consequence of the split is a feedback cycle that contributes to repeatedly exhausting the GHCR rate limit. Every component change in any source repository produces a new aggregator version in `platform-mesh/ocm`. Renovate picks up the new aggregator tag and opens a bump PR in `helm-charts` to update `local-setup/kustomize/components/ocm/component.yaml`. Each bump PR triggers a full CI run. Compounded with developers running local-setups that continuously poll the GHCR registry, the combined request volume has repeatedly pushed the project over the GHCR rate limit ([`helm-charts` #1923](https://github.com/platform-mesh/helm-charts/pull/1923)).

## Decision Drivers

* **Atomic changes**: a chart and the OCM descriptor for that chart should always be in the same PR.
* **Fast feedback**: CI must be able to validate that the OCM component builds correctly and produces functioning infrastructure before a PR is merged and an OCM CV is built, not after.
* **Single source of truth**: one repository owns all release artifacts - chart definitions, component constructors, and the root aggregator.
* **Eliminate GHCR rate limit exhaustion**: the current model requires `local-setup` and CI to continuously poll the GHCR registry, as described above. When the OCM assembly runs in the same repository as `local-setup`, the component versions are known in PRs and in local-setups and can be referenced directly - removing the need to poll the registry at all.

## Considered Options

### A. Status quo - keep the two repositories separate

No file moves. Contributors continue to coordinate chart and OCM descriptor changes manually across two repositories.

* Good, because no disruptive changes to repository structure or downstream consumers.
* Bad, because cross-repo breakage continues - changes in either repository can silently invalidate the other with no CI detection at review time.
* Bad, because the inconsistency (some constructors already in `helm-charts`) grows as the ADR 001 migration continues.

### B. Move service component descriptors to `platform-mesh/helm-charts`, keep `ocm` for the aggregator

Move `constructor/service-component.yaml` and variants from `platform-mesh/ocm` into `platform-mesh/helm-charts/.ocm/`. Retarget the shared `.github` workflows to dispatch against `helm-charts` for the service-component build step. The `platform-mesh/ocm` repo retains only the root aggregator and associated workflows.

* Good, because chart and descriptor changes land in the same PR.
* Good, because CI on `helm-charts` PRs can validate descriptors before merge.
* Bad, because two repositories still exist, meaning two separate CI contexts - a change that passes in one repo can still break the other undetected until downstream.
* Bad, because the aggregator and its templates remain in a separate repository, requiring cross-repo coordination for any change that spans the aggregator and a chart.
* Bad, because `local-setup` still needs to poll the GHCR registry to discover the latest aggregator version, so the GHCR rate limit exhaustion described in the context is not resolved.
* Bad, because the shared `.github` workflows must be updated to know when to target `helm-charts` versus `ocm` depending on the operation, adding a new decision point for contributors.

### C. Move service component descriptors into the originating source repositories, root aggregator into `helm-charts`

Each operator repository (e.g. `platform-mesh/security-operator`) owns the OCM component constructor for its own service component. The root aggregator (`constructor/component-constructor.yaml`) and its workflow move to `platform-mesh/helm-charts`. The shared `.github` workflows are updated so that when a source repository tags a release, it builds the service OCM component version directly and triggers the aggregator in `helm-charts` to incorporate the new version into the root component. `platform-mesh/ocm` is left with no remaining responsibilities and is archived.

* Good, because the OCM descriptor for a service lives in the same repository as the code and image it describes - the most natural ownership boundary.
* Good, because a breaking change to an operator's image or component structure is caught in the same PR that introduces it.
* Good, because moving the root aggregator to `helm-charts` means `local-setup` and the aggregator are in the same repository, eliminating the cross-repo polling cycle and the GHCR rate limit exhaustion.
* Bad, because the descriptor is separated from the Helm chart that defines how the component is deployed - a class of chart/descriptor mismatches (OCI path changes, resource renames in `helm-charts`) can still occur across the repository boundary.

### D. Consolidate `helm-charts` and `platform-mesh/ocm` into a single repository

Merge the two repositories. All chart definitions, service component constructors, the root aggregator, OCM workflows, and signing keys live in one place. The shared `.github` workflows are updated to dispatch against the consolidated repository for both service-component builds and root aggregator version bumps.

* Good, because every chart change and its OCM descriptor update land in a single PR with a single CI run.
* Good, because a descriptor/chart mismatch fails the PR before merge - it cannot propagate downstream.
* Good, because there is one repository for everything related to chart and OCM component releases; contributors and the shared `.github` workflows do not need to know which repository owns which layer.
* Good, because it eliminates the existing split-brain state (constructors partially already in `helm-charts`) and completes the migration.
* Bad, because consolidating two active repositories requires coordinating history, open PRs, CI secrets, Renovate configuration, and downstream workflow references across the organisation.
* Bad, because the consolidated repository will be larger; contributors who previously worked only in `helm-charts` will see OCM aggregator machinery they did not encounter before.

## Decision Outcome

Chosen option: **D - consolidate `helm-charts` and `platform-mesh/ocm` into a single repository**, because it is the only option that makes cross-repository breakage structurally impossible. Options A and B retain two repositories with separate CI contexts. Option C resolves the polling cycle but keeps service descriptors spread across many source repositories, leaving chart/descriptor mismatches (changes in `helm-charts` that affect OCI paths or resource names) detectable only at aggregator build time rather than at the PR that introduces them.

Concretely:

1. The `ocm` repository is migrated into `helm-charts`.
2. The exact directory layout is an implementation decision deferred to the migration.
3. CI on PRs to the consolidated repository runs both chart validation and OCM component build as required status checks, so a mismatch between a chart and its descriptor blocks merge rather than propagating to downstream deployments. Local tasks can also check or propagate the versions.
4. The shared workflows in [`platform-mesh/.github`](https://github.com/platform-mesh/.github) - which today orchestrate cross-repository updates (building the service OCM component version when an operator publishes a new image, opening a PR to bump the version reference in the root aggregator) - are updated to target the consolidated repository for all OCM-related dispatch steps.
5. `platform-mesh/ocm` is archived after all open work is migrated.

The exact directory layout of the merged content, and migration sequencing are implementation decisions to be resolved during the migration - this ADR records the architectural intent.

### Consequences

* Good, because a chart change and its OCM descriptor change are always in the same PR - cross-repo breakage is structurally prevented.
* Good, because CI validation of the OCM component runs on every PR, giving immediate feedback.
* Good, because there is one place to find and understand the full chart-release pipeline for any Platform Mesh service.
* Good, because the existing inconsistency (constructors split across two repos) is resolved permanently.
* Good, because the shared `.github` workflows become simpler - they dispatch to one repository for all chart and OCM operations instead of routing between two.
* Good, because `local-setup` no longer needs to poll the GHCR registry for the latest aggregator version - the component version is produced in the same repository and known at merge time, breaking the feedback cycle that has repeatedly pushed the project over the GHCR rate limit.
* Bad, because consolidating two active repositories requires a coordinated migration: history merging, CI secret migration, Renovate configuration updates, and retargeting of workflows in `platform-mesh/.github` and individual operator repositories.
* During the migration period, contributors must coordinate changes across both the old and new repository until all workflows are retargeted and `platform-mesh/ocm` is archived.

## References

* [ADR 001: SBOM Generation and OCM Component Restructuring](001-sbom-generation-and-ocm-component-restructuring.md) - established the three-component model (root, chart, image) that this ADR builds on.
* [`platform-mesh/helm-charts`](https://github.com/platform-mesh/helm-charts) - current Helm chart repository.
* [`platform-mesh/ocm`](https://github.com/platform-mesh/ocm) - current OCM assembly repository.
* [`platform-mesh/.github`](https://github.com/platform-mesh/.github) - shared reusable workflows that orchestrate cross-repository OCM updates.
