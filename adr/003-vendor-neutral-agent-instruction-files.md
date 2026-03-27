# ADR 003: Vendor-Neutral Agent Instruction Files

| Status  | Proposed   |
|---------|------------|
| Date    | 2026-03-26 |

## Context and Problem Statement

Platform Mesh is a multi-repo organization with dozens of separate repositories (Go operators, Node.js applications, Helm charts, shared libraries, infrastructure code). Team members and external contributors use different AI coding agents (Claude Code, GitHub Copilot, Cursor, Windsurf, and others) to assist with development across these repositories. We cannot dictate which AI tool contributors use, yet we want every contributor — regardless of their tool choice — to benefit from project-specific guidance. Each tool has its own proprietary configuration file for project-level instructions:

| Tool | Config File |
|------|------------|
| Claude Code | `CLAUDE.md` |
| Cursor | `.cursorrules` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Windsurf | `.windsurfrules` |

No repositories currently commit agent instruction files. Without a checked-in instruction file, contributors' AI tools have no project-specific guidance — they don't know the build system, code conventions, or architectural patterns. If we were to add instruction files, choosing a tool-specific format (e.g., `CLAUDE.md` for Claude Code, `.cursorrules` for Cursor) would lock that knowledge into a single tool's ecosystem. Contributors using other tools would either get no guidance or we'd need to maintain duplicate files per tool in every repository.

We need a single, vendor-neutral file that all AI coding agents can read, reducing maintenance burden across the organization and ensuring every contributor receives consistent guidance regardless of which tool they use.

## Decision Drivers

* **Vendor neutrality**: No lock-in to a single AI tool vendor. Contributors should be free to choose their preferred coding assistant.
* **Contributor experience**: External contributors and new team members should get project-specific AI guidance out of the box, without needing to use a specific tool or set up additional configuration.
* **Maintenance burden**: Maintaining multiple config files with overlapping content is error-prone and creates drift.
* **Broad tool support**: The chosen format must be supported by the major AI coding tools used in the team today and likely to be adopted by future tools.
* **Multi-repo consistency**: The chosen format must work identically across all repositories in the organization, providing a uniform developer experience.
* **Simplicity**: The format should be plain Markdown with no proprietary schema, making it easy to author and review.
* **Per-repository flexibility**: Must support a root-level file per repository, with optional per-directory overrides for repositories that contain multiple projects (e.g., `helm-charts/`).

## Considered Options

1. **Do not commit any agent instruction files** — continue without checked-in agent guidance; contributors configure their tools locally.
2. **Adopt `AGENTS.md` as the primary instruction file** — use the vendor-neutral standard to provide project-specific guidance to all AI coding tools.
3. **Commit tool-specific files per repository** — add `CLAUDE.md`, `.cursorrules`, `.github/copilot-instructions.md`, etc. as needed for each tool.

## Decision Outcome

Chosen option: **"Adopt `AGENTS.md` as the primary instruction file"**, because it is the only option that is natively supported across all major AI coding tools, is backed by an open standard under the Linux Foundation, and minimizes duplication while allowing tool-specific overrides where necessary.

### Implementation

1. **Create `AGENTS.md` at the root of each repository** containing agent-specific context: build commands, code conventions, testing patterns, and architectural guidance. The language should be terse and imperative — optimized for AI tool consumption, not human readability.
2. **Create per-directory `AGENTS.md` files** where needed within repositories that contain multiple projects (e.g., individual chart directories in `helm-charts/`).
3. **Instruct agents to consume `CONTRIBUTING.md`** — each `AGENTS.md` file should reference the repository's `CONTRIBUTING.md` and instruct agents to follow the contributing guidelines in addition to the agent-specific instructions. This keeps human-facing guidance (contribution workflow, review process, code of conduct) in `CONTRIBUTING.md` and agent-facing guidance (terse commands, hard boundaries, conventions) in `AGENTS.md`.
4. **Do not commit vendor-specific instruction files** (`CLAUDE.md`, `.cursorrules`, `.windsurfrules`, `.github/copilot-instructions.md`). All agent guidance should go through `AGENTS.md`.
5. **Content guidelines for `AGENTS.md`**:
   - Keep it concise, terse, and human-authored. Use imperative language. Research shows that LLM-generated instruction files reduce agent performance.
   - Focus on judgment calls, architecture, and boundaries — move enforceable rules to linter configs and toolchain settings.
   - Use plain Markdown with no required schema. Any headings and structure that aids comprehension.
   - Include: build/test commands, code conventions, project structure, git conventions, and hard boundaries (things agents must never do).
   - Exclude: information already in `CONTRIBUTING.md` or `README.md`, enforceable linting rules, verbose documentation that belongs in docs/.
