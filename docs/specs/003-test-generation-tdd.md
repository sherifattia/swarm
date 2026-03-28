# Specification: Test Generation & Red Gate (TDD Phase)

> **Status**: Placeholder — needs full SDD treatment before TDD.

The test-generation phase of the delegate pipeline. Takes an approved canonical spec, mechanically maps each section to the corresponding test type, generates minimal stubs, and enforces the Red Gate — all tests must fail before implementation begins. This is the most deterministic phase: the mapping from spec section to test type is fixed, not LLM-judged.

This runs inside an Open SWE sandbox. The target repo is already cloned. The agent detects the repo's language and selects the appropriate test framework.

## Scope

- Spec parsing: extract each section and subsection from canonical 5-section format
- Mechanical test mapping (fixed rules, not LLM judgment):

| Spec Section | Test Type | Naming Convention |
|---|---|---|
| 1.2 Postconditions | Unit tests | `test_postcondition_{description}` |
| 1.1 Preconditions | Error-expecting tests | `test_precondition_violation_{error_type}` |
| 1.3 Invariants | Property-based tests | `test_invariant_{property}` |
| 3 Edge Cases | Dedicated edge case tests | `test_edge_case_{number}_{description}` |
| 2 Interfaces | Integration tests | `test_integration_{interface}` |
| 4.1 Performance | Performance tests (marked slow) | `test_perf_{bound}` |
| 4.2 Memory | Resource tests | `test_memory_{constraint}` |
| 4.3 Security | Security tests | `test_security_{requirement}` |
| 5.2 Testable Properties | Property-based tests | `test_property_{name}` |

- Language detection and test framework selection:

| Language | Test Framework | Linter | Coverage | Property-based |
|---|---|---|---|---|
| Python | pytest | ruff, black | coverage.py | hypothesis |
| TypeScript/JS | jest or vitest | eslint, prettier | nyc or c8 | fast-check |
| Go | go test | golangci-lint | go test -cover | rapid |
| Rust | cargo test | clippy, rustfmt | cargo tarpaulin | proptest |
| Java/Kotlin | JUnit, gradle | checkstyle, ktlint | jacoco | — |

- Minimal stub generation: correct function signatures with `NotImplementedError` (or language equivalent) bodies — never spec-defined errors, never partial implementations
- Red Gate enforcement: run all tests, assert 100% failure rate
  - If any test passes → `RedGateFailureError` with list of passing test names
  - Passing tests indicate tautological tests or stubs that accidentally implement behavior
- Test naming convention enforcement (all test names must match one of the patterns above)
- Test file placement following repo conventions (or sensible defaults if none exist)

## Key Interfaces

```
Input:  Approved spec file (canonical 5-section markdown)
        + target repo (cloned in sandbox, language detected)

Output: Test suite files (one or more test files following repo conventions)
        + stub files (minimal implementations with correct signatures)
        + RedGateResult { passed: true }
        | RedGateFailureError { passing_tests: string[] }
```

## Error Behaviors

| Condition | Behavior |
|---|---|
| Spec missing required sections | Reject — spec should have been approved by SDD phase. Report which sections are missing. |
| Unsupported language | Reply in thread: "Target repo uses {language}, not currently supported. Supported: Python, TypeScript/JS, Go, Rust, Java, Kotlin." Do not proceed. |
| No test framework in repo | Install default framework for detected language (e.g., `pip install pytest` for Python) |
| Some tests pass against stubs (Red Gate failure) | Reply in thread: "Red Gate failed — {N} tests passed against stubs. These may be tautological. Flagging for human review: [test names]." Do not proceed to implementation. |
| Existing tests in repo conflict with generated test names | Prefix generated tests with `vdd_`. Never modify or delete existing tests. |

## Edge Cases to Address

- Spec has edge cases that are untestable (e.g., "user abandons mid-session") — generate a test marked as `skip` with a comment explaining why
- Repo has a custom test runner or non-standard test directory structure
- Spec references types or interfaces that don't exist yet (stubs must create them)
- Property-based tests require a library not in the repo's dependencies (install it)
- Performance tests need baseline data that doesn't exist yet

## Key Design Decisions

- **Mechanical, not creative**: The mapping from spec section → test type is a fixed table, not an LLM judgment call. This makes the phase reproducible and auditable.
- **Stubs must be minimal**: `NotImplementedError` only. If a stub accidentally implements behavior, the Red Gate catches it. This is why stubs never raise spec-defined errors — those would make error-expecting tests pass.
- **Implementation order** (for the subsequent VDD phase): postconditions (1.2) first, then precondition violations (1.1), invariants (1.3), edge cases (3), integration (2), performance (4.1) last. This order is encoded in the test file structure.

## Performance Bounds

| Metric | Bound |
|---|---|
| Test generation + stub creation + Red Gate | < 10 minutes |
