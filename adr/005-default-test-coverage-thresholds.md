# ADR 005: Default Test Coverage Thresholds

| Status  | Approved   |
|---------|------------|
| Date    | 2026-04-02 |

## Context and Problem Statement

The primary goal is high-quality, relevant test suites — not a coverage number. Coverage thresholds are not a target to optimize for but a guardrail to ensure that tests are written. High coverage targets can work against this by incentivizing low-value tests written solely to satisfy the metric. AI-assisted development amplifies this by making it easy to generate inflated test suites that are hard to maintain and review.

At the same time, removing coverage gates entirely is not viable — enterprise product standards commonly require a minimum test coverage percentage, so adopting organizations expect enforcement. Hard gates also keep coverage visible to approvers and prevent untested code from merging.

## Decision Outcome

Set the default coverage thresholds to **80/80/80**:

```yaml
threshold:
  file: 80
  package: 80
  total: 80
```

Individual repositories may override these thresholds in either direction based on their requirements and maturity — for example, early-stage prototypes or proof-of-concepts may justify lower thresholds.

### Consequences

* Good, because uniform thresholds are simple to understand and communicate.
* Good, because 80% is achievable without writing tests for trivial code paths.
* Good, because a hard gate at 80% prevents untested features from merging.
* Good, because enforcing a coverage threshold satisfies enterprise product standards that require minimum test coverage.
* Bad, because some contributors may treat 80% as a ceiling rather than a floor.