6. **Ensure `CONTRIBUTING.md` exists for each active repository** — human-facing contribution guidelines (workflow, review process, style guidance in prose form) should live there, not in `AGENTS.md`.

### Consequences

* Good, because a single `AGENTS.md` file is natively read by Claude Code, GitHub Copilot, Cursor, Windsurf, Continue.dev, and Aider — all tools used or likely to be used by the team.
* Good, because `AGENTS.md` is an open standard under the Agentic AI Foundation (Linux Foundation), reducing risk of the format being abandoned or controlled by a single vendor.
* Good, because maintenance effort is reduced — one file per repository instead of N tool-specific files.
* Good, because new team members and external contributors using any AI tool immediately get project-specific guidance without additional setup.
* Good, because the format is plain Markdown with no proprietary schema, making it reviewable in standard code review workflows.
* Good, because per-directory `AGENTS.md` files support repositories with multiple projects.
* Good, because separating agent instructions (`AGENTS.md`) from human contribution guidelines (`CONTRIBUTING.md`) allows distinct language and detail levels for each audience — terse and imperative for agents, descriptive and contextual for humans.
* Neutral, because no existing committed files need migration — this is a greenfield adoption.
* Neutral, because repositories will need `CONTRIBUTING.md` files alongside `AGENTS.md` — but contributing guidelines are valuable independently of this decision.
* Neutral, because adoption of `AGENTS.md` by future AI tools is expected but not guaranteed — however, the format is simple enough that any tool can parse it with minimal effort.

## Pros and Cons of the Options

### Option 1: Do not commit any agent instruction files

Keep the status quo — no agent instruction files are checked into repositories. Contributors configure their AI tools locally or rely on README files and documentation.

* Good, because there is zero maintenance overhead for agent-specific files.
* Good, because no decision on file format is needed.
* Bad, because AI tools have no project-specific guidance — they don't know the build system, conventions, or architectural patterns.
* Bad, because each contributor must manually configure their tool or work without guidance, leading to inconsistent AI-assisted contributions.
* Bad, because project knowledge that could improve AI tool output remains undocumented in a machine-readable form.

### Option 2: Adopt `AGENTS.md` as the primary instruction file

Use the vendor-neutral `AGENTS.md` standard for all shared instructions.

* Good, because it is natively supported by all major AI coding tools (Claude Code, Copilot, Cursor, Windsurf, Continue.dev, Aider).
* Good, because it is backed by the Linux Foundation as an open standard.
* Good, because the format is simple Markdown — no schema to learn, no tooling required.
* Good, because over 60,000 open source projects have adopted it, indicating strong ecosystem momentum.
* Good, because it reduces maintenance to a single primary file per repository.
* Bad, because authoring `AGENTS.md` files for each repository requires initial effort to document conventions and build commands.

### Option 3: Commit tool-specific files per repository

Add separate instruction files for each AI coding tool to each repository.

* Good, because each file can use tool-specific features and syntax.
* Bad, because content drifts between files as updates are made to one but not others.
* Bad, because the maintenance burden scales linearly with the number of tools multiplied by the number of repositories.
* Bad, because contributors using unsupported tools get no project guidance.
* Bad, because it creates implicit vendor lock-in — the tool with the most maintained config file becomes the de facto standard.

## More Information

### About AGENTS.md

[AGENTS.md](https://agents.md) was created by OpenAI in 2025 and donated to the Agentic AI Foundation (AAIF) under the Linux Foundation in December 2025. It provides a unified, vendor-neutral way for AI coding agents to understand project setup, conventions, and workflows. The format is intentionally simple — plain Markdown with no required fields or schema.

### Rollout Plan

1. Ensure each active repository has a `CONTRIBUTING.md` with human-facing contribution guidelines (workflow, review process, style guidance).
2. Draft an `AGENTS.md` for each active repository containing terse, agent-specific build commands, conventions, and architectural guidance. Include a directive for agents to also read `CONTRIBUTING.md`.
3. Add per-directory `AGENTS.md` files where repositories contain multiple projects (e.g., `helm-charts/`).
4. Remove any tool-specific instruction files (`.cursorrules`, `.windsurfrules`, `.github/copilot-instructions.md`) if they exist, since `AGENTS.md` supersedes them.
5. Update contributing guidelines to reference `AGENTS.md` as the standard for agent instructions across all repositories.

### Related Resources

* [AGENTS.md Official Site](https://agents.md)
* [AGENTS.md GitHub Repository](https://github.com/agentsmd/agents.md)
* [AGENTS.md Specification](https://asdlc.io/practices/agents-md-spec/)
* [MADR ADR Template](https://github.com/adr/madr/blob/develop/template/adr-template.md)
