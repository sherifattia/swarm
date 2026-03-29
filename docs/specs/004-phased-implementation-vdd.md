# Specification: Phased Implementation with Critics (VDD Phase)

> **Status**: Placeholder — needs full SDD treatment before TDD.

The implementation phase of the delegate pipeline. Takes a plan with explicitly marked phases and a test suite where all tests currently fail (Red Gate passed), then implements each phase while running parallel adversarial critics and enforcing test gates between phases.

This runs inside an Open SWE sandbox. Implementation and critique happen via subagents spawned with the `task` tool. Each critic is a fresh instance per iteration — no context carryover.

## Scope

- Phase extraction from plan (parsing `## Phase N: {description}` markers)
- Per-phase implementation via Open SWE `task` tool subagent
- Parallel critic invocation after each phase (two fresh subagent instances):
  - **code-quality-enforcer**: runs linters, tests, coverage analysis; checks for bugs, anti-patterns, error handling gaps, type safety, inefficient patterns
  - **architecture-critic**: reads changed files, explores surrounding codebase, evaluates pattern consistency, coupling, breaking changes, API surface, module boundaries
- Critique finding consolidation:
  - Merge findings by file:line reference
  - When both critics flag the same location, take the higher severity
  - Severity levels: critical (must fix), major (should fix), minor (can defer), suggestion (optional)
- Decision logic per iteration:
  - If critical or major findings AND iteration < 3: fix issues and re-run critics
  - If critical or major findings AND iteration >= 3: escalate to human review
  - If no critical or major findings: proceed to next phase
- Test gate between phases: run full test suite, all tests for completed phases must pass
- Iteration control: max 3 critique iterations per phase
- Structured logging per phase:
  ```
  === PHASE {N} START: {description} ===
  MODIFIED: {file}
  CREATED: {file}
  DEFERRED [minor]: {file}:{line} - {issue}
  === PHASE {N} COMPLETE ({iterations} iterations) ===
  ```
- Audit trail: post phase completion status and critic findings to source thread

## Key Interfaces

```
Input:  Plan file (markdown with ## Phase N markers)
        + test suite (all tests failing — Red Gate passed)
        + repo in sandbox (cloned, stubs in place)

Output: Implementation files (all phase tests passing)
        + structured log of phases, modifications, and deferred issues
        + audit trail messages
        | TestGateFailureError (tests failing after 3 critique iterations)
```

## Error Behaviors

| Condition | Behavior |
|---|---|
| Tests fail after implementation + 3 critique iterations | Reply: "Tests failing after 3 rounds of critique: [failing tests]. Flagging for human review." |
| Sandbox times out during test run | Reply: "Sandbox timed out after {duration}. Test suite may need optimization." Do not retry automatically. |
| Sandbox becomes unreachable | Auto-recreate (Open SWE default). If recreation fails 3×, reply: "Sandbox unavailable." |
| LLM rate limiting during parallel critics | Retry with exponential backoff. If persistent, serialize critics (code-quality-enforcer first, then architecture-critic). |
| Critics disagree on severity for same finding | Take the higher severity. Both findings appear in audit trail. |
| Test gate fails but critics found no issues | Re-enter critique loop with test failure output appended to critic prompts as additional context. Count as a critique iteration. |

## Edge Cases to Address

- Plan has no explicit phase markers (treat entire plan as one phase)
- A phase produces no file changes (skip critique, proceed)
- Implementation introduces a regression in a previously-passing phase's tests
- Critic flags an issue in code that was not modified in the current phase (out-of-scope finding)
- Deferred minor issues accumulate across phases — should they be collected in the PR description?

## Key Design Decisions

- **Fresh critic instances**: Each critique spawns new subagents via `task`. No conversation history from prior iterations. This prevents anchoring to previous findings and ensures independent review.
- **Parallel critics**: Code-quality-enforcer and architecture-critic run simultaneously, not sequentially. They examine different dimensions (correctness vs. fit) and their findings are merged post-hoc.
- **Severity calibration prevents thrashing**: Minor and suggestion findings are deferred, not fixed. Only critical and major trigger re-implementation. When uncertain, critics default to lower severity.
- **Empirical floor**: Tests, coverage, and linters are non-negotiable. An LLM cannot convince itself its code is correct — it must prove it with passing tests.

## VDD Subagent Definitions

- **code-quality-enforcer** (`~/code/vdd/agents/code-quality-enforcer.md`): Zero-tolerance enforcer. No placeholder TODOs, no bare `except:`, no stub implementations, no magic numbers. Runs language-specific linters (ruff/eslint/clippy/checkstyle), unit tests, and coverage (80% threshold default).
- **architecture-critic** (`~/code/vdd/agents/architecture-critic.md`): Senior architect persona. Protects codebase from architectural rot. Evaluates pattern consistency, dependency direction, breaking changes, API usability, module boundaries, portability.

## Performance Bounds

| Metric | Bound |
|---|---|
| Single VDD phase (implementation + up to 3 critique iterations) | < 20 minutes |
| Full VDD execution (typical: 3 phases × 1-2 iterations) | < 40 minutes |
| Full VDD execution (worst case: 4 phases × 3 iterations) | < 80 minutes |
