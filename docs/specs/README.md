# Specs

Specs are the primary artifact in swarm. Each spec describes a feature end-to-end — behavior, constraints, API surface, and enough implementation detail for a coding agent to build it in a single session.

## Conventions

- One file per feature: `NNN-descriptive-title.md`
- Specs are numbered sequentially: `001-*.md`, `002-*.md`, etc.
- Each spec references its parent ADRs for architectural context

## Status Values

| Status | Meaning |
|--------|---------|
| `draft` | Work in progress, not ready for implementation |
| `ready` | Written and approved, ready for a coding agent |
| `in-progress` | A coding agent is actively implementing this spec |
| `done` | Implemented and verified |
| `blocked` | Waiting on a dependency (noted in the spec) |

## Current Specs

| Spec | Title | Status |
|------|-------|--------|
