# Plan: ADRs + Spec for delegate POC

## Context

Setting up the delegate POC for future implementation agents. We clone and customize Open SWE with VDD methodology. Our job: write the ADRs and spec so any implementation agent can build from them.

## Deliverables

### 3 ADRs + 1 Spec

| # | File | Title |
|---|------|-------|
| ADR-0002 | `docs/decisions/0002-spec-methodology.md` | Adopt VDD as spec-driven development methodology |
| ADR-0003 | `docs/decisions/0003-sandbox-provider.md` | Sandbox provider selection for POC |
| ADR-0004 | `docs/decisions/0004-poc-hosting.md` | POC hosting on LangSmith cloud |
| Spec 001 | `docs/specs/001-vsdd-slack-github-agent.md` | VSDD Slack/GitHub Agent |

### Also update
- `docs/decisions/README.md` — add ADR entries
- `docs/specs/README.md` — add spec entry

---

## ADR-0002: Adopt VDD as Spec-Driven Development Methodology

**Problem**: Need a methodology for translating feature requests into tested PRs. The spec format and development pipeline determine the quality of agent output.

**Decision Drivers**:
- Must support autonomous headless execution (not interactive IDE pairing)
- Must produce tests mechanically from spec (not ad hoc)
- Must have adversarial quality gates (not self-review)
- Must define a canonical spec format parseable by agents
- Should bound iteration to prevent runaway costs

**Considered Options**:
1. VDD (Verification Driven Development) — our custom methodology
2. GitHub Spec Kit — by GitHub
3. BMAD-METHOD — open source multi-persona framework
4. Agent OS — by Builder Methods
5. Spec Kitty — by Priivacy AI
6. OpenSpec — lightweight open source

**Decision**: VDD, because it is the only option with adversarial parallel critique (fresh instances per iteration), mechanical spec-to-test mapping (spec sections → test types), empirical verification gates (tests/coverage/linting as non-negotiable floor), and Red Gate discipline (all tests must fail before implementation). It is designed for headless autonomous execution, which aligns with Open SWE's architecture.

