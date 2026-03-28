# Specification: VSDD Slack/GitHub Agent

A customized Open SWE agent that runs the VSDD pipeline (SDD → TDD → VDD) to turn feature requests — received via Slack thread or GitHub issue — into tested draft pull requests.

> **This is the system-level spec.** It defines the end-to-end contract but is not directly TDD-ready — postconditions are system outcomes, not function-level behaviors. The sub-specs below decompose it into modules with clear input→output interfaces that TDD can mechanically derive tests from.

### Sub-Specs

| Spec | Module | Input → Output |
|------|--------|----------------|
| [002](002-invocation-and-routing.md) | Invocation & Routing | Raw webhook → validated `InvocationMessage` or error |
| [003](003-sdd-pipeline.md) | SDD Pipeline | Feature description → approved canonical spec |
| [004](004-tdd-pipeline.md) | TDD Pipeline | Approved spec → test suite + stubs + Red Gate result |
| [005](005-vdd-pipeline.md) | VDD Pipeline | Plan + tests → implementation (all tests passing) |
| [006](006-delivery.md) | Delivery | Implementation artifacts → draft PR + audit trail |

---

## 1. Behavioral Contract

### 1.1 Preconditions

| # | Precondition | Error |
|---|---|---|
| P1 | Invocation is a valid Slack @-mention or GitHub issue/PR `@delegate` comment | `InvalidInvocationError` |
| P2 | Message contains a feature description with enough detail to fill Sections 1-3 of the canonical spec, OR the agent can elicit sufficient detail via follow-up questions | `InsufficientDescriptionError` |
| P3 | Message contains a target repository reference (`repo:owner/name`, `repo:name`, or GitHub URL) or defaults are configured (`DEFAULT_REPO_OWNER`, `DEFAULT_REPO_NAME`) | `NoTargetRepoError` |
| P4 | GitHub App is installed on the target repository with required permissions (Contents R/W, Pull Requests R/W, Issues R/W, Metadata R) | `GitHubAppNotInstalledError` |
| P5 | LLM API key is configured and the model is reachable | `LLMUnavailableError` |
| P6 | Sandbox provider is configured and reachable (`SANDBOX_TYPE` + credentials) | `SandboxUnavailableError` |
| P7 | Invoking user is in the allowed GitHub orgs list (`ALLOWED_GITHUB_ORGS`) or no allowlist is configured | `UnauthorizedUserError` |

### 1.2 Postconditions

| # | Postcondition |
|---|---|
| Q1 | An approved spec file exists in the sandbox in canonical 5-section format (Behavioral Contract, Interface Definition, Edge Case Catalog, Non-Functional Requirements, Verification Strategy) |
| Q2 | A test suite exists in the sandbox with tests mechanically derived from the spec: postconditions → unit tests, preconditions → error tests, invariants → property-based tests, edge cases → dedicated tests, interfaces → integration tests |
| Q3 | The Red Gate passed: all generated tests failed against minimal stubs before implementation began |
| Q4 | Implementation is complete and all phase tests pass (unit, integration, edge case, property-based) |
| Q5 | A draft pull request is open on the target repository linking to the source Slack thread or GitHub issue |
| Q6 | The PR includes: spec file, test suite, and implementation changes |
| Q7 | The source Slack thread or GitHub issue contains a full audit trail of the pipeline execution (SDD iterations, TDD results, VDD phase completions, critic findings) |

### 1.3 Invariants

| # | Invariant |
|---|---|
| I1 | The spec is always in canonical 5-section format — deviations are rejected by the spec-critic |
| I2 | Each critique iteration uses a fresh agent instance (no context carryover from previous critique) |
| I3 | The test suite is always run between VDD phases — no phase proceeds without a passing test gate |
| I4 | Severity calibration is consistent across all critics: critical (must fix), major (should fix), minor (can defer), suggestion (optional) |
| I5 | Iteration counts never exceed limits: 5 for SDD, 3 per VDD phase |
| I6 | Every precondition violation (P1-P7) produces the corresponding named error, not a generic failure |
| I7 | Sandbox isolation is maintained: one sandbox per task, no cross-contamination between concurrent requests |
| I8 | The agent never pushes directly to the default branch — all changes go through a draft PR on a feature branch |

---

## 2. Interface Definition

### 2.1 Input Types

**Primary input: Invocation message**

