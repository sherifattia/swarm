# Specification: Spec Authoring (SDD Phase)

> **Status**: Placeholder — needs full SDD treatment before TDD.

The spec-writing phase of the delegate pipeline. Takes a feature description and repository context, orchestrates the spec-builder and spec-critic subagents in an adversarial loop, and produces an approved specification in VDD's canonical 5-section format — or escalates to the user if the spec cannot be approved.

This runs inside an Open SWE sandbox. The spec-builder and spec-critic are spawned as subagents via the `task` tool, each getting its own isolated context. The spec-critic is always a fresh instance (no carryover from previous iterations) to prevent anchoring bias.

## Scope

- Spec-builder subagent invocation via Open SWE `task` tool
  - Input: feature description + repo context (`AGENTS.md`, language, existing patterns)
  - Output: canonical 5-section spec file
- Spec-critic subagent invocation as a fresh instance per iteration
  - Examines: ambiguous language, missing edge cases, unstated assumptions, internal contradictions, verification soundness
  - Severity ratings: critical (must fix), major (should fix), minor (can defer), suggestion (optional)
- Adversarial loop control:
  - Max 5 iterations
  - Continue if critic finds critical or major issues and iterations remain
  - Approve if no critical or major findings
  - Escalate if 5 iterations exhausted with unresolved critical/major issues
- Canonical 5-section format validation (all sections and required subsections present):
  1. Behavioral Contract (Preconditions, Postconditions, Invariants)
  2. Interface Definition (Input Types, Output Types, Error Types)
  3. Edge Case Catalog
  4. Non-Functional Requirements (Performance, Memory, Security)
  5. Verification Strategy (Provable, Testable, Constraints)
- Clarification round-trip with user (up to 2 rounds via Slack/GitHub thread) when description is insufficient
- Audit trail: post SDD iteration status and critic findings to source thread

## Key Interfaces

```
Input:  Feature description (string)
        + repo context (AGENTS.md content, detected language, existing patterns)
        + source thread reference (for clarification and audit trail)

Output: Approved spec file (canonical 5-section markdown in sandbox)
        + audit trail messages (iteration count, critic findings, approval status)
        | SpecRejectedError (after 5 iterations with unresolved critical/major findings)
        | InsufficientDescriptionError (after 2 clarification rounds with no adequate response)
```

## Error Behaviors

| Condition | Behavior |
|---|---|
| Description too vague for Sections 1-3 | Ask up to 2 clarifying questions in source thread |
| Still insufficient after 2 rounds | Reply: "I need more detail to write a spec. Please describe: [specific missing elements]" |
| Spec-critic finds critical/major after 5 iterations | Reply: "Spec could not be approved after 5 iterations. Last findings: [summary]" |
| LLM API unreachable | Retry 3× with exponential backoff, then reply: "Unable to reach LLM API" |
| Spec-builder produces spec missing sections | Spec-critic catches as critical finding, loop continues |

## Edge Cases to Address

- User abandons mid-SDD (no reply for extended period) — run stays active via LangGraph durable execution, resumes on reply
- Spec-builder produces contradictory postconditions/preconditions
- Feature description in non-English language
- Spec-critic flags the same issue across multiple iterations (indicates spec-builder cannot resolve it)
- LLM rate limiting during spec-builder/spec-critic calls

## Key Design Decisions

- **Fresh instances**: Each spec-critic invocation is a new subagent (no conversation history from prior iterations). This prevents the critic from anchoring to its previous findings and ensures independent review.
- **Severity calibration**: When uncertain between severity levels, critics default to the lower severity. Finding many issues is good; over-rating causes thrashing.
- **Spec is single source of truth**: Once approved, the spec drives all downstream phases (TDD test generation, VDD implementation). Changes require re-running SDD, not mid-implementation edits.

## VDD Subagent Definitions

- **spec-builder** (`~/code/vdd/agents/spec-builder.md`): Systems engineer persona. Writes specs like mathematical proofs. Every behavior as precondition/postcondition/invariant. Exact types with bounds. Exhaustive edge cases.
- **spec-critic** (`~/code/vdd/agents/spec-critic.md`): Formal methods practitioner persona. Finds every way a spec can be misinterpreted. Examines ambiguous language, missing edge cases, unstated assumptions, contradictions, verification soundness.

## Performance Bounds

| Metric | Bound |
|---|---|
| Single iteration (spec-builder + spec-critic) | < 5 minutes |
| Full SDD phase (worst case, 5 iterations) | < 25 minutes |
| Clarification round-trip | User-dependent (no timeout) |