**Consequences**:
- Good: adversarial critique catches issues self-review misses
- Good: mechanical spec-to-test mapping eliminates ambiguity in what gets tested
- Good: empirical verification floor prevents reward-hacking (LLM can't fake passing tests)
- Good: severity calibration (critical/major/minor/suggestion) prevents perfectionism
- Good: iteration limits (5 SDD, 3 VDD) bound costs
- Bad: requires 4 subagent definitions (spec-builder, spec-critic, code-quality-enforcer, architecture-critic)
- Bad: Opus for all agents = higher cost per run
- Bad: custom methodology means no community ecosystem (unlike Spec Kit)

**Pros/cons for each option**: Full comparison table with spec format, multi-agent support, review/critique mechanism, test generation approach for all 6 options. Reference the research we did.

**Source**: VDD framework at `~/code/vdd/`

---

## ADR-0003: Sandbox Provider Selection

**Problem**: Open SWE requires an isolated sandbox per task. Must choose from: LangSmith (default), Daytona, Runloop, Modal, local, or custom.

**Decision Drivers**:
- POC: path of least resistance
- Must support full Linux environment (shell, git, test runners)
- Must allow parallel task execution

**Considered Options**:
1. LangSmith Sandboxes (default)
2. Daytona
3. Runloop
4. Modal
5. Local (dev only)

**Decision**: LangSmith Sandboxes (default) for POC — comes with the LangSmith ecosystem we're already using. Revisit for production (where GCP-native would be preferred).

**Consequences**:
- Good: zero additional setup (default provider)
- Good: integrated with LangSmith deployment and tracing
- Bad: another LangSmith dependency (vendor lock-in for POC)
- Bad: not GCP-native (production will need different provider)

---

## ADR-0004: POC Hosting on LangSmith Cloud

**Problem**: Need to host the LangGraph agent server somewhere. Options range from LangSmith Cloud (managed) to self-hosted on GCP.

**Decision Drivers**:
- POC: fastest path to working demo
- Production will be GCP (org is GCP-only shop)
- Free tier acceptable for POC

**Considered Options**:
1. LangSmith Cloud (Developer tier, free)
2. Self-hosted on GKE
3. Self-hosted on Cloud Run
4. LangSmith Enterprise (self-hosted, custom pricing)

**Decision**: LangSmith Cloud Developer tier for POC. Free, managed, fastest to working demo. Accept that agent runs on LangChain's infra (not GCP). Revisit for production.

**Consequences**:
- Good: zero infrastructure to manage
- Good: free tier (1 seat, 5k traces, 50 Fleet runs)
- Good: fastest path to demo
- Bad: agent runs outside our GCP (not representative of production)
- Bad: 50 runs/mo limit
- Bad: creates dependency on LangChain's platform availability

---

## Spec 001: VSDD Slack/GitHub Agent

Written in VDD canonical 5-section format. This is the spec that an implementation agent will build from.

**What it specifies**:
- A customized Open SWE agent that runs the VSDD pipeline
- Invocation via Slack thread or GitHub issue
- SDD phase: Slack-based iterative spec refinement with adversarial critique
- TDD phase: mechanical test generation from approved spec, Red Gate
- VDD phase: phased implementation with parallel critics and test gates
- PR creation with all changes linked to source thread/issue
- Four subagents mapped to Open SWE's `task` tool
- Middleware for Slack message injection mid-run

**Section 1 — Behavioral Contract**:
- Preconditions: valid invocation (Slack mention or GitHub @-mention), target repo accessible, GitHub App installed on repo, AGENTS.md present (optional), LLM API key configured
- Postconditions: approved spec written to sandbox, test suite generated and Red Gate passed, implementation complete with all phase tests passing, draft PR opened linking to source thread/issue
- Invariants: spec always in canonical 5-section format, each critique iteration uses fresh agent instance, test suite always run between VDD phases, severity calibration consistent across all critics

**Section 2 — Interface Definition**:
- Input: Slack message or GitHub issue comment containing feature description + repo reference
- Output: GitHub draft PR with spec file, test suite, implementation, linked to source
- Error types: insufficient description (ask for more), spec rejected after 5 iterations (escalate), Red Gate failure (escalate), test gate failure after 2 retries (escalate), sandbox unreachable (retry/escalate)

**Section 3 — Edge Case Catalog**:
- Ambiguous feature description (not enough to fill Sections 1-3)
- User abandons mid-SDD (no Slack reply for extended period)
- Red Gate fails (some tests pass — existing behavior or tautological test)
- Critics disagree on severity (code-quality says major, architecture says minor)
- Test gate fails after critique approved the code
- Target repo has no test framework configured
- Target repo uses unsupported language
- Sandbox times out during long test suite
- User sends follow-up message mid-implementation
- Multiple concurrent requests for same repo

**Section 4 — Non-Functional Requirements**:
- Performance: SDD iteration < 5 min, full pipeline < 60 min for typical feature
- Cost: bounded by iteration limits (5 SDD × 2 agents + 4 VDD phases × 3 iterations × 2 critics = max ~34 Opus calls)
- Security: GitHub App tokens scoped to installation, no secrets in Slack messages, sandbox isolated

**Section 5 — Verification Strategy**:
- Provable: spec format validated by parser (canonical sections present), test naming conventions match spec sections
- Testable: end-to-end pipeline produces PR from Slack message, critic findings include file:line references, test gates catch implementation regressions, Slack thread contains full audit trail

---

## Execution Order

1. Write ADR-0002 (spec methodology) — this is the meatiest one with the comparison table
2. Write ADR-0003 (sandbox provider) — straightforward
3. Write ADR-0004 (POC hosting) — straightforward
4. Write Spec 001 (VSDD agent) — full canonical 5-section format
5. Update README indexes

## Files to Create
- `docs/decisions/0002-spec-methodology.md`
- `docs/decisions/0003-sandbox-provider.md`
- `docs/decisions/0004-poc-hosting.md`
- `docs/specs/001-vsdd-slack-github-agent.md`

## Files to Modify
- `docs/decisions/README.md`
- `docs/specs/README.md`