```
InvocationMessage {
  source:       "slack" | "github"
  thread_id:    string            // Slack thread_ts or GitHub issue/PR number
  user_id:      string            // Slack user ID or GitHub username
  repo:         RepoRef           // Parsed from message content or defaults
  description:  string            // Feature description (free-form text)
  raw_message:  string            // Full original message
}

RepoRef {
  owner:  string                  // GitHub org/user
  name:   string                  // Repository name
}
```

**Secondary input: Follow-up messages (mid-run)**

```
FollowUpMessage {
  source:       "slack" | "github"
  thread_id:    string            // Must match an active run's thread_id
  user_id:      string
  content:      string            // Additional context, clarifications, or corrections
}
```

Follow-up messages are injected into the agent's context via `check_message_queue_before_model` middleware before the next model call.

### 2.2 Output Types

**Primary output: Draft pull request**

```
PullRequest {
  repo:           RepoRef
  branch:         string          // Feature branch name (e.g., "delegate/feature-name")
  title:          string          // PR title derived from spec Section 1
  body:           string          // PR body with: summary, spec link, test results, audit trail link
  draft:          true            // Always draft — human review required
  linked_source:  string          // URL to source Slack thread or GitHub issue
  files: {
    spec:         string          // Path to spec file in repo (e.g., "docs/specs/NNN-feature.md")
    tests:        string[]        // Paths to generated test files
    implementation: string[]      // Paths to implementation files
  }
}
```

**Secondary output: Audit trail messages**

```
AuditMessage {
  destination:  "slack_thread" | "github_comment"
  thread_id:    string
  phase:        "sdd" | "tdd" | "vdd"
  content:      string            // Phase status, critic findings, test results
}
```

### 2.3 Error Types

| Error | Trigger | Response |
|---|---|---|
| `InvalidInvocationError` | Message is not a valid @-mention or does not target the bot | Ignore silently (do not reply to unrelated messages) |
| `InsufficientDescriptionError` | Feature description cannot fill Sections 1-3 after 2 clarification attempts | Reply in thread: "I need more detail to write a spec. Please describe: [specific missing elements]" |
| `NoTargetRepoError` | No repo reference in message and no defaults configured | Reply in thread: "Please specify a target repo using `repo:owner/name`" |
| `GitHubAppNotInstalledError` | GitHub App not installed on target repo or missing permissions | Reply in thread: "The delegate GitHub App is not installed on {owner}/{name}. Please install it first." |
| `LLMUnavailableError` | LLM API returns errors or is unreachable | Reply in thread: "I'm unable to reach the LLM API. Please try again later." Retry up to 3 times with exponential backoff before escalating. |
| `SandboxUnavailableError` | Sandbox cannot be created or becomes unreachable | Auto-recreate sandbox (Open SWE default behavior). If recreation fails 3 times, reply in thread: "Sandbox is unavailable. Please check the sandbox provider configuration." |
| `UnauthorizedUserError` | User not in allowed orgs list | Ignore silently |
| `SpecRejectedError` | Spec-critic finds critical issues after 5 SDD iterations | Reply in thread: "Spec could not be approved after 5 iterations. Last critic findings: [summary]. Please refine the description and try again." |
| `RedGateFailureError` | Some generated tests pass against stubs (indicating tautological tests or pre-existing behavior) | Reply in thread: "Red Gate failed — {N} tests passed against stubs. These tests may be tautological. Flagging for human review: [test names]" |
| `TestGateFailureError` | Tests fail after VDD phase implementation, not resolved after 3 critique iterations | Reply in thread: "Tests are failing after implementation and 3 rounds of critique could not resolve: [failing tests]. Flagging for human review." |
| `SandboxTimeoutError` | Sandbox operation exceeds timeout (test suite, build, etc.) | Reply in thread: "Sandbox operation timed out after {duration}. The test suite or build may be too large for the current sandbox configuration." |

---

## 3. Edge Case Catalog

