# ADR 009: Repository Tiers and `contrib-` Prefix for Best-Effort Code

| Status  | Proposed   |
|---------|------------|
| Date    | 2026-05-28 |

## Context and Problem Statement

The Platform Mesh organization hosts a mix of public, private, and internal repositories. Some are production components with releases, CI, and security guarantees; others are examples, proofs of concept, demos, samples, or experimental wrappers maintained on a best-effort basis. From the outside there is no reliable way to tell which is which — naming is inconsistent (`examples`, `example-httpbin-operator`, `poc-kcp-observability`, `samples-opendesk-ocm-landscaper`, `demo-collab-hub-app`, `httpbin-ui`), and visibility is split across `PUBLIC`, `PRIVATE`, and `INTERNAL` without a stated rule for why.

This creates two problems:

1. **Consumers cannot calibrate expectations.** A user discovering `example-httpbin-operator` has no signal that it lacks the release process, security backports, and review standards of a core component like [account-operator](https://github.com/platform-mesh/account-operator). Issues filed against best-effort repos compete for attention with issues against supported ones.
2. **The organization is not transparently OSS-first.** Platform Mesh is an open source project. Code that exists but is hidden in private/internal repos undermines that posture without a clear reason. Contributors and downstream adopters cannot find or fork what they cannot see.

## Decision

Adopt two public repository tiers with a naming convention that makes the tier visible from the repo name alone, and treat non-public visibility as an exception requiring justification.

### Tier 1: Supported (no prefix)

Repositories without a prefix are **officially supported** by Platform Mesh. They must:

* Be `PUBLIC`.
* Follow the shared CI workflow strategy ([ADR 004](004-shared-ci-workflow-strategy.md)) and test coverage thresholds ([ADR 005](005-default-test-coverage-thresholds.md)).
* Emit SBOMs and OCM components ([ADR 001](001-sbom-generation-and-ocm-component-restructuring.md)).
* Follow the security process ([RFC 007](../rfc/007_security-process.md)) — including coordinated vulnerability handling and security backports for supported branches.
* Have a `CONTRIBUTING.md` and `AGENTS.md` ([ADR 003](003-vendor-neutral-agent-instruction-files.md)).
* Have a documented release process and accept issues/PRs through the normal review path.

Examples: `account-operator`, `kubernetes-graphql-gateway`, `iam-service`, `portal`, `helm-charts`.

### Tier 2: Contrib (`contrib-` prefix)

Repositories prefixed with `contrib-` are **open code with best-effort support only**. They are public so others can read, learn from, fork, and contribute, but Platform Mesh makes **no** guarantees about:

* Release cadence or versioning stability.
* Security review or vulnerability backports.
* Issue or PR response times.
* Compatibility with current supported components.

`contrib-*` repos must still be `PUBLIC`, carry a license, and include a `README.md` that states the best-effort status explicitly. They are exempt from the CI, coverage, SBOM, and release requirements of Tier 1.

Use this tier for: examples, samples, demos, proofs of concept, experimental operators, reference implementations, integration sketches, and any work that is valuable to share but that the org is not committing to maintain.

**Official forks of upstream projects also live under `contrib-`** (e.g. `contrib-<upstream-name>`). A fork is by definition not a project Platform Mesh owns end-to-end, and the support tier should reflect that — even when the fork is actively used internally. If a fork eventually becomes a hard runtime dependency that the org commits to maintaining, it can be promoted (see below) or upstreamed.

Suggested names: `contrib-examples`, `contrib-httpbin-operator`, `contrib-mongodb-multiclusterruntime`, `contrib-kcp-observability`, `contrib-opendesk-ocm-landscaper`.

### Promotion and demotion

**Promotion (`contrib-*` → Tier 1)** is allowed and expected. A `contrib-*` repo can be promoted when it has a committed owner, meets the Tier 1 requirements above (CI, coverage, SBOM, security process, release process), and the TSC approves the change. Promotion is a rename — drop the `contrib-` prefix — plus the standard GitHub redirect. Downstream Go module consumers update their import path once.

**Demotion (Tier 1 → `contrib-*`) requires a full deprecation policy, which is TBD.** Demotion is a stronger signal than promotion: it tells existing users that guarantees they relied on are being withdrawn. Until that policy is written, do not demote a Tier 1 repo — archive it instead, or keep maintaining it. The deprecation policy should at minimum cover: advance notice window, communication channels, handling of open issues/PRs, security-fix expectations during the deprecation window, and whether existing release branches retain support after demotion.

### Visibility: public by default

`PRIVATE` and `INTERNAL` repositories are discouraged. A non-public repo is permitted only when at least one of the following applies, and the reason should be recorded in the repo description:

* Contains secrets, credentials, or customer data that cannot be redacted.
* Holds security-sensitive work-in-progress under embargo (pre-disclosure).
* Is a transient staging area for a repo that will be made public on a known timeline.

"It's not ready yet" is not a justification — that is what `contrib-*` exists for. "It's internal tooling" is not a justification — publish it as `contrib-*` if it has any external value, or archive it if it does not.

### Migration

This ADR does not require an immediate rename of every existing repo. Apply the rules as follows:

1. **New repositories** follow the rules from the date this ADR is approved.
2. **Existing public repos with ad-hoc prefixes** (`example-*`, `poc-*`, `samples-*`, `demo-*`) should be renamed to `contrib-*` opportunistically — when a repo is next touched, or in a batch by the TSC. GitHub redirects old URLs, so the cost is low.
3. **Existing private/internal repos** should be reviewed by the TSC against the visibility exceptions above. Repos without a valid exception should be made public (as Tier 1 or `contrib-*`) or archived.
4. **Repos that no longer have an owner** should be archived rather than left in an ambiguous state.

## Consequences

* Good, because consumers can tell support level from the repo name alone, before clicking through.
* Good, because the org's OSS-first posture becomes structurally enforced rather than aspirational.
* Good, because best-effort work has a legitimate home — contributors can publish experiments without inheriting the obligations of a Tier 1 repo.
* Good, because issues and PRs against `contrib-*` repos can be triaged with explicitly different SLAs without surprising the reporter.
* Good, because the `contrib-` prefix sorts and filters cleanly in GitHub's UI and `gh` listings.
* Neutral, because graduating a `contrib-*` repo to Tier 1 requires a rename — but rename is cheap and the explicit promotion is a feature, not a bug.
* Neutral, because demotion is intentionally blocked until a deprecation policy exists — this prevents accidental withdrawal of guarantees but means a struggling Tier 1 repo has only two interim options (keep maintaining, or archive).
* Bad, because existing repo URLs and `go get` paths will need redirects when renamed. GitHub handles the HTTP redirect; Go module consumers will need a one-time path update.
* Bad, because the prefix adds visual noise to repo names.

## References

* [ADR 001: SBOM Generation and OCM Component Restructuring](001-sbom-generation-and-ocm-component-restructuring.md)
* [ADR 003: Vendor-Neutral Agent Instruction Files](003-vendor-neutral-agent-instruction-files.md)
* [ADR 004: Shared CI Workflow Strategy](004-shared-ci-workflow-strategy.md)
* [ADR 005: Default Test Coverage Thresholds](005-default-test-coverage-thresholds.md)
* [RFC 007: Platform Mesh Security Process](../rfc/007_security-process.md)
