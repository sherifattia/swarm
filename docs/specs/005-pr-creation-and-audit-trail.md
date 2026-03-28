# Specification: PR Creation & Audit Trail

> **Status**: Placeholder — needs full SDD treatment before TDD.

The delivery layer of the delegate pipeline. Creates a draft pull request from the implementation artifacts in the sandbox and posts audit trail messages to the source Slack thread or GitHub issue throughout the entire pipeline run. This is not just an end-of-pipeline step — audit messages are posted during SDD, TDD, and VDD phases as they happen.

This builds on Open SWE's existing `commit_and_open_pr` tool and `open_pr_if_needed` safety-net middleware. The spec covers delegate-specific behavior: branch naming, PR body format, source linking, and the structured audit trail.

## Scope

### PR Creation
- Feature branch creation: `delegate/{feature-slug}-{short-id}` (unique ID prevents branch collisions for concurrent requests on the same repo)
- Commit assembly: spec file + test suite + implementation files
- Draft PR creation via GitHub App API:
  - Title: derived from spec Section 1 (Behavioral Contract)
  - Body: summary, link to spec file in the PR, test results summary, link to source thread/issue
  - Always draft (`draft: true`) — human review required before merge
  - Base branch: repo's default branch
- Source linking: PR body links to originating Slack thread or GitHub issue
- Safety net: `open_pr_if_needed` middleware auto-commits and opens PR if the agent completes without doing so

### Audit Trail
- Structured messages posted to source thread (Slack) or issue comments (GitHub) at each pipeline milestone:

| Phase | Events |
|---|---|
| **Start** | "Starting VSDD pipeline for {repo}. I'll write a spec, generate tests, and implement." |
| **SDD** | Iteration N started/completed, critic findings summary (by severity), spec approved/rejected |
| **TDD** | Test generation complete ({N} tests), Red Gate result (pass/fail with details) |
| **VDD** | Phase N started/completed, critic findings summary, test gate result, iterations used |
| **Delivery** | PR created: {link}. Summary of what was built and test results. |
| **Error** | Pipeline stopped: {error type and message}. What the user can do next. |

- Message formatting adapted to destination (Slack markdown vs. GitHub markdown)
- Thread continuity: all messages in the same Slack thread or GitHub issue

### Follow-up Message Handling
- Messages arriving mid-pipeline are injected via `check_message_queue_before_model` middleware
- If the message is a correction/clarification: agent adjusts current phase
- If the message is a question: agent replies in thread and continues
- Audit trail records that a follow-up was received and how it affected the run

## Key Interfaces

```
Input (PR):     Implementation artifacts in sandbox (spec file, test files, implementation files)
                + pipeline metadata (source thread/issue URL, repo ref, feature name)

Output (PR):    Draft pull request on target repo
                { repo, branch, title, body, draft: true, linked_source, files: { spec, tests, implementation } }

Input (Audit):  Pipeline events (phase transitions, critic findings, test results, errors)
                + destination (slack_thread | github_comment) + thread_id

Output (Audit): Formatted messages posted to source thread/issue
```

## Error Behaviors

| Condition | Behavior |
|---|---|
| GitHub App lacks PR creation permission on target repo | Reply in thread: "Unable to create PR — GitHub App missing Pull Requests write permission on {repo}." |
| Branch name collision (extremely unlikely with unique ID) | Append timestamp suffix to branch name |
| Audit message fails to post (Slack/GitHub API error) | Log warning, continue pipeline. Audit trail is best-effort, not a blocking dependency. |
| Agent completes without creating PR | `open_pr_if_needed` middleware auto-commits and opens draft PR (safety net) |

## Edge Cases to Address

- Implementation produces no file changes (e.g., spec-only or test-only PR) — still create PR with available artifacts
- Multiple concurrent requests for the same repo — each gets its own branch with unique ID, no conflicts at git level
- Slack thread exceeds message limit — switch to thread summary with link to full log
- GitHub issue already has many comments — keep audit messages concise to avoid noise
- Feature request came from GitHub but PR is on the same repo — link PR to issue via `Closes #N` or `Related to #N`

## Security Requirements

- No secrets (API keys, tokens, credentials) in PR body, commit messages, or audit trail messages
- GitHub App tokens scoped to installation
- PR never targets default branch directly — always a feature branch with draft status
- Sandbox isolation: agent never pushes from outside the sandbox

## Open SWE Extension Points

- `agent/tools/` — `commit_and_open_pr` tool (existing)
- `agent/server.py` — `open_pr_if_needed` middleware (existing)
- `agent/server.py` — `check_message_queue_before_model` middleware (existing)
- `agent/tools/` — `slack_thread_reply` tool (existing)

## Performance Bounds

| Metric | Bound |
|---|---|
| PR creation (commit + push + API call) | < 2 minutes |
| Audit message posting | < 5 seconds per message |
| Webhook-to-first-acknowledgment | < 30 seconds |