| # | Edge Case | Expected Behavior |
|---|---|---|
| E1 | **Ambiguous feature description** — not enough detail to fill Sections 1-3 of canonical spec | Agent asks up to 2 clarifying questions in the source thread. If still insufficient after 2 rounds, raises `InsufficientDescriptionError` with specific missing elements. |
| E2 | **User abandons mid-SDD** — no Slack reply for >30 minutes during clarification | Agent waits. Run remains active (LangGraph durable execution). If user replies later, `check_message_queue_before_model` middleware injects the reply and the agent resumes. No timeout-based termination. |
| E3 | **Red Gate fails** — some tests pass against stubs | Raises `RedGateFailureError`. Lists the passing test names. Does not proceed to implementation. Requires human review to determine if tests are tautological or if stubs inadvertently implement behavior. |
| E4 | **Critics disagree on severity** — code-quality-enforcer rates an issue as major, architecture-critic rates it as minor | Use the higher severity. Both findings are included in the audit trail. The consolidation step merges findings by file:line, taking the maximum severity when the same location is flagged by both critics. |
| E5 | **Test gate fails after critique approved** — critics find no critical/major issues but tests still fail | This indicates the critics missed a real bug. Re-enter critique loop with the test failure output appended to the critic prompt as additional context. Count this as a critique iteration. If max iterations (3) exhausted, raise `TestGateFailureError`. |
| E6 | **Target repo has no test framework configured** | Agent detects language from repo files and installs the default test framework for that language (pytest for Python, jest/vitest for TypeScript/JS, go test for Go, cargo test for Rust). If the language is unsupported, raises `UnsupportedLanguageError` and reports in thread. |
| E7 | **Target repo uses unsupported language** — language not in VDD's supported set (Python, TypeScript/JS, Go, Rust, Java/Kotlin) | Reply in thread: "The target repo appears to use {language}, which is not currently supported. Supported languages: Python, TypeScript/JavaScript, Go, Rust, Java, Kotlin." Do not proceed. |
| E8 | **Sandbox times out during long test suite** | Raises `SandboxTimeoutError`. Reports the timeout duration and which operation timed out. Does not retry automatically — the test suite may need optimization or the sandbox may need more resources. |
| E9 | **User sends follow-up message mid-implementation** | `check_message_queue_before_model` middleware injects the message before the next model call. Agent reads it and adjusts if the message contains corrections or new requirements. If the message is a question, agent replies in thread and continues. |
| E10 | **Multiple concurrent requests for the same repo** | Each request runs in its own sandbox with its own branch. No conflict at the sandbox level. PR branch names include a unique identifier to prevent branch collisions. If two PRs touch the same files, GitHub's merge conflict detection handles it at review time. |
| E11 | **Spec-builder produces spec missing required sections** | Spec-critic catches this as a critical finding ("missing Section N"). SDD loop continues. If still missing after 5 iterations, raises `SpecRejectedError`. |
| E12 | **LLM rate limiting during parallel critic execution** | Retry with exponential backoff (standard LLM client behavior). If rate limiting persists, serialize critic execution (code-quality-enforcer first, then architecture-critic) rather than running them in parallel. |
| E13 | **Repo has existing tests that conflict with generated tests** | Generated tests use VDD naming conventions (`test_postcondition_*`, `test_precondition_violation_*`, etc.) that are unlikely to collide. If a collision occurs, the agent prefixes generated test names with `vdd_`. Existing tests are never modified or deleted. |
| E14 | **Feature request duplicates existing functionality** | The agent does not check for duplicates — this is a human judgment at spec review time. The spec-critic may flag overlap if it is apparent from the spec content, but this is not guaranteed. |

---

## 4. Non-Functional Requirements

### 4.1 Performance Bounds

| Metric | Bound | Notes |
|---|---|---|
| SDD iteration (single spec-builder + spec-critic round) | < 5 minutes | Dominated by LLM latency (2 Opus calls per iteration) |
| Full SDD phase (up to 5 iterations) | < 25 minutes | Worst case: 5 iterations × 5 min |
| TDD phase (test generation + Red Gate) | < 10 minutes | Single pass: generate tests, create stubs, run tests |
| VDD phase per implementation phase (up to 3 critique iterations) | < 20 minutes | Implementation + 2 parallel critics + test gate per iteration |
| Full VSDD pipeline (typical feature) | < 60 minutes | Typical: 3 SDD iterations + TDD + 3 VDD phases × 1-2 iterations |
| Full VSDD pipeline (worst case) | < 120 minutes | Max: 5 SDD + TDD + 4 VDD phases × 3 iterations |
| Webhook-to-first-response (acknowledgment in thread) | < 30 seconds | Agent sends initial acknowledgment before starting pipeline |
| Follow-up message injection latency | < 1 model call | Message injected before the very next model invocation |

### 4.2 Memory Constraints

| Constraint | Bound |
|---|---|
| Sandbox disk | Provider-determined (LangSmith default) |
| Agent context window | Managed by Deep Agents auto-summarization — no manual context management needed |
| Concurrent runs | Limited by LangSmith Cloud tier (50 Fleet runs/month for POC) |

