---
status: "accepted"
date: 2026-03-28
decision-makers: ["sherifattia"]
---

# Adopt VDD as the spec-driven development methodology

## Context and Problem Statement

delegate is an autonomous coding agent that turns feature requests into tested pull requests. The quality of its output depends entirely on the methodology that governs how specs are written, how tests are derived, and how implementation is validated. We need a spec-driven development methodology that:

1. Defines a canonical spec format that agents can parse mechanically
2. Produces tests directly from the spec (not ad hoc)
3. Includes adversarial quality gates (not self-review)
4. Works headlessly — no interactive IDE pairing required
5. Bounds iteration to prevent runaway costs

Several open-source frameworks have emerged for structuring AI-assisted development around specifications. This ADR evaluates six options and selects one as delegate's methodology.

## Decision Drivers

* Must support autonomous headless execution (triggered by webhook, not interactive IDE session)
* Must produce tests mechanically from the spec (spec sections map to test types)
* Must include adversarial quality gates using fresh agent instances (not self-review by the same context)
* Must define a canonical spec format that is parseable and enforceable by agents
* Must bound iteration counts to cap LLM costs per feature
* Should separate concerns: spec writing, critique, test generation, implementation, and code review should be distinct roles
* Should include empirical verification gates (tests, coverage, linting) that cannot be reward-hacked

## Considered Options

