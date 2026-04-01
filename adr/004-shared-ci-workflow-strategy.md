# ADR 004: Shared CI Workflow Strategy

| Status  | Proposed   |
|---------|------------|
| Date    | 2026-04-01 |

## Context and Problem Statement

Platform Mesh maintains a [`.github` repository](https://github.com/platform-mesh/.github) containing shared GitHub Actions workflows that are called by all repositories in the organization. These workflows include both **pipeline workflows** (end-to-end orchestrations like `pipeline-golang-app.yml`, `pipeline-node-app.yml`, `pipeline-chart.yml`) and **job workflows** (self-contained building blocks like `job-docker.yml`, `job-ocm.yml`, `job-sbom.yml`).

The pipeline workflows attempt to standardize the full CI/CD lifecycle across all repositories. However, experience from the team and from other open-source projects using this pattern (notably [Crossplane's shared build system](https://github.com/crossplane/build)) has shown that fully shared pipelines create significant problems:

* **Fragile coupling**: Any change to a shared pipeline workflow affects every repository that calls it. Changes cannot be tested in isolation — a modification intended for one repo may break others.
* **Lowest common denominator**: Repositories with slightly different needs (different inputs, additional steps, conditional logic) cannot deviate without adding `if/else` branches to the shared workflow, increasing complexity for all consumers.
* **Higher contribution barrier**: Contributors must understand how changes to the `.github` repo propagate across all projects before making even small CI modifications. Making a CI change requires PRs to two repositories.
* **Reduced velocity**: The need to coordinate changes across the shared repo and consuming repos slows down iteration. Contributors are reluctant to touch shared pipelines for fear of breaking other projects.
* **Indirection without value**: Many shared pipeline workflows simply dispatch to `make`, `task`, or a GitHub Action — the shared layer adds indirection without meaningful abstraction.

## Decision Drivers

* **Contributor velocity**: CI changes should be fast, local, and testable without cross-repo coordination.
* **Low contribution barrier**: New contributors should be able to modify CI without understanding the full organization-wide impact.
* **Maintainability at scale**: The approach must remain manageable as the number of repositories grows.
* **Consistency where it matters**: Shared building blocks (Docker builds, OCM packaging, SBOM generation) should remain standardized.
* **Agent-assisted maintenance**: AI coding agents can now efficiently compare, update, and create PRs across many repositories, reducing the cost of maintaining per-repo workflows.

## Considered Options

1. **Keep the current fully shared pipeline model** — all pipeline and job workflows live in the `.github` repo. Repositories call shared pipelines for their entire CI lifecycle.
2. **Move pipeline workflows into each repository; keep shared job workflows** — repositories own their end-to-end pipeline orchestration, calling shared job workflows for standardized building blocks.
3. **Eliminate all shared workflows** — every repository maintains its own complete CI configuration with no shared workflows.

## Decision Outcome

Chosen option: **"Move pipeline workflows into each repository; keep shared job workflows"**, because it gives each repository full control over its CI orchestration while preserving standardized building blocks for operations that genuinely benefit from sharing (Docker builds, OCM packaging, SBOM generation, etc.).

### Implementation

1. **Identify pipeline vs. job workflows** in the `.github` repo:
   - **Pipeline workflows** (to be moved into each repo): `pipeline-golang-app.yml`, `pipeline-node-app.yml`, `pipeline-chart.yml`, and similar end-to-end orchestration workflows.
   - **Job workflows** (to remain shared): `job-docker.yml`, `job-ocm.yml`, `job-sbom.yml`, `job-chart-version-update.yml`, and other self-contained, well-parameterized building blocks.
2. **Copy pipeline workflows into each consuming repository** under `.github/workflows/`, adjusting triggers, inputs, and steps to match the repository's specific needs.
3. **Remove pipeline workflows from the `.github` repo** once all repositories have been migrated.
4. **Keep shared job workflows well-parameterized and self-contained** — each job workflow should do one thing, accept clear inputs, and produce clear outputs. Avoid adding repository-specific conditionals.
5. **Use AI agents for cross-repo maintenance** — when a shared job workflow changes or a common pattern needs updating across repositories, use agent-assisted tooling to compare and create PRs across all affected repos.

### Consequences

* Good, because each repository can modify its CI pipeline independently without risk of breaking other projects.
* Good, because CI changes require only a single PR to the affected repository, improving contributor velocity.
* Good, because new contributors only need to understand the CI configuration of the repository they are working on.
* Good, because repositories with unique requirements (additional steps, different triggers, conditional logic) can accommodate them without polluting shared workflows.
* Good, because standardized building blocks (Docker, OCM, SBOM) remain shared, ensuring consistency for operations that genuinely benefit from it.
* Good, because AI coding agents reduce the cost of maintaining per-repo pipelines — cross-repo updates that previously required shared workflows can now be automated via agent-assisted PRs.
* Bad, because pipeline definitions are duplicated across repositories, creating the possibility of drift between repos over time.
* Bad, because updates to common pipeline patterns require touching multiple repositories instead of one (mitigated by agent-assisted tooling).
* Neutral, because migration requires a one-time effort to copy and customize pipeline workflows into each repository.

## Pros and Cons of the Options

### Option 1: Keep the current fully shared pipeline model

Continue with all pipeline and job workflows centralized in the `.github` repo.

* Good, because a single change to a pipeline workflow updates all repositories at once.
* Good, because pipeline definitions are not duplicated.
* Bad, because any pipeline change affects all consuming repositories and cannot be tested in isolation.
* Bad, because repositories with different needs require conditional logic in shared workflows, increasing complexity for all consumers.
* Bad, because contributors must coordinate across two repositories for CI changes, increasing the contribution barrier.
* Bad, because teams are reluctant to modify shared pipelines, leading to stagnation or workarounds.

### Option 2: Move pipeline workflows into each repository; keep shared job workflows

Repositories own their pipeline orchestration and call shared job workflows for standardized building blocks.

* Good, because repositories have full control over their CI lifecycle.
* Good, because shared job workflows provide consistency for operations that benefit from standardization.
* Good, because CI changes are local, fast, and testable.
* Bad, because pipeline logic is duplicated across repositories.
* Bad, because cross-repo pipeline updates require multiple PRs (mitigated by agent tooling).

### Option 3: Eliminate all shared workflows

Every repository maintains its own complete CI configuration with no dependencies on the `.github` repo.

* Good, because repositories are fully independent with no external CI dependencies.
* Bad, because standardized operations (Docker builds, OCM packaging, SBOM generation) would be duplicated, leading to inconsistency and higher maintenance burden.
* Bad, because bug fixes or improvements to building blocks must be applied to every repository individually.
* Bad, because there is no single source of truth for operations that should be standardized across the organization.

## More Information

### What Qualifies as a Shared Job Workflow

A workflow should remain shared in the `.github` repo if it meets **all** of these criteria:

* **Self-contained**: It performs a single, well-defined operation (e.g., build a Docker image, generate an SBOM, publish an OCM component).
* **Well-parameterized**: Its behavior is fully controlled by inputs — no repository-specific conditionals or hardcoded assumptions.
* **Stable interface**: Its inputs and outputs change infrequently.
* **Genuinely reusable**: At least three repositories use it with the same interface.

If a workflow requires `if/else` branches for different repositories or needs frequent repository-specific adjustments, it should be moved into consuming repositories.

### Migration Strategy

1. Start with one repository type (e.g., Go operators) as a pilot.
2. Copy the relevant pipeline workflow into the pilot repository and verify CI passes.
3. Remove the pilot repository's dependency on the shared pipeline workflow.
4. Roll out to remaining repositories, customizing pipelines as needed.
5. After all consumers are migrated, remove pipeline workflows from the `.github` repo.
6. Document the shared job workflow interface (inputs/outputs) to make per-repo pipelines easy to author.

### Prior Art

* [Crossplane build](https://github.com/crossplane/build) — a shared build system across 100+ provider repos. The team reports that the system has become effectively immutable because changes risk breaking many repositories, and contributors avoid modifying it.
