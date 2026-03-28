# Specification: SDD Pipeline

> **Status**: Placeholder — needs full SDD treatment before TDD.
>
> **Parent**: [001 — VSDD Slack/GitHub Agent](001-vsdd-slack-github-agent.md)

The spec-writing phase of the VSDD pipeline. Takes a feature description and repository context, orchestrates the spec-builder and spec-critic subagent loop, and produces an approved canonical 5-section spec or escalates to the user.

## Scope

- Spec-builder subagent invocation (via Open SWE `task` tool) with feature description and repo context
- Spec-critic subagent invocation as a fresh instance per iteration
- Iteration loop control (max 5 iterations)
- Severity evaluation of critic findings (critical/major → revise, minor/suggestion → approve)
- Canonical 5-section format validation (all sections and required subsections present)
- Clarification round-trip with the user (up to 2 rounds via Slack/GitHub thread)
- Approval criteria: no critical or major findings from spec-critic

## Key Interfaces

```
Input:  Feature description (string) + repo context (AGENTS.md, language, existing patterns)
Output: Approved spec file (canonical 5-section markdown) | SpecRejectedError | InsufficientDescriptionError
```

## Covered Parent Spec Elements

- Preconditions: P2, P5
- Postconditions: Q1
- Invariants: I1, I2, I5
- Error types: InsufficientDescriptionError, SpecRejectedError, LLMUnavailableError
- Edge cases: E1, E2, E11, E12