* [VDD (Verification Driven Development)](#vdd-verification-driven-development) — custom methodology
* [GitHub Spec Kit](#github-spec-kit) — by GitHub
* [BMAD-METHOD](#bmad-method) — by BMad Code
* [Agent OS](#agent-os) — by Builder Methods
* [Spec Kitty](#spec-kitty) — by Priivacy AI
* [OpenSpec](#openspec) — by Fission AI

## Decision Outcome

Chosen option: "VDD", because it is the only methodology that combines all four properties we require:

1. **Adversarial parallel critique** — spec-critic, code-quality-enforcer, and architecture-critic run as fresh agent instances on every iteration, eliminating self-consistency bias
2. **Mechanical spec-to-test mapping** — each of the five canonical spec sections maps to specific test types (postconditions → unit tests, preconditions → error tests, invariants → property-based tests, edge cases → dedicated tests, interfaces → integration tests), enabling `/tdd` to generate tests without human judgment
3. **Empirical verification gates** — tests, coverage thresholds, and linters run between every VDD phase; an LLM cannot fake passing tests
4. **Red Gate discipline** — all generated tests must fail before implementation begins, proving the tests are real and the stubs are minimal

VDD is purpose-built for headless autonomous execution, which aligns with Open SWE's single Deep Agent architecture. The four VDD subagents (spec-builder, spec-critic, code-quality-enforcer, architecture-critic) map naturally to Open SWE's `task` tool for subagent spawning.

### Consequences

* Good, because adversarial critique with fresh instances catches issues that self-review misses (eliminates anchoring bias)
* Good, because mechanical spec-to-test mapping eliminates ambiguity in what gets tested — the spec is the single source of truth
* Good, because empirical verification (tests/coverage/linting) prevents reward-hacking — the LLM cannot convince itself its code is correct without proof
* Good, because severity calibration (critical/major/minor/suggestion) prevents perfectionism from blocking progress
* Good, because iteration limits (5 SDD iterations, 3 VDD iterations per phase) bound costs to a predictable ceiling (~34 Opus calls max)
* Good, because the Red Gate proves tests are meaningful before implementation begins
* Bad, because it requires defining and maintaining 4 subagent prompts (spec-builder, spec-critic, code-quality-enforcer, architecture-critic)
* Bad, because using Opus for all agents increases cost per run compared to mixed-model approaches
* Bad, because it is a custom methodology with no community ecosystem — we own all maintenance and evolution
* Neutral, because the canonical 5-section format is rigid by design — features that don't fit the format require adaptation

## Pros and Cons of the Options

### VDD (Verification Driven Development)

Custom methodology. Source: `~/code/vdd/`

VDD is a three-phase pipeline — SDD (spec), TDD (tests), VDD (implementation) — orchestrated by the `/vsdd` command. It uses a canonical 5-section spec format (Behavioral Contract, Interface Definition, Edge Case Catalog, Non-Functional Requirements, Verification Strategy) that is mechanically parsed by the TDD phase to generate tests. Four specialized subagents handle distinct roles: spec-builder drafts specs, spec-critic provides adversarial review, code-quality-enforcer runs linters/tests/coverage and analyzes code, and architecture-critic evaluates pattern consistency and coupling. Each critic runs as a fresh agent instance on every iteration to prevent anchoring bias.

The TDD phase enforces a Red Gate: all generated tests must fail before implementation begins (proving stubs raise `NotImplementedError`, not spec-defined errors). The VDD phase implements code in plan-defined phases, running two parallel critics (code-quality-enforcer + architecture-critic) after each phase with up to 3 iterations per phase. Severity calibration (critical/major/minor/suggestion) is consistent across all agents and prevents over-rating from causing thrashing.

* Good, because the canonical 5-section format enables mechanical test derivation — no human judgment needed for test generation
* Good, because adversarial critique uses fresh agent instances, eliminating self-consistency bias
* Good, because the Red Gate proves tests are meaningful before any implementation
* Good, because parallel dual critics (code quality + architecture) catch different failure modes
* Good, because iteration limits (5 SDD, 3 VDD) bound costs predictably
* Good, because severity calibration prevents perfectionism from blocking progress
* Good, because it supports auto-detected language tooling (Python/pytest/ruff, TypeScript/jest/eslint, Go, Rust, Java/Kotlin)
* Bad, because it is a custom methodology with zero community — we own all maintenance
* Bad, because Opus for all agents means higher per-run cost
* Bad, because the rigid 5-section format may not fit every feature shape
* Bad, because correlated failures are possible (same model for implementation and critique)

### GitHub Spec Kit

[GitHub](https://github.com/github/spec-kit) · TypeScript · Open Source

GitHub Spec Kit implements "intent-driven development" — developers focus on what users need and why, while AI agents handle the mechanical translation to implementation. It provides slash commands (`/speckit.specify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.implement`) that guide development through structured templates. A constitutional system (`/speckit.constitution`) establishes project-level governance principles. Optional commands include `/speckit.clarify` for ambiguity resolution, `/speckit.analyze` for cross-artifact consistency checks, and `/speckit.checklist` for quality validation described as "unit tests for English."

The spec format is template-driven (user stories, requirements) rather than a fixed canonical structure. Quality enforcement relies on checklists and consistency analysis rather than adversarial multi-agent critique. An extension system supports integrations with Jira, Linear, and GitHub Issues.

* Good, because it is backed by GitHub with active maintenance and community
* Good, because the constitutional system provides project-level governance that persists across features
* Good, because `/speckit.analyze` provides cross-artifact consistency checking
* Good, because `/speckit.checklist` generates quality validation ("unit tests for English")
* Good, because the extension system supports Jira, Linear, and GitHub Issues integrations
* Bad, because quality enforcement uses checklists, not adversarial multi-agent critique
* Bad, because the spec format is template-driven, not a fixed canonical structure — test generation requires human judgment
* Bad, because there is no Red Gate or test-first methodology — tests are not mechanically derived from spec sections
* Bad, because it is designed for interactive IDE sessions (slash commands), not headless autonomous execution
* Bad, because there is no iteration bounding or cost control mechanism

### BMAD-METHOD

[GitHub](https://github.com/bmad-code-org/bmad-method) · NPM Package · Open Source

BMAD-METHOD (Build More Architect Dreams) is an AI-driven agile framework with scale-adaptive intelligence — it automatically adjusts planning depth based on project complexity. It installs as an NPM package providing specialized AI agents and guided workflows. The framework supports greenfield projects with a comprehensive workflow (PRD → architecture → implementation) and a Quick Flow track for smaller changes. It includes adversarial review and an "edge case hunter" for validation. A "party mode" enables multi-agent group discussion for ideation.

Context engineering principles ensure each development phase produces documentation that informs subsequent steps. The framework covers the full lifecycle from brainstorming to deployment.

* Good, because scale-adaptive intelligence adjusts methodology depth to project complexity
* Good, because it includes adversarial review and edge case hunting
* Good, because the NPM package makes installation straightforward (`npx bmad-method install`)
* Good, because it supports non-interactive CI/CD installation mode
* Good, because context engineering ensures documentation flows between phases
* Bad, because the spec format is not canonical — it is adaptive rather than mechanically parseable
* Bad, because there is no mechanical spec-to-test mapping — test generation is not deterministic from spec structure
* Bad, because there is no Red Gate discipline
* Bad, because the scale-adaptive approach may under-specify for complex autonomous agent scenarios
* Bad, because the full lifecycle scope (PRD → deploy) is broader than what we need — we want spec → tested PR

### Agent OS

[GitHub](https://github.com/buildermethods/agent-os) · Markdown Profiles · Open Source · Free

Agent OS is a spec-driven agentic development system with six phases: product planning, spec shaping, spec writing, task creation, task implementation, and task orchestration. It uses modular profiles containing agent definitions, command workflows, and coding standards that install into any project. It supports both single-agent and multi-agent orchestration modes with specialized subagents for planning, specification, verification, and implementation. The spec writer agent creates specs with: overview, user stories, technical requirements, API endpoints, security considerations, testing requirements, and acceptance criteria.

Multi-agent orchestration delegates tasks to specialized implementer agents, coordinates dependencies, runs verifiers, and handles errors with automatic retries. It supports parallel task execution and progress reporting.

* Good, because the six-phase lifecycle is comprehensive (planning through orchestration)
* Good, because multi-agent orchestration supports parallel execution with dependency tracking
* Good, because modular profiles are language/framework agnostic
* Good, because it includes a verifier step after task completion
* Good, because the spec writer agent produces structured specs with testable acceptance criteria
* Bad, because the spec format (user stories + acceptance criteria) is not a canonical structure that enables mechanical test derivation
* Bad, because there is no adversarial critique — verification is pass/fail, not multi-perspective review
* Bad, because there is no Red Gate or test-first discipline
* Bad, because there is no iteration bounding or cost control mechanism
* Bad, because the retry mechanism is simple (retry on error) rather than critique-driven improvement

### Spec Kitty

[GitHub](https://github.com/priivacy-ai/spec-kitty) · CLI · Open Source

Spec Kitty is an open-source CLI workflow for spec-driven development with a pipeline: `spec` → `plan` → `tasks` → `implement` → `review` → `merge`. Its differentiator is multi-agent coordination — multiple AI coding agents work on a single feature simultaneously, each in isolated git worktrees to minimize merge friction. A live kanban dashboard provides real-time visibility into progress across all work packages. It supports 10+ AI coding agents (Claude Code, Cursor, Windsurf, Gemini CLI, GitHub Copilot, and others).

The orchestration system handles autonomous implementation with dependency-aware task ordering, automatic review stages, and built-in fallback mechanisms.

* Good, because multi-agent parallel execution in isolated worktrees minimizes merge friction
* Good, because it supports 10+ AI coding agents — maximum flexibility in agent choice
* Good, because the live kanban dashboard provides real-time visibility
* Good, because dependency-aware task ordering prevents conflicts
* Good, because it includes automatic review stages in the orchestration pipeline
* Bad, because the spec format is not canonical — it is flexible rather than mechanically parseable
* Bad, because review is a pipeline stage, not adversarial multi-perspective critique with fresh instances
* Bad, because there is no mechanical spec-to-test mapping
* Bad, because there is no Red Gate or test-first discipline
* Bad, because the multi-agent coordination adds complexity that we don't need — Open SWE already provides subagent orchestration via the `task` tool

### OpenSpec

[GitHub](https://github.com/fission-ai/openspec) · CLI · Open Source

OpenSpec is a lightweight framework where a spec is defined as a "verifiable behavior contract at a boundary." It uses a progressive rigor approach — humans provide intent, constraints, and examples; agents convert these into behavior-first requirements and scenarios. The artifact pipeline is: `proposal` → `specs` → `design` → `tasks` → `apply`. A single `/opsx:propose` command generates all planning artifacts in one step. The CLI provides `openspec instructions` for agent consumption with JSON output.

The philosophy emphasizes behavior-first contracts, progressive rigor, and agent-aligned authoring. Specs focus on what users, integrators, or operators can observe and rely on.

* Good, because "verifiable behavior contract" aligns with our goal of testable specs
* Good, because progressive rigor allows starting loose and tightening
* Good, because JSON output mode is designed for agent consumption
* Good, because it is lightweight and unopinionated about tooling
* Good, because the single-command artifact generation (`/opsx:propose`) is fast for simple features
* Bad, because there is no adversarial critique or multi-agent review
* Bad, because the spec format is flexible (behavior-first but not canonical sections) — test generation is not mechanical
* Bad, because there is no Red Gate or test-first methodology
* Bad, because there is no iteration bounding or cost control
* Bad, because it is early-stage — spec inheritance, multi-repo support, and dependency tracking are still open topics

## Comparison Matrix

| | VDD | Spec Kit | BMAD | Agent OS | Spec Kitty | OpenSpec |
|---|---|---|---|---|---|---|
| **Spec format** | Canonical 5-section (mechanical) | Templates (flexible) | Adaptive (flexible) | User stories + acceptance criteria | Flexible | Behavior contracts (flexible) |
| **Multi-agent** | 4 specialized subagents | Single agent | Multi-persona + party mode | Specialized subagents | Multi-agent parallel | Single agent |
| **Adversarial critique** | Fresh instances, parallel dual critics | Checklists + analysis | Adversarial review + edge case hunter | Verifier (pass/fail) | Review stage | None |
| **Test generation** | Mechanical (spec section → test type) | Not specified | Not specified | Acceptance criteria | Not specified | Not specified |
| **Red Gate** | Yes (all tests must fail first) | No | No | No | No | No |
| **Iteration limits** | 5 SDD, 3 VDD per phase | None | None | Retry on error | None | None |
| **Severity calibration** | Critical/major/minor/suggestion | N/A | N/A | N/A | N/A | N/A |
| **Headless execution** | Yes (designed for it) | No (IDE slash commands) | Yes (CI/CD mode) | Yes (orchestration) | Yes (CLI orchestration) | Yes (CLI + JSON) |
| **Cost control** | Iteration caps (~34 Opus calls max) | None | Scale-adaptive | None | None | None |

## More Information

### The VDD Pipeline (VSDD)

The full pipeline runs three phases in sequence:

```
Feature Request
    │
    ▼
┌─────────────────────────────────────┐
│  SDD Phase (Spec)                   │
│  spec-builder drafts canonical spec │
│  spec-critic reviews (max 5 iter)   │
│  Human approves                     │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  TDD Phase (Tests)                  │
│  Mechanical test generation         │
│  Red Gate: ALL tests must fail      │
│  Minimal stubs (NotImplementedError)│
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  VDD Phase (Implementation)         │
│  Per-phase implementation           │
│  Parallel critics after each phase  │
│    ├── code-quality-enforcer        │
│    └── architecture-critic          │
│  Max 3 iterations per phase         │
│  Test gate between phases           │
└──────────────┬──────────────────────┘
               │
               ▼
         Tested PR
```

### Canonical Spec Format (5 Sections)

```markdown
# Specification: {Feature Name}

## 1. Behavioral Contract
### 1.1 Preconditions        → error-expecting tests
### 1.2 Postconditions       → unit tests
### 1.3 Invariants           → property-based tests

## 2. Interface Definition
### 2.1 Input Types          → integration tests
### 2.2 Output Types
### 2.3 Error Types

## 3. Edge Case Catalog      → dedicated edge case tests

## 4. Non-Functional Requirements
### 4.1 Performance Bounds   → performance tests
### 4.2 Memory Constraints   → resource tests
### 4.3 Security Requirements → security tests

## 5. Verification Strategy
### 5.1 Provable Properties
### 5.2 Testable Properties  → property-based tests
### 5.3 Verification Constraints
```

### Four Subagents

| Agent | Role | Runs During |
|-------|------|-------------|
| **spec-builder** | Drafts and revises specs in canonical format; treats specs like mathematical proofs | SDD phase |
| **spec-critic** | Adversarial reviewer: finds ambiguity, missing edge cases, unstated assumptions, contradictions | SDD phase |
| **code-quality-enforcer** | Runs linters, tests, coverage; analyzes for bugs, anti-patterns, error handling gaps, type safety | VDD phase |
| **architecture-critic** | Evaluates pattern consistency, coupling, breaking changes, API surface, module boundaries | VDD phase |

### VDD Source

The VDD framework is defined at `~/code/vdd/`. Key files:

* `commands/vsdd.md` — full pipeline orchestration
* `commands/sdd.md` — spec phase with canonical format definition
* `commands/tdd.md` — test generation with Red Gate and mechanical mapping
* `commands/vdd.md` — phased implementation with parallel critics
* `agents/spec-builder.md` — spec drafting agent
* `agents/spec-critic.md` — adversarial spec reviewer
* `agents/code-quality-enforcer.md` — code quality analysis agent
* `agents/architecture-critic.md` — architectural fit reviewer
* `docs/design-rationale.md` — methodology philosophy and theory
