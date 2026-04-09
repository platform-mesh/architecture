# RFC 007: Security Process for Platform Mesh GitHub Organization

| Status  | Proposed   |
|---------|------------|
| Author  | @xmudrii   |
| Created | 2026-04-06 |
| Updated | 2026-04-08 |

## Context and Problem Statement

Platform Mesh hosts ~30 public repositories (Go operators, Helm charts, custom Docker images, TypeScript code) under the [platform-mesh](https://github.com/platform-mesh) GitHub organization. As a public open-source project, it faces two distinct categories of security risk: vulnerabilities in its own code that external researchers or users may discover, and vulnerabilities in the hundreds of third-party dependencies and container base images that its repositories pull in.

The project already has some security practices in place — Renovate is used in some repositories for dependency updates, and a `SECURITY.md` is defined at the org level with reporting channels (GitHub Security Advisories and a security mailing list). What is missing is a unified process that ties these pieces together: org-wide tooling coverage, a defined triage and remediation workflow, and a consistent path from detection through disclosure for both vulnerability channels.

This RFC proposes tooling for automated vulnerability detection and remediation, and a unified vulnerability management process that brings both channels — external reports and automated dependency findings — under a single triage and response workflow.

## Decision Drivers

* **Org-wide coverage**: The solution must work across all ~30 repositories with minimal per-repo effort.
* **Automated remediation**: Vulnerabilities with known fixes should result in automated pull requests, not manual dependency bumps.
* **Notification and tracking**: Maintainers must be notified when a CVE is discovered, and remediation must be tracked via GitHub issues or PRs.
* **Low noise**: The process must not flood maintainers with PRs or alerts for low-severity or non-actionable findings.
* **Minimal tooling overhead**: Prefer tools that are free for open-source projects and require minimal infrastructure.
* **Alignment with OSS best practices**: Follow patterns established by CNCF projects (e.g. Kubernetes) and other mature open-source organizations.
* **Multi-branch support**: The solution must cover not only the main branch but also supported release branches (current-N, to be defined in a dedicated proposal).

## Vulnerability Management Process

Vulnerabilities reach Platform Mesh through two channels: external reports from security researchers and users about issues in Platform Mesh's own code, and automated findings about vulnerabilities in third-party dependencies and container base images. Both channels feed into a single process managed by the security response team.

### Security response team

A security response team (`platform-mesh/security` GitHub team) is responsible for all vulnerability-related activity across both channels. Responsibilities include: receiving and acknowledging external reports, reviewing automated findings, coordinating fixes, publishing advisories, and communicating with reporters and users. Team members should be listed in `SECURITY.md`.

### Establishing the announcements mailing a list

A dedicated subscription based mailing list (e.g. `platform-mesh-announcements@lists.neonephos.org`) should be established for notifiying users about vulnerabilities in Platform Mesh. This mailing lists is intentionally separate from the security mailing list for technical reasons — everyone can send emails to the security mailing lists (`platform-mesh-security@lists.neonephos.org`), but only the security response team can receive and read emails. In the case of the announcement mailing lists, it should be the other way around — Platform Mesh maintainers can send emails, but anyone subscribed can receive and read.

### Intake: external reports

The project's `SECURITY.md` already defines two reporting channels: GitHub Security Advisories and a dedicated security mailing list (`platform-mesh-security@lists.neonephos.org`). Private vulnerability reporting should be enabled org-wide so that the Security Advisories intake works on every repository. GitHub Security Advisories serve as the single source of truth for coordinating all externally reported vulnerabilities — when a report arrives via email, the security response team should immediately create a draft security advisory on the affected repository so that triage, discussion, fix development, and disclosure are tracked in one place regardless of how the report was received.

Reports received through either channel are triaged by the security response team. The reporter should receive an acknowledgment and be kept informed about progress until the issue is resolved or closed.

Dependency scanner results without proof of exploitability in Platform Mesh should not be treated as confirmed vulnerabilities — the team should evaluate whether the affected code path is reachable. This policy applies equally to external reports that consist solely of scanner output and to automated findings from internal tooling.

### Intake: automated tooling findings

Vulnerabilities in third-party dependencies and container images are surfaced by automated scanning tools. The specific tooling is described later in this RFC, but regardless of which tools are adopted, the security response team must monitor findings from all of them. Automated findings typically arrive as pull requests with proposed fixes or alerts in the GitHub Security tab. The appropriate security team (`platform-mesh/security` GitHub team) will be automatically tagged on each automatically created security PR as a way to inform and notify them of a security issue and its fix. GitHub Security will be configured in a similar way to allow access to and notify members of the `platform-mesh/security` GitHub team.

Scanning should cover both the main branch and supported release branches, so vulnerabilities are detected regardless of which branch they affect.

### Triage

Every vulnerability — whether reported by a human or flagged by a tool — enters the same triage process. A member of the security response team evaluates the finding and assigns a severity:

* **Critical / High** — immediate action. For external reports about Platform Mesh's own code, this may involve using GitHub's private fork feature within a draft security advisory to develop the fix confidentially.
* **Medium / Low** — scheduled for the next sprint. The finding is tracked as a GitHub issue labeled `security`.
* **Informational / False positive** — closed with a comment explaining the rationale. For external reports, the reporter is notified of the decision.

For automated findings, triage includes verifying whether the vulnerability is actually reachable in Platform Mesh's usage of the affected dependency. A CVE in a transitive dependency whose vulnerable code path is never invoked does not warrant the same urgency as one in a directly used function. For that purpose, the project should be publishing [VEX manifests](https://cyclonedx.org/capabilities/vex/), but that is out of scope of this proposal.

### Remediation

The remediation path depends on the source of the vulnerability:

**For vulnerabilities in Platform Mesh's own code:** The security team develops a fix privately using the GitHub Private Security Forks feature, applies it to the main branch, and backports it to all affected supported release branches. The fix is reviewed and tested before disclosure.

**For third-party dependency CVEs:** When automated tooling opens a fix PR, the security response team reviews it, verifies that the updated dependency does not introduce breaking changes, and merges it. When no automated PR is available, the team updates the dependency manually.

**For container image CVEs:** If the vulnerability is in the base image and a newer tag is available, the base image reference is updated. If the fix requires a patched package within the same image tag, the team triggers a rebuild. If no fix is available upstream, the team documents the finding and monitors for a fix.

**For security posture regressions (e.g. branch protection disabled, token permissions too broad):** The team addresses the configuration issue directly, guided by the finding's remediation steps.

### Verification

After a fix is merged, the next automated scan should confirm the vulnerability is resolved. If it persists, the team investigates further. For fixes that are not covered by automated scanning (e.g. manual configuration changes), the team verifies the fix manually.

### Disclosure and user notification

The disclosure process differs based on the vulnerability source:

**For vulnerabilities in Platform Mesh's own code:** Public disclosure happens through GitHub Security Advisories. A draft advisory is created on the affected repository upon receiving a report. This allows development of the fix using Private Security Forks feature. Users are notified via the announcement mailing list about pending security releases and their target release dates, once the target release dates are known (without any information about vulnerability and its severity). Given the fix is ready, releases and the advisory are published on the target date. The advisory must contain: description, affected versions, patched versions, severity (CVSS), and CVE identifier (GitHub can assign CVEs as a CNA). Users are again notified that the security releases have been released via the mailing list. The fix is highlighted in  release notes under a dedicated "Security Fixes" section. The advisory automatically feeds into the GitHub Advisory Database, so downstream consumers of libraries are also notified via GitHub.

In case of critical and high severity vulnerabilies, the advisory may be published up to 2 working days after the release, to give users more time to upgrade and secure their environements before details are published publicly.

**For third-party dependency and container image fixes:** These do not typically require a security advisory, since the upstream project owns the CVE. The fix is included in the next release and noted in release notes. PRs labeled `security` can be used to automatically group security changes in release notes using tooling such as the [Kubernetes release-notes tool](https://github.com/kubernetes/release/tree/master/cmd/release-notes). In case of an expoitable critical or high severity vulnerability in a third-party dependency, users should be notified via the announcement mailing list with a target release date for releases that will include the updated dependency.

### Vulnerabilities in previously released versions

The process and tools described in this proposals must also be applied to all supported release branches. The number of previously released versions that receive security fixes is out of scope for this RFC and should be addressed in a dedicated proposal covering the release and support lifecycle. The tooling described here supports multiple branches and can be extended to cover whatever support window is agreed upon.

### SECURITY.md

The existing `SECURITY.md` in the `platform-mesh/.github` repository should be updated to reflect the process defined in this RFC, especially about the notification process.

## Technical Implementation

To implement the vulnerability management process described above, this RFC proposes the following tooling. The tools were selected to match the process requirements: automated detection and fix PRs for dependency CVEs, container image scanning for OS-level vulnerabilities, and continuous security posture monitoring. The tooling consists of three options that can be adopted independently or together, depending on the team's capacity and priorities:

1. **Renovate Bot** — dependency vulnerability detection and automated fix PRs. This is the highest-value option and the recommended starting point.
2. **Trivy** — container image scanning for OS-level vulnerabilities via GitHub Actions. Adds value primarily for repos that build images with non-minimal base images.
3. **OpenSSF Scorecard** — continuous security posture assessment and best practice verification via GitHub Actions. A lightweight addition that verifies the overall security hygiene of each repository.

Adopting all three provides the broadest coverage, but each option is self-contained. The team can start with Renovate alone and layer in Trivy and Scorecard as time permits.

### Consequences

* Good, because dependency vulnerabilities are automatically remediated via grouped PRs with minimal noise.
* Good, because container images are scanned for OS-level CVEs that dependency tools cannot detect.
* Good, because Scorecard continuously monitors repository security practices and surfaces regressions in the GitHub Security tab.
* Good, because maintainers are notified through GitHub's existing mechanisms (PR assignments, issue notifications, Slack integration).
* Good, because the configuration can be shared across all repos via a central config in the `platform-mesh/.github` repository.
* Good, because all three tools are free for open-source projects and require no self-hosted infrastructure.
* Good, because Renovate reads from two vulnerability data sources (GitHub Advisory Database + OSV), providing broad coverage including Go-specific advisories.
* Good, because Scorecard provides a public badge and score that demonstrates the project's security posture to consumers and stakeholders.
* Bad, because Renovate requires initial configuration and onboarding PRs across all repos.
* Bad, because Trivy and Scorecard each require a GitHub Actions workflow to be created in each repository.

## Option 1: Renovate Bot for Dependency Vulnerability Remediation

[Renovate](https://github.com/renovatebot/renovate) is an open-source dependency automation tool by Mend.io, available as a free hosted GitHub App. It scans dependency files across all supported package managers (Go modules, npm, Docker, Helm, GitHub Actions, and 90+ others), detects outdated or vulnerable dependencies, and creates pull requests to update them.

For Platform Mesh, Renovate can be configured in **security-only mode**: all regular dependency updates are disabled, and PRs are created only when a known CVE affects a dependency. This keeps repos quiet during normal operation and generates immediate, actionable PRs when a vulnerability is published. Renovate is installed as a GitHub App at the org level, with a shared configuration in `platform-mesh/.github` that each repo extends from with a one-line `renovate.json`.

Renovate uses two independent vulnerability data sources:

* **`vulnerabilityAlerts` (GitHub-specific):** Reads Dependabot alerts via GitHub's GraphQL API. Covers both direct and transitive dependencies because GitHub's dependency graph resolves the full tree. Requires the Dependency Graph and Dependabot Alerts to be enabled in org settings (not Dependabot security updates — the PR-creating bot is replaced by Renovate).
* **`osvVulnerabilityAlerts` (platform-independent):** Queries the OSV database (osv.dev), which aggregates data from the Go Vulnerability Database, GitHub advisories, PyPA, RustSec, and others. Particularly valuable for a Go-heavy organization where Go-specific advisories may appear in OSV before GitHub's advisory database.

When both sources flag the same CVE, Renovate deduplicates at the PR level — only one PR is created, targeting the highest fixed version. Security PRs use a `force` mechanism that overrides all other configuration (schedules, enabled/disabled state, PR limits), ensuring they are never blocked.

### What Renovate covers

* Go modules (`go.mod`, `go.sum`)
* npm/yarn/pnpm packages (`package.json`, lockfiles)
* Docker base images (`FROM` lines in Dockerfiles)
* Helm chart dependencies (`Chart.yaml`, `Chart.lock`)
* GitHub Actions versions (`.github/workflows/*.yml`)

### What Renovate does NOT cover

* OS-level packages inside built container images (e.g. `openssl`, `glibc`) — this is Trivy's job.
* Source code vulnerabilities (SAST) — no code analysis is performed.
* Repository security configuration (branch protection, token permissions) — this is Scorecard's job.

## Option 2: Trivy for Container Image Scanning

[Trivy](https://github.com/aquasecurity/trivy) is an open-source vulnerability scanner that can scan container images for known CVEs in OS-level packages and application dependencies. It runs as a GitHub Actions workflow, scans built images, and uploads results to the GitHub Security tab via SARIF format.

Trivy fills the gap that Renovate cannot cover. Renovate sees your `Dockerfile` and can bump the base image tag, but once `docker build` runs, the final image contains OS packages (`openssl`, `glibc`, `ca-certificates`, etc.) that are not declared in any manifest file Renovate reads. If a CVE is published against one of those packages, only a container image scanner can detect it.

The Trivy workflow should be implemented as a reusable workflow in `platform-mesh/.github` and called by repos that build container images. Repos using `scratch` or `distroless` base images have minimal OS-level attack surface, so Trivy adds limited value there — it should be prioritized for repos using `alpine`, `ubuntu`, or similar base images.

### What Trivy covers

* OS-level package vulnerabilities in built container images (Alpine apk, Debian apt, etc.)
* Application dependency vulnerabilities embedded in the image (as a secondary check)

### What Trivy does NOT cover

* Dependency files in source repos — that is Renovate's job.
* Repository security configuration — that is Scorecard's job.
* It does not create fix PRs — it surfaces findings in the Security tab. Remediation typically involves updating the base image, which Renovate can handle when a newer image tag is available. When the fix is a patched package within the same image tag, a rebuild is needed.

## Option 3: OpenSSF Scorecard for Security Posture Verification

[OpenSSF Scorecard](https://scorecard.dev/) is an automated tool from the Open Source Security Foundation that assesses repositories against a set of security best practice checks and assigns each a score of 0–10. Unlike Renovate and Trivy which find specific vulnerabilities, Scorecard evaluates the project's **security posture** — whether the repository is configured in a way that prevents vulnerabilities from being introduced in the first place.

Scorecard checks include (among others):

* **Branch protection** — is the default branch protected with required reviews and status checks?
* **Pinned dependencies** — are dependencies (including GitHub Actions) pinned to specific versions or SHAs?
* **Code review** — are changes reviewed before merging?
* **CI tests** — does the project run tests in CI?
* **SAST** — is a static analysis tool (e.g. CodeQL) configured?
* **Vulnerabilities** — are there known unfixed vulnerabilities?
* **Security policy** — does the repo have a `SECURITY.md`?
* **Dependency update tool** — is a dependency update tool (e.g. Renovate) configured?
* **Token permissions** — are workflow tokens scoped to least privilege?
* **Dangerous workflow** — does the project avoid dangerous CI patterns like `pull_request_target` with code checkout?

The Scorecard GitHub Action runs on every push to the default branch, uploads results to the GitHub Security tab via SARIF, and optionally publishes the score to the OpenSSF API for a public badge. Results include severity ratings and actionable remediation steps.

Scorecard is configured **per repository** — there is no org-level toggle. Each repository needs a workflow file that runs the Scorecard action. To minimize duplication, the workflow can be defined once as a reusable workflow in `platform-mesh/.github` and called from each repo with a minimal caller workflow. Additionally, the [scorecard-monitor](https://github.com/ossf/scorecard-monitor) tool can be set up in a single repository to track and compare scores across all repos in the organization.

### What Scorecard covers

* Repository configuration and security practices (branch protection, token permissions, dangerous workflows)
* Whether security tooling is in place (SAST, dependency update tools, security policy)
* Public security score and badge for the project

### What Scorecard does NOT cover

* Specific CVEs in dependencies or container images — it does not scan code or artifacts.
* It does not create fix PRs — it surfaces findings in the Security tab with remediation guidance.

## Alternatives Considered

### GitHub Dependabot

Dependabot is GitHub's built-in dependency update and vulnerability alert tool. Dependabot alerts (the detection side) are still used — they are enabled in org settings so that Renovate can read the vulnerability data via GitHub's GraphQL API. However, Dependabot security updates (the PR-creating bot) are disabled because Renovate replaces it.

Renovate was chosen over Dependabot for the following reasons:

* Dependabot requires a `dependabot.yml` configuration file per repository for granular control over grouping, scheduling, and ecosystem-specific rules. Renovate supports a single shared configuration in `platform-mesh/.github` that all repos extend from, reducing maintenance overhead across 30 repos.
* Dependabot has no security-only mode without version updates — achieving this requires setting `open-pull-requests-limit: 0` per ecosystem as a workaround. Renovate supports this natively by disabling all regular updates while keeping vulnerability alerts enabled via a `force` override.
* Renovate's Dependency Dashboard provides a per-repo issue listing all pending updates with the ability to approve major updates via checkbox. While GitHub's org-level Security Overview provides a cross-repo vulnerability dashboard (available on Team and Enterprise plans), it does not offer this per-repo approval workflow.
* Renovate additionally reads the OSV database, providing a second independent vulnerability data source that includes Go-specific advisories not yet in GitHub's advisory database.

### LFX Security

[LFX Security](https://lfx.linuxfoundation.org/tools/security/) is a free security scanning platform from the Linux Foundation, powered by Snyk.

LFX Security was not chosen for the following reasons:

* Scans run on a weekly cadence, which means newly published CVEs may take up to a week to surface — too slow for critical vulnerabilities that require immediate attention.
* The platform's user experience presents challenges: page load times are noticeably slow, the interface can feel unresponsive during navigation, and the overall dashboard usability lags behind GitHub-native tooling that the team already uses daily.
* It is a monitoring dashboard only — it does not create PRs or GitHub issues automatically, making it an additive layer rather than a standalone solution.
* Onboarding requires registration with the LFX platform, separate from GitHub, and notifications are dashboard-based and email-based rather than GitHub-native.

LFX Security could be reconsidered in the future if the project needs aggregated cross-repo dashboards or license compliance scanning.

### Snyk (Free Tier)

[Snyk](https://snyk.io/) is a commercial developer security platform with a free tier for open-source projects, covering SCA, SAST, container scanning, and IaC scanning.

Snyk was not chosen for the following reasons:

* It is a commercial tool whose free tier scope has been reduced several times, creating uncertainty about long-term availability.
* It does not handle dependency version bumps as persistently as Renovate — it offers one-time fix suggestions rather than continuous monitoring with grouped PRs.
* It adds an external service dependency and requires granting repo access to a third party.
* It can produce false positives, especially in SAST scanning.

The combination of Renovate + Trivy covers the SCA and container scanning use cases that Snyk provides, without the commercial dependency.

## Migration Strategy

1. **Establish the security response team**: Designate the `platform-mesh/security` GitHub team and confirm membership. This team will own the vulnerability management process from day one.
2. **Enable GitHub org settings**: Turn on Dependency graph, Dependabot alerts, secret scanning, and push protection org-wide. Enable private vulnerability reporting on all repositories. Ensure Dependabot security updates are disabled.
3. **Update SECURITY.md**: Update the existing org-level `SECURITY.md` in `platform-mesh/.github` using the template in this RFC, including the reporting channels and the scanner-results policy.
4. **Install Renovate**: Install the [Mend Renovate App](https://github.com/apps/renovate) on the platform-mesh organization. Start with 2–3 repos for validation.
5. **Create shared Renovate config**: Add the security-only configuration to `platform-mesh/.github` so all repos can extend from it.
6. **Merge onboarding PRs**: Review and merge Renovate's onboarding PRs on the initial repos. Validate that security PRs are created correctly and that the triage process works end-to-end.
7. **Roll out org-wide**: Grant Renovate access to all repositories. Merge onboarding PRs across the org.
8. **Add Trivy workflow**: Create the reusable container scanning workflow in `platform-mesh/.github`. Add it to repos that build container images with non-minimal base images.
9. **Add Scorecard workflow**: Add the OpenSSF Scorecard GitHub Action to all repositories. Enable `publish_results: true` and add the Scorecard badge to each repo's README.
10. **Configure notifications**: Set code scanning notification recipients to `platform-mesh/security` in each repo's settings. Optionally, subscribe to `security_alert` events in the team's Slack channel via the GitHub-Slack integration.

## Future Considerations

The following areas are not covered by the current proposal and should be considered as long-term next steps:

* **License compliance scanning** — verify that all dependencies use licenses compatible with Apache-2.0. Tools like [wwhrd](https://github.com/frapposelli/wwhrd) (for Go) or [go-licenses](https://github.com/google/go-licenses) can be integrated into CI to flag incompatible licenses before they are merged.
* **Static application security testing (SAST)** — analyze Go and TypeScript source code for vulnerabilities such as injection flaws, path traversal, or insecure function usage. GitHub's [CodeQL](https://codeql.github.com/) is free for public repositories and can be enabled via a GitHub Actions workflow. Enabling CodeQL would also improve OpenSSF Scorecard scores, as Scorecard checks for SAST tooling.
* **SBOM-based vulnerability tracking** — RFC 001 introduces SBOM generation (CycloneDX and SPDX) for container images. A future step would be to continuously scan those SBOMs against vulnerability databases after release, ensuring that newly disclosed CVEs are detected in already-shipped artifacts, not just in source repositories.
