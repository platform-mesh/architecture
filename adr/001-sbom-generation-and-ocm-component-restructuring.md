# ADR 001: SBOM Generation and OCM Component Restructuring for Docker Images

| Status  | Approved   |
|---------|------------|
| Date    | 2026-03-02 |

## Context and Problem Statement

Platform Mesh uses the [Open Component Model (OCM)](https://ocm.software) to package and distribute software components. Today, the chart pipeline ([`pipeline-chart.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/pipeline-chart.yml)) creates a single OCM component per software component that bundles the Helm chart, the OCI image reference, and source metadata together (see the [`component-constructor.yaml`](https://github.com/platform-mesh/helm-charts/blob/main/.ocm/component-constructor.yaml) template used by [`job-ocm.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/job-ocm.yml)).

No Software Bill of Materials (SBOM) is currently generated for any container image in the platform. Supply-chain security requirements increasingly demand SBOMs in standard formats (CycloneDX, SPDX) for every shipped artifact.

Adding SBOMs raises a structural question: **where should image SBOMs live in the OCM component?** Adding them to the existing component that is built in the chart pipeline is suboptimal because:

* The SBOM describes the *image*, not the chart — it should be produced and co-located with the image at build time, where full build context is available for accurate scanning.
* The chart pipeline runs after image builds; generating SBOMs there introduces a delay between image creation and SBOM availability.
* Coupling SBOMs to the chart pipeline means every chart release must also regenerate SBOMs, even if the image hasn't changed.

## Decision Drivers

* **Supply-chain security**: All container images must have SBOMs in CycloneDX JSON and SPDX JSON formats.
* **OCM as the authority for image versions**: In the standard platform-mesh deployment, OCM component references determine which image version is used — not the `appVersion` field in `Chart.yaml`.
* **Clean ownership**: An image's SBOM should be produced and packaged alongside the image at build time, not in a downstream chart pipeline.
* **Reusability**: SBOM generation must work for any image pipeline ([`pipeline-golang-app.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/pipeline-golang-app.yml), [`pipeline-node-app.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/pipeline-node-app.yml), custom image builds).
* **OCM spec compliance**: SBOMs should use the standard `sbom` resource type with appropriate media types.
* **Standalone Helm compatibility**: Charts should remain usable without OCM by maintaining `appVersion` updates.

## Considered Options

1. **Add SBOMs to the existing chart-pipeline OCM component** — keep the current single-component model, add SBOM resources alongside chart + image.
2. **Create separate image OCM components in source-repo pipelines** — each image gets its own OCM component containing the image reference and SBOMs; the software component references image components via `componentReferences`.
3. **Separate chart and image OCM components under a root component** — both chart and image get their own OCM component; a root component references both via `componentReferences`, keeping all artifacts independently versioned and reusable.

## Decision Outcome

Chosen option: **"Separate chart and image OCM components under a root component"**, because it cleanly separates all artifact types into independently versioned components, follows established OCM conventions for composing deliverables, provides a natural extension point for future cross-cutting metadata (deployment config, signing artifacts), and explicitly establishes version dependencies between chart and image — addressing the need for traceable multi-version support.

### Consequences

* Good, because every container image gets CycloneDX and SPDX SBOMs generated at build time with full provenance.
* Good, because OCM becomes the authority for image version resolution in the standard deployment — image versions are determined by the root component's references to chart and image components, not `appVersion`.
* Good, because SBOM generation is reusable across all app pipeline types via shared workflows.
* Good, because image and chart components are independently versioned and consumable — image components can be consumed for vulnerability scanning without pulling charts, and chart components can be consumed without pulling images.
* Good, because the root component explicitly establishes the dependency between a specific chart version and a specific image version, enabling traceable multi-version support (e.g. `v1.*.*` and `v2.*.*` in parallel).
* Good, because the root component provides a clean extension point for deployment metadata, signing artifacts, or other cross-cutting resources without polluting chart or image components.
* Good, because components with multiple images (e.g. `terminal-controller-manager` with operator + terminal images) are modeled cleanly — one component reference per image.
* Good, because `appVersion` continues to be updated, preserving standalone Helm compatibility for users deploying without OCM.
* Bad, because the total number of OCM components in the registry increases (root + chart + image per software component).
* Bad, because existing [`component-constructor.yaml`](https://github.com/platform-mesh/helm-charts/blob/main/.ocm/component-constructor.yaml) files in chart repos need to be migrated to produce chart-only components instead of the current monolithic component.
* Bad, because the `ocm` repo needs a new pipeline (`pipeline-root-component.yml`) to assemble root components, and the aggregator may need updates to resolve the new component-reference tree.

## Pros and Cons of the Options

### Option 1: Add SBOMs to the existing chart-pipeline OCM component

Keep the current model where [`job-ocm.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/job-ocm.yml) creates a single OCM component with chart + image + sources, and add SBOM resources to this same component.

* Good, because it requires minimal pipeline changes — only [`job-ocm.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/job-ocm.yml) and [`component-constructor.yaml`](https://github.com/platform-mesh/helm-charts/blob/main/.ocm/component-constructor.yaml) need updates.
* Good, because the total number of OCM components stays the same.
* Bad, because SBOMs are disconnected from the image build — a rebuild of the image without a chart release leaves stale SBOMs.
* Bad, because the chart pipeline must scan images it did not build, meaning SBOM accuracy depends on scanning a remote artifact rather than the local build context.
* Bad, because components with multiple images (e.g. `terminal-controller-manager`) require multiple image resources and multiple SBOM pairs in one component with no clean association.

### Option 2: Create separate image OCM components in source-repo pipelines

Each source repo that produces a Docker image also creates an OCM component containing the image reference and its SBOMs. The software component keeps the Helm chart and sources but references image components via `componentReferences` instead of embedding image resources directly.

* Good, because image + SBOMs are produced together at build time with complete build context.
* Good, because components with multiple images (e.g. `terminal-controller-manager`) are modeled naturally — one `componentReference` per image.
* Good, because image components are independently versioned and consumable.
* Bad, because it introduces more OCM components, though each is small and self-contained.
* Bad, because it requires changes to both app pipelines (new SBOM + OCM jobs) and chart pipelines (switch to component references).
* Bad, because migration must be coordinated across repos.
* Bad, because the software component embeds the chart as a resource while referencing images as components — this asymmetry means the chart cannot be consumed independently and the software component mixes resource types with component references.

### Option 3: Separate chart and image OCM components under a root component

Each image gets its own OCM component with SBOMs (same as Option 2). The chart is also extracted into its own OCM component. A **root component** references both chart and image components via `componentReferences`, establishing explicit version dependencies between all artifacts. The root component version follows the chart version (the leading version), while the image component is versioned independently.

```
root-component (github.com/platform-mesh/security-operator)
  ├── componentRef: chart (github.com/platform-mesh/helm-charts/security-operator)
  │     └── resource: chart
  └── componentRef: image (github.com/platform-mesh/images/security-operator)
        ├── resource: image
        ├── resource: sbom-cyclonedx
        └── resource: sbom-spdx
```

* Good, because image + SBOMs are produced together at build time with complete build context.
* Good, because both chart and image are independently versioned and reusable — neither is embedded as a resource in the other's component.
* Good, because the root component provides a clean place for deployment metadata, signing artifacts, or other cross-cutting resources without polluting chart or image components.
* Good, because components with multiple images are modeled naturally — additional image component references are added to the root.
* Good, because it follows established OCM conventions for composing deliverables from independent building blocks.
* Good, because the root component explicitly establishes the relationship between chart and image versions, enabling traceable multi-version support.
* Good, because each pipeline has a single responsibility — source-repo pipelines produce image components, chart pipelines produce chart components, and the `ocm` repo assembles root components.
* Bad, because it introduces the most OCM components (root + chart + image per software component).
* Bad, because migration is the most involved — existing chart-pipeline OCM components must be converted to chart-only components, a new root-component pipeline must be created in the `ocm` repo, and all image resources must be extracted to image components.


## More Information

### Proposed Architecture

```
Source repo: build → test → docker push → syft SBOM → OCM(image+SBOMs) → trigger chart version update
Chart repo:  test chart → release chart → OCM(chart component) → trigger ocm repo
OCM repo:    assemble root component (refs to chart+image) → transfer → trigger aggregator
```

### New Reusable Workflows (`.github` repo)

| Workflow | Purpose |
|----------|---------|
| `job-sbom.yml` | Generates CycloneDX JSON and SPDX JSON SBOMs for a given image using [syft](https://github.com/anchore/syft) |
| `job-image-ocm.yml` | Creates an OCM component for an image and its SBOMs, then transfers to the OCM registry |
| `job-chart-ocm.yml` | Creates an OCM component for a Helm chart, then transfers to the OCM registry |

### Modified Reusable Workflows

| Workflow | Change |
|----------|--------|
| [`pipeline-golang-app.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/pipeline-golang-app.yml) | Add `job-sbom` + `job-image-ocm` before the existing `job-chart-version-update` step. The image OCM component is published first, then chart version update is triggered as today. |
| [`pipeline-node-app.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/pipeline-node-app.yml) | Same as above |
| [`job-ocm.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/job-ocm.yml) | Produce only a chart component instead of the current monolithic component. After publishing, trigger the `ocm` repo pipeline to assemble the root component. |
| [`pipeline-chart.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/pipeline-chart.yml) | Still triggered by chart version updates from image builds. OCM step updated to produce chart component only and trigger `ocm` repo. |

### New Pipeline (`ocm` repo)

| Workflow | Purpose |
|----------|---------|
| `pipeline-root-component.yml` | Triggered by chart pipelines. Assembles the root OCM component with `componentReferences` to the chart and image components, then transfers to the OCM registry and triggers the aggregator. Receives component name, chart version, and `appVersion` as inputs. |

### Naming Convention

Component names avoid technical artifact suffixes (e.g. `-image`, `-chart`) and instead use path-based namespacing to group components by type. This follows the OCM convention that component names should be free from technical artifact names, allowing storage and access technologies to change without renaming components.

| Component Type | Name Pattern | Example |
|----------------|-------------|---------|
| Root (existing name, unchanged) | `github.com/platform-mesh/<component-name>` | `github.com/platform-mesh/security-operator` |
| Chart (new) | `github.com/platform-mesh/helm-charts/<chart-name>` | `github.com/platform-mesh/helm-charts/security-operator` |
| Image (new) | `github.com/platform-mesh/images/<repo-name>` | `github.com/platform-mesh/images/security-operator` |

### OCM Component Structure

**Image component (new, per image — produced by source-repo pipeline):**

```yaml
components:
  - name: github.com/platform-mesh/images/<repo-name>
    version: "{{ .APP_VERSION }}"
    provider:
      name: Platform Mesh Team
    resources:
      - name: image
        type: ociImage
        relation: external
        version: "{{ .APP_VERSION }}"
        access:
          type: ociArtifact
          imageReference: "{{ .IMAGE_NAME }}:{{ .APP_VERSION }}"
      - name: sbom-cyclonedx
        type: sbom
        mediaType: application/vnd.cyclonedx+json
        input:
          type: file
          path: sbom-cyclonedx.json
      - name: sbom-spdx
        type: sbom
        mediaType: application/spdx+json
        input:
          type: file
          path: sbom-spdx.json
    sources:
      - name: source
        type: git
        version: "{{ .APP_VERSION }}"
        access:
          commit: "{{ .COMMIT }}"
          repoUrl: "{{ .SOURCE_REPO }}"
          type: gitHub
```

**Chart component (new — produced by chart pipeline):**

```yaml
components:
  - name: github.com/platform-mesh/helm-charts/<chart-name>
    version: "{{ .VERSION }}"
    provider:
      name: Platform Mesh Team
    resources:
      - name: chart
        type: helmChart
        relation: external
        version: "{{ .VERSION }}"
        access:
          type: ociArtifact
          imageReference: "{{ .CHART_OCI_PATH }}:{{ .VERSION }}"
    sources:
      - name: source
        type: git
        version: "{{ .VERSION }}"
        access:
          commit: "{{ .COMMIT }}"
          repoUrl: "{{ .CHART_REPO }}"
          type: gitHub
```

**Root component (new — produced by `ocm` repo pipeline, references chart + image):**

```yaml
components:
  - name: "{{ .COMPONENT_NAME }}"
    version: "{{ .VERSION }}"
    provider:
      name: Platform Mesh Team
    componentReferences:
      - name: chart
        componentName: "github.com/platform-mesh/helm-charts/{{ .CHART_NAME }}"
        version: "{{ .VERSION }}"
      - name: image
        componentName: "github.com/platform-mesh/images/{{ .IMAGE_COMPONENT_NAME }}"
        version: "{{ .APP_VERSION }}"
```

### Software Component Versioning

The chart version is the leading version. Since image builds trigger chart version updates (bumping both `appVersion` and `chartVersion`), every image change results in a new chart release. The root component version follows the chart version. The chart component is versioned by the chart version. The image component is versioned independently (matching the image tag). The `ocm` repo pipeline assembles the root component using `appVersion` to pin the image component reference to the matching image component version, establishing an explicit dependency between chart and image versions.

This explicit version pinning addresses the need for traceable multi-version support — when maintaining multiple version streams (e.g. `v1.*.*` and `v2.*.*` in parallel), the root component clearly records which image version belongs to which chart version.

### Standalone Helm Compatibility

The `appVersion` field in `Chart.yaml` continues to be updated on every image build, preserving the ability to install charts without OCM. Users who deploy via Helm directly (without OCM) still get the correct image version resolved from `appVersion`. Users who prefer to pin image versions via values can use Renovate to follow the OCM image component for automated updates.

When deploying via OCM, the `appVersion` value in the chart is overridden by the image version from the component reference, making OCM the authoritative source for image versions in the standard deployment.

### Integrating Image OCM Components with the Existing Chart Release Flow

The existing chart-version-update flow is preserved. The structural change is how the software component models its artifacts — chart and image are both referenced components under a root, rather than a monolithic component with embedded resources.

**Today:**

1. Source repo builds image `1.2.3`, calls [`job-chart-version-update.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/job-chart-version-update.yml) to bump `appVersion` and `chartVersion`.
2. Chart pipeline runs, reads `appVersion`, creates OCM component with chart + inline image resource + sources.

**Proposed:**

1. Source repo builds image `1.2.3`, generates SBOMs, publishes image OCM component (`images/<repo-name>` with image + SBOMs).
2. Source repo calls [`job-chart-version-update.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/job-chart-version-update.yml) to bump `appVersion` and `chartVersion` (unchanged from today).
3. Chart pipeline runs, creates the **chart component** (`helm-charts/<chart-name>`) containing only the Helm chart resource, then triggers the `ocm` repo.
4. The `ocm` repo pipeline assembles the **root component** (`<component-name>`) referencing the chart component (at the chart version) and the image component (at `appVersion`), then transfers and triggers the aggregator.

Key points:

* **OCM is authoritative for image versions** in the standard platform-mesh deployment. The root component's reference to the image component determines which image version is deployed.
* **`appVersion` is updated for compatibility** — it continues to be bumped on every image build so that standalone Helm deployments (without OCM) still resolve the correct image tag. It also drives the image `componentReference` version in the root component.
* **Chart version remains the leading version** for releases — the existing versioning model is unchanged.
* **Explicit version binding:** The root component records exactly which chart version and image version belong together, enabling reproducible deployments and multi-version support.
* **Clean separation of responsibilities:** Each pipeline owns exactly one artifact type — source repos own images, chart repos own charts, and the `ocm` repo owns the composition of root components.

### Third-Party / Upstream Images

The same pattern should be applied to third-party images that are rebuilt from the `platform-mesh/ocm` repo (e.g. Keycloak, PostgreSQL, and other upstream images). Each upstream image build should produce its own OCM image component with SBOMs, so that root components referencing these images can use `componentReferences` consistently — regardless of whether the image is built in-house or rebuilt from upstream.

### Migration Strategy

1. Implement `job-sbom.yml`, `job-image-ocm.yml`, and `job-chart-ocm.yml` as new reusable workflows.
2. Update [`pipeline-golang-app.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/pipeline-golang-app.yml) and [`pipeline-node-app.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/pipeline-node-app.yml): add `job-sbom` + `job-image-ocm` before the existing `job-chart-version-update` step. The image OCM component is published before triggering the chart version update.
3. Implement `pipeline-root-component.yml` in the `ocm` repo: receives component name, chart version, and `appVersion` as inputs, assembles the root component with `componentReferences`, transfers to the registry, and triggers the aggregator.
4. Update [`job-ocm.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/job-ocm.yml) to produce only a chart component and trigger the `ocm` repo pipeline (backward-compatible — existing constructors keep working during migration).
5. Migrate chart repos one-by-one: switch [`component-constructor.yaml`](https://github.com/platform-mesh/helm-charts/blob/main/.ocm/component-constructor.yaml) from the monolithic component to chart-only component constructors. `appVersion` continues to be updated by `job-chart-version-update` as today.
6. Apply the same SBOM + image-component pattern to upstream/third-party image builds.
7. Update the OCM aggregator (`platform-mesh/ocm`) to resolve the new component-reference tree.
