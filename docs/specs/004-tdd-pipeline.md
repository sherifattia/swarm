# Specification: TDD Pipeline

> **Status**: Placeholder — needs full SDD treatment before TDD.
>
> **Parent**: [001 — VSDD Slack/GitHub Agent](001-vsdd-slack-github-agent.md)

The test-generation phase of the VSDD pipeline. Takes an approved canonical spec, mechanically maps each section to the corresponding test type, generates minimal stubs, and enforces the Red Gate (all tests must fail before implementation).

## Scope

- Spec parsing: extract preconditions, postconditions, invariants, edge cases, interfaces, NFRs, and verification properties
- Mechanical test mapping:
  - Section 1.2 (Postconditions) → unit tests (`test_postcondition_*`)
  - Section 1.1 (Preconditions) → error-expecting tests (`test_precondition_violation_*`)
  - Section 1.3 (Invariants) → property-based tests (`test_invariant_*`)
  - Section 3 (Edge Cases) → dedicated tests (`test_edge_case_*`)
  - Section 2 (Interfaces) → integration tests (`test_integration_*`)
  - Section 4.1 (Performance) → performance tests (`test_perf_*`)
  - Section 4.3 (Security) → security tests (`test_security_*`)
  - Section 5.2 (Testable Properties) → property-based tests (`test_property_*`)
- Language detection and test framework selection (pytest, jest/vitest, go test, cargo test, JUnit)
- Minimal stub generation (correct signatures, `NotImplementedError` bodies)
- Red Gate enforcement: run all tests, assert 100% failure, flag any passing tests
- Test naming convention enforcement

## Key Interfaces

```
Input:  Approved spec file (canonical 5-section markdown) + target repo language/framework
Output: Test suite files + stub files + RedGateResult (pass | RedGateFailureError with list of passing test names)
```

## Covered Parent Spec Elements

- Postconditions: Q2, Q3
- Invariants: I1 (spec parsing validates format)
- Error types: RedGateFailureError
- Edge cases: E3, E6, E7, E13
