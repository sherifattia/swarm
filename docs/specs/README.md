# Specs

Specs are the refined input documents that get handed to a coding agent to produce a feature. They are interface-driven and detailed enough that any contributor can feed the same spec to a coding agent and get substantially similar output.

A spec is a historical snapshot — the final version of everything that led up to the point where we could reliably stamp out a feature. Once committed, it serves as a record of what was given to the agent and why the feature exists in its current form.

## Conventions

- One file per feature: `NNN-descriptive-title.md`
- Specs are numbered sequentially: `001-*.md`, `002-*.md`, etc.
- Specs are reviewed and approved before implementation begins

## Specs

| Spec | Title | Status |
|------|-------|--------|
| [001](001-vsdd-slack-github-agent.md) | VSDD Slack/GitHub Agent (system-level) | draft |
| [002](002-invocation-and-routing.md) | Invocation & Routing | placeholder |
| [003](003-sdd-pipeline.md) | SDD Pipeline | placeholder |
| [004](004-tdd-pipeline.md) | TDD Pipeline | placeholder |
| [005](005-vdd-pipeline.md) | VDD Pipeline | placeholder |
| [006](006-delivery.md) | Delivery | placeholder |
