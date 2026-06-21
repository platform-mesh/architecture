# platform-mesh - Architecture

## About

This repository contains the architectural documentation for the [Platform Mesh](https://github.com/platform-mesh).

Architecture decisions and design proposals are documented through two complementary formats:

- **Architecture Decision Records (ADRs)** — capture significant technical decisions along with their context and consequences. ADRs follow the [MADR](https://github.com/adr/madr) template.
- **Requests for Comments (RFCs)** — propose and discuss larger features or cross-cutting changes before implementation begins. RFCs are more comprehensive than ADRs and are used to gather feedback from the broader team. A single RFC can lead to multiple ADRs as the design is refined into concrete decisions.

## Repository Structure

```
adr/           Architecture Decision Records
  └─ assets/   Diagrams and images referenced by ADRs
rfc/           Requests for Comments
  └─ assets/   Diagrams and images referenced by RFCs
```

## Creating a New Record

1. Create a new file in the `adr/` or `rfc/` directory.
2. Name it with the next available number and a short description: `001-<name>.md`.
3. For ADRs, follow the [MADR template](https://github.com/adr/madr/blob/develop/template/adr-template.md).
4. Open a pull request for review.

## Process

RFCs are open for discussion and feedback from all contributors. Use the pull request review process to comment, suggest changes, and iterate on the proposal.

ADRs require approval from the Platform Mesh **Technical Steering Committee (TSC)**, who are the [code owners](CODEOWNERS) of this repository. An ADR is considered accepted once the pull request is approved and merged.

<p align="center"><img alt="Bundesministerium für Wirtschaft und Energie (BMWE)-EU funding logo" src="https://apeirora.eu/assets/img/BMWK-EU.png" width="400"/></p>
