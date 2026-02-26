# ADR 001: SBOM Generation and OCM Component Restructuring for Docker Images

| Status  | Proposed   |
|---------|------------|
| Date    | 2026-02-19 |

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

## Considered Options

1. **Add SBOMs to the existing chart-pipeline OCM component** — keep the current single-component model, add SBOM resources alongside chart + image.
2. **Create separate image OCM components in source-repo pipelines** — each image gets its own OCM component containing the image reference and SBOMs; the software component references image components via `componentReferences`.
3. **Generate SBOMs in the chart pipeline and store them externally** — generate SBOMs in the chart pipeline but push them to the OCI registry as standalone artifacts rather than embedding in OCM.

## Decision Outcome

Chosen option: **"Create separate image OCM components in source-repo pipelines"**, because it co-locates each image with its SBOMs at build time, establishes OCM as the authority for image version resolution in the standard deployment while preserving the chart-version-update flow for standalone Helm compatibility, and aligns with OCM's component-reference model for composing larger deliverables from smaller, independently versioned components.

### Consequences

* Good, because every container image gets CycloneDX and SPDX SBOMs generated at build time with full provenance.
* Good, because OCM becomes the authority for image version resolution in the standard deployment — image versions are determined by component references, not `appVersion`. The `appVersion` field is still updated for standalone Helm compatibility.
* Good, because SBOM generation is reusable across all app pipeline types via shared workflows.
* Good, because image OCM components can be consumed independently of charts (e.g. for vulnerability scanning).
* Good, because components with multiple images (e.g. `terminal-controller-manager` with operator + terminal images) are modeled cleanly — one component reference per image.
* Bad, because the total number of OCM components in the registry increases (one additional per image-producing repo).
* Bad, because existing [`component-constructor.yaml`](https://github.com/platform-mesh/helm-charts/blob/main/.ocm/component-constructor.yaml) files in all chart repos need to be migrated from inline image resources to `componentReferences`.
* Bad, because the prerelease aggregator (`platform-mesh/ocm`) may need updates to resolve the new component-reference tree.

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
* Neutral, because it introduces more OCM components, but each is small and self-contained.
* Bad, because it requires changes to both app pipelines (new SBOM + OCM jobs) and chart pipelines (switch to component references).
* Bad, because migration must be coordinated across repos.

### Option 3: Generate SBOMs in chart pipeline, store externally

Generate SBOMs during the chart pipeline (similar to Option 1) but push them as standalone OCI artifacts to the registry rather than embedding them in the OCM component.

* Good, because SBOMs are accessible via standard OCI tooling.
* Good, because no changes to the OCM component model are needed.
* Bad, because SBOMs are disconnected from the OCM component model — there is no standard way to discover which SBOMs belong to which component.
* Bad, because the chart pipeline still needs to scan images it did not build.
* Bad, because it introduces a parallel distribution mechanism outside OCM, reducing the value of using OCM as the single source of truth.

## More Information

### Proposed Architecture

**Current flow:**

```
Source repo: build → test → docker push ──────────────────────→ trigger chart version update
Chart repo:  test chart → release chart → OCM(chart+image+sources) → transfer → trigger aggregator
```

**Proposed flow:**

```
Source repo: build → test → docker push → syft SBOM → OCM(image+SBOMs) → trigger chart version update (as today)
Chart repo:  test chart → release chart → OCM(chart + componentRef:image + sources) → transfer → trigger aggregator
```

The key difference from today: source-repo pipelines now additionally generate SBOMs and publish an image OCM component **before** triggering [`job-chart-version-update.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/job-chart-version-update.yml). The chart-version-update flow remains unchanged — image builds still bump `appVersion` and `chartVersion` in the helm-charts repo for standalone Helm compatibility. The chart pipeline's OCM step is updated to use `componentReferences` to the image OCM component instead of embedding the image resource directly. The chart pipeline uses `appVersion` to pin the `componentReference` to the matching image OCM component version, ensuring the software component always references the correct image component version.

### New Reusable Workflows (`.github` repo)

| Workflow | Purpose |
|----------|---------|
| `job-sbom.yml` | Generates CycloneDX JSON and SPDX JSON SBOMs for a given image using [syft](https://github.com/anchore/syft) |
| `job-image-ocm.yml` | Creates an OCM component for an image and its SBOMs, then transfers to the OCM registry |

### Modified Reusable Workflows

| Workflow | Change |
|----------|--------|
| [`pipeline-golang-app.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/pipeline-golang-app.yml) | Add `job-sbom` + `job-image-ocm` before the existing `job-chart-version-update` step. The image OCM component is published first, then chart version update is triggered as today. |
| [`pipeline-node-app.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/pipeline-node-app.yml) | Same as above |
| [`job-ocm.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/job-ocm.yml) | Support `componentReferences` for images instead of direct `image` resources |
| [`pipeline-chart.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/pipeline-chart.yml) | Still triggered by chart version updates from image builds. OCM step updated to use `componentReferences`. |

### Naming Convention

Image OCM components use a `-image` suffix to avoid collision with software component names. Software components keep their current names — no migration of existing names is needed.

| Component Type | Name Pattern | Example |
|----------------|-------------|---------|
| Software (existing) | `github.com/platform-mesh/<component-name>` | `github.com/platform-mesh/security-operator` |
| Image (new) | `github.com/platform-mesh/<repo-name>-image` | `github.com/platform-mesh/security-operator-image` |

### OCM Component Structure

**Image component (new, per image):**

```yaml
components:
  - name: github.com/platform-mesh/<repo-name>-image
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

**Software component (updated, chart resource + image componentReference):**

```yaml
components:
  - name: "{{ .COMPONENT_NAME }}"
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
    componentReferences:
      - name: image
        componentName: "github.com/platform-mesh/{{ .IMAGE_COMPONENT_NAME }}-image"
        version: "{{ .APP_VERSION }}"
    sources:
      - name: chart
        type: git
        version: "{{ .VERSION }}"
        access:
          commit: "{{ .COMMIT }}"
          repoUrl: "{{ .CHART_REPO }}"
          type: gitHub
```

### Software Component Versioning

The chart version is the leading version. Since image builds trigger chart version updates (bumping both `appVersion` and `chartVersion`), every image change results in a new chart release. The software component version in the aggregator (`platform-mesh/ocm`) follows the chart version directly. The image OCM component is versioned independently (matching the image tag). The chart pipeline uses `appVersion` to pin the `componentReference` to the matching image component version, ensuring consistency between chart and image in the resulting OCM component.

### Integrating Image OCM Components with the Existing Chart Release Flow

The existing chart-version-update flow is preserved. The structural change is how the software component models image dependencies — using `componentReferences` to the image OCM component instead of embedding the image resource directly.

**Today:**

1. Source repo builds image `1.2.3`, calls [`job-chart-version-update.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/job-chart-version-update.yml) to bump `appVersion` and `chartVersion`.
2. Chart pipeline runs, reads `appVersion`, creates OCM component with chart + inline image resource + sources.

**Proposed:**

1. Source repo builds image `1.2.3`, generates SBOMs, publishes image OCM component (`<repo>-image` with image + SBOMs).
2. Source repo calls [`job-chart-version-update.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/job-chart-version-update.yml) to bump `appVersion` and `chartVersion` (unchanged from today).
3. Chart pipeline runs, creates OCM component with chart resource + `componentReference` to the image OCM component (using `appVersion` to resolve the correct image component version) + sources.

Key points:

* **OCM is authoritative for image versions** in the standard platform-mesh deployment. The OCM component reference determines which image version is deployed, not the `appVersion` field.
* **`appVersion` is updated for compatibility** — it continues to be bumped on every image build so that standalone Helm deployments (without OCM) still resolve the correct image tag. It also drives the `componentReference` version in the software component.
* **Chart version remains the leading version** for releases — the existing versioning model is unchanged.
* **Structural improvement:** Image SBOMs are now accessible via the component reference. The software component no longer embeds the image resource directly but references the image component that carries both the image and its SBOMs.

### Third-Party / Upstream Images

The same pattern should be applied to third-party images that are rebuilt from the `platform-mesh/ocm` repo (e.g. Keycloak, PostgreSQL, and other upstream images). Each upstream image build should produce its own OCM image component with SBOMs, so that software components referencing these images can use `componentReferences` consistently — regardless of whether the image is built in-house or rebuilt from upstream.

### Migration Strategy

1. Implement `job-sbom.yml` and `job-image-ocm.yml` as new reusable workflows.
2. Update [`pipeline-golang-app.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/pipeline-golang-app.yml) and [`pipeline-node-app.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/pipeline-node-app.yml): add `job-sbom` + `job-image-ocm` before the existing `job-chart-version-update` step. The image OCM component is published before triggering the chart version update.
3. Update [`job-ocm.yml`](https://github.com/platform-mesh/.github/blob/main/.github/workflows/job-ocm.yml) to support `componentReferences` (backward-compatible — existing constructors keep working).
4. Migrate chart repos one-by-one: switch [`component-constructor.yaml`](https://github.com/platform-mesh/helm-charts/blob/main/.ocm/component-constructor.yaml) from inline image resources to component references. `appVersion` continues to be updated by `job-chart-version-update` as today.
5. Apply the same SBOM + image-component pattern to upstream/third-party image builds.
6. Update the OCM aggregator (`platform-mesh/ocm`) to resolve the new component-reference tree.