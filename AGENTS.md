## Repository Description

- `architecture` is the documentation repository for [Platform Mesh](https://platform-mesh.io). It holds Architecture Decision Records (ADRs) and Requests for Comments (RFCs) governed by the Platform Mesh Technical Steering Committee (TSC).
- ADRs follow the [MADR](https://github.com/adr/madr) template and capture significant technical decisions. RFCs propose larger features or cross-cutting changes prior to implementation; a single RFC can spawn several ADRs.
- This repo is prose only. There is no application code, no build system, and no test suite.
- Read the org-wide [AGENTS.md](https://github.com/platform-mesh/.github/blob/main/AGENTS.md) for general conventions that apply across all Platform Mesh repos.

## Core Principles

- Architecture documents are human-authored. Use AI tools to research, summarize prior art, polish prose, or check links — do not generate full ADRs or RFCs end-to-end. The intent and language must come from an author who can defend the decision in TSC review (see [ADR 003](adr/003-vendor-neutral-agent-instruction-files.md)).
- Keep ADRs decision-focused and RFCs proposal-focused. Do not mix the two formats in one document.
- Treat merged ADRs as historical record. Do not rewrite a merged decision; supersede it with a new ADR instead.
- Reference rather than duplicate. If a topic is owned by another doc (`README.md`, `CONTRIBUTING.md`, an existing ADR), link to it.

## Project Structure

- `adr/` — Architecture Decision Records. One file per decision, named `NNN-short-name.md`.
- `adr/assets/` — diagrams and images referenced by ADRs.
- `rfc/` — Requests for Comments. One file per proposal, named `NNN-short-name.md`.
- `rfc/assets/` — diagrams and images referenced by RFCs.
- `LICENSES/` — third-party / SPDX license texts.
- `CODEOWNERS` — assigns all documents to `@platform-mesh/tsc`.
- `README.md`, `CONTRIBUTING.md` — repository overview and human-facing contribution flow.

## Authoring Conventions

- New ADRs and RFCs use the next free zero-padded sequence number. Do not reuse numbers from abandoned drafts and do not renumber existing files.
- Filename: number, separator, short descriptive slug, `.md`. The existing tree uses both `-` and `_` as the separator after the number; match the convention of neighbouring files in the same directory rather than enforcing a rewrite.
- ADRs follow the MADR template and start with a status table containing at minimum `Status` and `Date`.
- Status starts as `Proposed`. The TSC moves it to `Approved`, `Rejected`, or `Superseded` as part of merge or follow-up review.
- One decision per ADR. Split unrelated decisions into separate ADRs.
- Store new diagrams under the relevant `assets/` directory. Prefer Mermaid sources when practical; if a rendered image is committed, keep its source alongside it.
- Cross-link related ADRs and RFCs by relative path.
- PR titles for new records use the form `ADR NNN: <title>` or `RFC NNN: <title>` (no Conventional Commits prefix). Use Conventional Commits prefixes (`docs:`, `chore:`, `feat:`, `fix:`) for repo maintenance PRs that do not introduce a new record.

## Commands

- This repo has no `Taskfile`, `Makefile`, or build steps.
- `git log --oneline -- adr/ rfc/` — quick history of decisions and proposals.
- `gh issue view <n>` / `gh pr view <n>` — fetch issue or PR context (this repo's PRs and issues live in `platform-mesh/architecture`).
- Mermaid in fenced ` ```mermaid ` blocks renders on GitHub; no local toolchain required for preview.

## Do Not

- Auto-generate ADRs or RFCs with AI tools and submit them as-is.
- Edit a merged ADR's decision content. Open a new ADR that supersedes the previous one.
- Renumber existing ADRs or RFCs, or reuse a number from an abandoned draft.
- Add tool-specific instruction files (`.cursorrules`, `.windsurfrules`, `.github/copilot-instructions.md`). `CLAUDE.md` exists only as a one-line `@AGENTS.md` import so [Claude Code](https://code.claude.com/docs/en/memory) loads this file; do not duplicate AGENTS.md content into it.
- Move or rename existing ADR/RFC files — external links and downstream tooling reference these paths.

## Hard Boundaries

- Never merge ADRs or RFCs without TSC review. The TSC (`@platform-mesh/tsc`) are the [code owners](CODEOWNERS) and the only approvers for ADRs.
- Do not change ADR `Status` values on behalf of the TSC.
- Ask before introducing new top-level directories or restructuring `adr/` or `rfc/`. The org-wide AGENTS.md and other repos link into these paths.

## Human-Facing Guidance

- See [`README.md`](README.md) for the repository overview and the steps to create a new ADR or RFC.
- See [`CONTRIBUTING.md`](CONTRIBUTING.md) for the contribution process, DCO requirements, and review flow.
- See [ADR 003](adr/003-vendor-neutral-agent-instruction-files.md) for the rationale behind this file.
