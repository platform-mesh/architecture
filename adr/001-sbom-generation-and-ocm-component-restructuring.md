# ADR 001: SBOM Generation and OCM Component Restructuring for Docker Images

| Status  | Proposed   |
|---------|------------|
| Date    | 2026-02-19 |

## Context and Problem Statement

Platform Mesh uses the [Open Component Model (OCM)](https://ocm.software) to package and distribute software components. Today, the chart pipeline (`pipeline-chart.yml`) creates a single OCM component per software component that bundles the Helm chart, the OCI image reference, and source metadata together (see the `component-constructor.yaml` template used by `job-ocm.yml`).

No Software Bill of Materials (SBOM) is currently generated for any container image in the platform. Supply-chain security requirements increasingly demand SBOMs in standard formats (CycloneDX, SPDX) for every shipped artifact.

Adding SBOMs raises a structural question: **where should image SBOMs live in the OCM component?** Adding them to the existing component that is built in the chart pipeline is suboptimal because:

* The SBOM describes the *image*, not the chart — it should be produced and co-located with the image at build time, where full build context is available for accurate scanning.
* The chart pipeline runs after image builds; generating SBOMs there introduces a delay between image creation and SBOM availability.
* Coupling SBOMs to the chart pipeline means every chart release must also regenerate SBOMs, even if the image hasn't changed.

## Decision Drivers

* **Supply-chain security**: All container images must have SBOMs in CycloneDX JSON and SPDX JSON formats.
* **OCM as the source of truth for versions**: Version resolution should be driven by the OCM component model, not by `appVersion` fields scattered across Helm charts. OCM components and their references are the authoritative record of which image version belongs to which release.
* **Clean ownership**: An image's SBOM should be produced and packaged alongside the image at build time, not in a downstream chart pipeline.
* **Reusability**: SBOM generation must work for any image pipeline (`pipeline-golang-app.yml`, `pipeline-node-app.yml`, custom image builds).
* **OCM spec compliance**: SBOMs should use the standard `sbom` resource type with appropriate media types.

## Considered Options

1. **Add SBOMs to the existing chart-pipeline OCM component** — keep the current single-component model, add SBOM resources alongside chart + image.
2. **Create separate image OCM components in source-repo pipelines** — each image gets its own OCM component containing the image reference and SBOMs; the chart component references image components via `componentReferences`.
3. **Generate SBOMs in the chart pipeline and store them externally** — generate SBOMs in the chart pipeline but push them to the OCI registry as standalone artifacts rather than embedding in OCM.

## Decision Outcome

Chosen option: **"Create separate image OCM components in source-repo pipelines"**, because it co-locates each image with its SBOMs at build time, establishes OCM as the source of truth for version resolution, and aligns with OCM's component-reference model for composing larger deliverables from smaller, independently versioned components.

### Consequences

* Good, because every container image gets CycloneDX and SPDX SBOMs generated at build time with full provenance.
* Good, because OCM becomes the authoritative source for version resolution — image versions are tracked in component references, not in `appVersion` fields.
* Good, because SBOM generation is reusable across all app pipeline types via shared workflows.
* Good, because image OCM components can be consumed independently of charts (e.g. for vulnerability scanning).
* Good, because components with multiple images (e.g. `terminal-controller-manager` with operator + terminal images) are modeled cleanly — one component reference per image.
* Bad, because the total number of OCM components in the registry increases (one additional per image-producing repo).
* Bad, because existing `component-constructor.yaml` files in all chart repos need to be migrated from inline image resources to `componentReferences`.
* Bad, because the prerelease aggregator (`platform-mesh/ocm`) may need updates to resolve the new component-reference tree.

### Confirmation

* `ocm get resources <image-component>` returns SBOM resources with `type: sbom` and correct media types (`application/vnd.cyclonedx+json`, `application/spdx+json`).
* `ocm get references <chart-component>` shows image component references.
* The prerelease aggregator (`ocm.yaml`) successfully resolves the full component tree including image sub-components.
* All existing CI pipelines remain green after migration.

## Pros and Cons of the Options

### Option 1: Add SBOMs to the existing chart-pipeline OCM component

Keep the current model where `job-ocm.yml` creates a single OCM component with chart + image + sources, and add SBOM resources to this same component.

* Good, because it requires minimal pipeline changes — only `job-ocm.yml` and `component-constructor.yaml` need updates.
* Good, because the total number of OCM components stays the same.
* Bad, because SBOMs are disconnected from the image build — a rebuild of the image without a chart release leaves stale SBOMs.
* Bad, because the chart pipeline must scan images it did not build, meaning SBOM accuracy depends on scanning a remote artifact rather than the local build context.
* Bad, because components with multiple images (e.g. `terminal-controller-manager`) require multiple image resources and multiple SBOM pairs in one component with no clean association.

### Option 2: Create separate image OCM components in source-repo pipelines

Each source repo that produces a Docker image also creates an OCM component containing the image reference and its SBOMs. The chart OCM component keeps the Helm chart and sources but references image components via `componentReferences` instead of embedding image resources directly.

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
Source repo: build → test → docker push → syft SBOM → OCM(image+SBOMs) → trigger OCM aggregator
Chart repo:  test chart → release chart → OCM(chart + componentRef:image + sources) → transfer → trigger OCM aggregator
OCM repo:    resolve latest image + chart components → build platform-mesh aggregate → transfer
```

The key difference: source repos no longer trigger chart version updates. Instead, they trigger the OCM aggregator (`platform-mesh/ocm`) directly after publishing their image component. The aggregator resolves the latest versions of all image and chart components and builds a new platform-mesh aggregate. Chart pipelines only run when the chart itself changes.

### New Reusable Workflows (`.github` repo)

| Workflow | Purpose |
|----------|---------|
| `job-sbom.yml` | Generates CycloneDX JSON and SPDX JSON SBOMs for a given image using [syft](https://github.com/anchore/syft) |
| `job-image-ocm.yml` | Creates an OCM component for an image and its SBOMs, then transfers to the OCM registry |

### Modified Reusable Workflows

| Workflow | Change |
|----------|--------|
| `pipeline-golang-app.yml` | Replace `job-chart-version-update` with `job-sbom` + `job-image-ocm` + `job-ocm-version-update` (trigger aggregator directly) |
| `pipeline-node-app.yml` | Same as above |
| `job-ocm.yml` | Support `componentReferences` for images instead of direct `image` resources |
| `pipeline-chart.yml` | Chart pipeline only runs on actual chart changes; no longer triggered by image version bumps |

### Naming Convention

Image OCM components use a `-image` suffix to avoid collision with chart component names. Chart components keep their current names — no migration of existing names is needed.

| Component Type | Name Pattern | Example |
|----------------|-------------|---------|
| Chart (existing) | `github.com/platform-mesh/<chart-name>` | `github.com/platform-mesh/security-operator` |
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

Image components and chart resources are versioned independently. The aggregator (`platform-mesh/ocm`) calculates the software component version by comparing old and new dependency versions, determining the semver bump level (patch/minor/major), and applying the same bump to the software component. If both dependencies changed, the higher bump wins. The version continues from the current value — no reset on adoption.

| Change | Dependency delta | Software component delta |
|--------|-----------------|--------------------------|
| Image patch | `v0.10.9` → `v0.10.10` | `0.42.3` → `0.42.4` |
| Chart minor | `0.19.16` → `0.20.0` | `0.42.4` → `0.43.0` |
| Image major + chart patch | `v0.10.10` → `v1.0.0` + `0.20.0` → `0.20.1` | `0.43.0` → `1.0.0` (major wins) |

### Decoupling Chart Releases from Image Releases

Once image OCM components are created in the source-repo pipeline, the chart no longer needs to know the exact image version at build time. Today the flow is:

1. Source repo builds image `1.2.3`, then calls `job-chart-version-update.yml` to update `appVersion` in the chart's `Chart.yaml`.
2. This triggers a PR / commit in the helm-charts repo, which triggers the chart pipeline.
3. The chart pipeline reads `appVersion` from `Chart.yaml` to resolve the image reference for the OCM component.

With the new model, the image version is captured in the image OCM component, and the OCM aggregator resolves which versions belong together.

**Proposed change:** Set `appVersion` to a static placeholder (e.g. `0.0.0`) in `Chart.yaml`. The actual image version used at deployment time is determined by the OCM component reference, not by `appVersion`. This means:

* **No more automated `appVersion` bumps** — source-repo pipelines no longer call `job-chart-version-update.yml`. Instead, after publishing the image OCM component, they trigger the OCM aggregator (`platform-mesh/ocm`) directly.
* **OCM aggregator builds new platform-mesh versions** — the aggregator resolves the latest versions of all image and chart components and builds a new platform-mesh aggregate. This happens both when an image is published (triggered by source repo) and when a chart is released (triggered by chart pipeline).
* **`appVersion` in Chart.yaml becomes informational** — It no longer drives the image tag used at deployment time.

**Trade-off:** Updating `appVersion` on every image release is common practice in many Helm charts and provides a useful signal about which image version a chart was last tested with. On the other hand, it artificially inflates chart versions without any actual chart changes. With OCM as the authoritative source for version resolution, the `appVersion` field loses its operational role and the trade-off favors fewer, more meaningful chart releases. While `appVersion` is removed from chart version coupling, the software component version tracked by the aggregator provides equivalent traceability — querying the OCM component always reveals which chart and image versions it bundles.

### Third-Party / Upstream Images

The same pattern should be applied to third-party images that are rebuilt from the `platform-mesh/ocm` repo (e.g. Keycloak, PostgreSQL, and other upstream images). Each upstream image build should produce its own OCM image component with SBOMs, so that chart components referencing these images can use `componentReferences` consistently — regardless of whether the image is built in-house or rebuilt from upstream.

### Migration Strategy

1. Implement `job-sbom.yml` and `job-image-ocm.yml` as new reusable workflows.
2. Update `pipeline-golang-app.yml` and `pipeline-node-app.yml`: replace `job-chart-version-update` with `job-sbom` + `job-image-ocm` + direct trigger to OCM aggregator.
3. Update `job-ocm.yml` to support `componentReferences` (backward-compatible — existing constructors keep working).
4. Migrate chart repos one-by-one: switch `component-constructor.yaml` from inline image resources to component references, set `appVersion` to `0.0.0`.
5. Apply the same SBOM + image-component pattern to upstream/third-party image builds.
6. Update the OCM aggregator (`platform-mesh/ocm`) to resolve the new component-reference tree.
7. Remove `job-chart-version-update.yml` usage from app pipelines once all charts are migrated.