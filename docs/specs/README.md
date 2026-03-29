# Specs

Specs are the refined input documents that get handed to a coding agent to produce a feature. They are interface-driven and detailed enough that any contributor can feed the same spec to a coding agent and get substantially similar output.

A spec is a historical snapshot — the final version of everything that led up to the point where we could reliably stamp out a feature. Once committed, it serves as a record of what was given to the agent and why the feature exists in its current form.

## Conventions

- One file per feature: `NNN-descriptive-title.md`
- Specs are numbered sequentially: `001-*.md`, `002-*.md`, etc.
- Each spec is a standalone vertical slice — self-contained with all context an implementation agent needs
- Specs follow the VDD canonical 5-section format when fully specified (see [ADR-0002](../decisions/0002-spec-methodology.md))
- Specs are reviewed and approved before implementation begins

## Specs

| Spec | Title | Status |
|------|-------|--------|
| [001](001-invocation-and-routing.md) | Invocation & Routing | placeholder |
| [002](002-spec-authoring-sdd.md) | Spec Authoring (SDD Phase) | placeholder |
| [003](003-test-generation-tdd.md) | Test Generation & Red Gate (TDD Phase) | placeholder |
| [004](004-phased-implementation-vdd.md) | Phased Implementation with Critics (VDD Phase) | placeholder |
| [005](005-pr-creation-and-audit-trail.md) | PR Creation & Audit Trail | placeholder |