### 4.3 Security Requirements

| # | Requirement |
|---|---|
| S1 | GitHub App tokens are scoped to the installation (one repo or org) — never use a personal access token |
| S2 | No secrets (API keys, tokens, credentials) are included in Slack messages, GitHub comments, or PR bodies |
| S3 | Sandbox is isolated — agent code in the sandbox cannot access the host, other sandboxes, or the LangGraph server's environment |
| S4 | User authorization is enforced via `ALLOWED_GITHUB_ORGS` before any work begins |
| S5 | Webhook signatures are validated (`GITHUB_WEBHOOK_SECRET`, `SLACK_SIGNING_SECRET`) before processing any event |
| S6 | The `TOKEN_ENCRYPTION_KEY` encrypts secrets at rest in LangGraph state |
| S7 | The agent never executes arbitrary code from user messages outside the sandbox — all shell commands run in the isolated sandbox environment |

---

## 5. Verification Strategy

### 5.1 Provable Properties

| # | Property | How to Prove |
|---|---|---|
| V1 | Spec is in canonical 5-section format | Parser validation: check that all 5 top-level sections and their required subsections exist in the spec file |
| V2 | Test naming conventions match spec sections | Static analysis: every test function name matches one of the VDD naming patterns (`test_postcondition_*`, `test_precondition_violation_*`, `test_invariant_*`, `test_edge_case_*`, `test_integration_*`, `test_perf_*`, `test_memory_*`, `test_security_*`, `test_property_*`) |
| V3 | Iteration counts are within limits | Counter check: SDD iterations <= 5, VDD iterations per phase <= 3 |
| V4 | PR is always draft | GitHub API assertion: `draft: true` in the PR creation call |
| V5 | Each critique uses a fresh agent instance | Architecture assertion: subagent spawned via `task` tool (Open SWE creates a new agent with its own sandbox, middleware, and conversation history per `task` call) |

### 5.2 Testable Properties

| # | Property | Test Approach |
|---|---|---|
| T1 | End-to-end pipeline produces a PR from a Slack message | Integration test: send a Slack webhook payload with a feature description targeting a test repo; assert a draft PR is created with spec, tests, and implementation |
| T2 | End-to-end pipeline produces a PR from a GitHub issue comment | Integration test: send a GitHub webhook payload with `@delegate` comment; assert same as T1 |
| T3 | Critic findings include file:line references | Unit test: run code-quality-enforcer and architecture-critic on a known-bad codebase; assert findings contain `file:line` references |
| T4 | Test gates catch implementation regressions | Integration test: introduce a deliberate bug after a passing VDD phase; assert the test gate fails and the agent enters the critique loop |
| T5 | Slack thread contains full audit trail | Integration test: complete a pipeline run; assert the Slack thread contains messages for each phase (SDD start/complete, TDD results, VDD phase completions, critic findings) |
| T6 | Red Gate rejects passing tests | Unit test: generate tests against a module with pre-existing implementation; assert Red Gate fails and reports which tests passed |
| T7 | Follow-up messages are injected mid-run | Integration test: start a pipeline run, send a follow-up message via Slack webhook; assert the message appears in the agent's conversation before the next model call |
| T8 | Concurrent requests for the same repo use separate sandboxes and branches | Integration test: send two invocations targeting the same repo simultaneously; assert two separate draft PRs are created on different branches |
| T9 | Unauthorized users are rejected silently | Unit test: send a webhook from a user not in `ALLOWED_GITHUB_ORGS`; assert no response and no agent run created |
| T10 | Insufficient description triggers clarification | Integration test: send a vague one-word feature description; assert the agent replies asking for more detail rather than generating a spec |

### 5.3 Verification Constraints

| # | Constraint |
|---|---|
| C1 | Integration tests (T1, T2, T4, T5, T7, T8, T10) require a running LangGraph server, sandbox provider, and test GitHub/Slack accounts — they cannot run in CI without infrastructure |
| C2 | End-to-end tests (T1, T2) incur LLM costs (~34 Opus calls worst case per run) — run sparingly, not on every commit |
| C3 | Performance bounds (Section 4.1) are LLM-latency-dominated and will vary with model provider load — treat as guidelines, not hard SLAs |
| C4 | Concurrent request testing (T8) requires the sandbox provider to support parallel sandbox creation — local provider cannot verify this |
