# ADR 004: Shared CI Workflow Strategy

| Status  | Approved   |
|---------|------------|
| Date    | 2026-04-01 |

## Context and Problem Statement

Platform Mesh maintains a [`.github` repository](https://github.com/platform-mesh/.github) with shared GitHub Actions workflows called by all repositories. These include **pipeline workflows** (end-to-end orchestrations like `pipeline-golang-app.yml`, `pipeline-chart.yml`) and **job workflows** (self-contained building blocks like `job-docker.yml`, `job-ocm.yml`, `job-sbom.yml`).

Fully shared pipelines create problems observed both internally and in other open-source projects using this pattern:

* **Fragile coupling**: Any pipeline change affects every consuming repository and cannot be tested in isolation.
* **Lowest common denominator**: Repositories with different needs require `if/else` branches in shared workflows, increasing complexity for all consumers.
* **Higher contribution barrier**: CI changes require PRs to two repositories and understanding of cross-repo impact.
* **Indirection without value**: Many shared pipelines simply dispatch to `make`, `task`, or a GitHub Action — the shared layer adds no meaningful abstraction.

## Decision Drivers

* CI changes should be fast, local, and testable without cross-repo coordination.
* New contributors should not need to understand organization-wide CI impact for routine changes.
* Standardized building blocks (Docker builds, OCM packaging, SBOM generation) should remain consistent.
* AI coding agents can now efficiently maintain per-repo workflows via cross-repo PRs, reducing the traditional cost of duplication.

## Considered Options

1. **Keep the current fully shared pipeline model** — all workflows live in `.github`.
2. **Move pipeline workflows into each repository; keep shared job workflows** — repositories own orchestration, shared repo provides building blocks.
3. **Eliminate all shared workflows** — full independence per repository.

## Decision Outcome

Chosen option: **"Move pipeline workflows into each repository; keep shared job workflows"**, because it gives each repository full control over its CI orchestration while preserving standardized building blocks for operations that genuinely benefit from sharing.

A workflow should remain shared only if it is **self-contained**, **well-parameterized** (no repo-specific conditionals), has a **stable interface**, and is **genuinely reusable** across multiple repositories. If a workflow requires `if/else` branches for different repos, it should be moved into consuming repositories.

### Consequences

* Good, because each repository can modify its CI pipeline independently without risk of breaking other projects.
* Good, because CI changes require only a single PR to the affected repository.
* Good, because repositories with unique requirements can accommodate them without polluting shared workflows.
* Good, because standardized building blocks remain shared, ensuring consistency where it matters.
* Bad, because pipeline definitions are duplicated across repositories, creating the possibility of drift (mitigated by agent-assisted cross-repo PRs).
