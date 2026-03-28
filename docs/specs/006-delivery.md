# Specification: Delivery

> **Status**: Placeholder — needs full SDD treatment before TDD.
>
> **Parent**: [001 — VSDD Slack/GitHub Agent](001-vsdd-slack-github-agent.md)

The exit point of the VSDD pipeline. Takes implementation artifacts from the sandbox, creates a draft pull request on the target repository, and posts audit trail messages to the source Slack thread or GitHub issue throughout the pipeline.

## Scope

- Feature branch creation with unique naming (`delegate/{feature}-{id}`)
- Commit assembly: spec file + test suite + implementation files
- Draft PR creation via GitHub App API:
  - Title derived from spec Section 1
  - Body with summary, spec link, test results, and source thread/issue link
  - Always draft (never ready for review automatically)
- Audit trail messaging throughout pipeline:
  - SDD: iteration status, critic findings, approval
  - TDD: test generation summary, Red Gate result
  - VDD: phase start/complete, critic findings, test gate results
- Message formatting for Slack threads vs. GitHub issue comments
- Source linking (PR → thread/issue, thread/issue → PR)
- Safety net: `open_pr_if_needed` middleware behavior (auto-commit + PR if agent didn't)

## Key Interfaces

```
Input:  Implementation artifacts (files in sandbox) + pipeline metadata (source thread, repo, audit log)
Output: Draft PR (GitHub) + audit trail messages (Slack thread or GitHub comments)
```

## Covered Parent Spec Elements

- Postconditions: Q5, Q6, Q7
- Invariants: I8
- Error types: GitHubAppNotInstalledError (permission failure at PR creation)
- Edge cases: E10 (branch collision avoidance), E14 (no duplicate detection — human concern at PR review)
