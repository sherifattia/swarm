# Specification: VDD Pipeline

> **Status**: Placeholder — needs full SDD treatment before TDD.
>
> **Parent**: [001 — VSDD Slack/GitHub Agent](001-vsdd-slack-github-agent.md)

The implementation phase of the VSDD pipeline. Takes a plan with explicitly marked phases and a passing test suite, implements each phase, runs parallel critics (code-quality-enforcer + architecture-critic) after each phase, and enforces test gates between phases.

## Scope

- Phase extraction from plan file (parsing `## Phase N` markers)
- Per-phase implementation orchestration via Open SWE `task` tool
- Parallel critic invocation (two fresh subagent instances per iteration):
  - code-quality-enforcer: linters, tests, coverage, bug/anti-pattern analysis
  - architecture-critic: pattern consistency, coupling, breaking changes, module boundaries
- Critique finding consolidation: merge by file:line, take max severity on conflicts
- Severity evaluation: critical/major → fix and re-critique, minor/suggestion → proceed
- Iteration control (max 3 per phase)
- Test gate between phases: run full test suite, all tests must pass before next phase
- Escalation when max iterations exhausted with unresolved critical/major findings

## Key Interfaces

```
Input:  Plan file (phased markdown) + test suite + repo sandbox
Output: Implementation files (all tests passing) | TestGateFailureError
```

## Covered Parent Spec Elements

- Postconditions: Q4
- Invariants: I2, I3, I4, I5
- Error types: TestGateFailureError, LLMUnavailableError, SandboxTimeoutError
- Edge cases: E4, E5, E8, E12
