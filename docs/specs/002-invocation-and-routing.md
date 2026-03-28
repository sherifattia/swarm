# Specification: Invocation & Routing

> **Status**: Placeholder — needs full SDD treatment before TDD.
>
> **Parent**: [001 — VSDD Slack/GitHub Agent](001-vsdd-slack-github-agent.md)

The entry point of the VSDD pipeline. Receives raw webhook payloads from Slack and GitHub, validates them, resolves the target repository, checks authorization, and produces a validated `InvocationMessage` or an error response.

## Scope

- Webhook signature validation (Slack signing secret, GitHub webhook secret)
- Payload parsing for Slack @-mentions and GitHub `@delegate` comments
- User authorization against `ALLOWED_GITHUB_ORGS`
- Repository resolution from message content (`repo:owner/name`, `repo:name`, GitHub URL, defaults)
- GitHub App installation verification (permissions check on target repo)
- Feature description extraction from message body
- Follow-up message routing (matching `thread_id` to active runs)

## Key Interfaces

```
Input:  Raw webhook payload (Slack event or GitHub webhook JSON)
Output: InvocationMessage | FollowUpMessage | error response (silent ignore, thread reply, or no-op)
```

## Covered Parent Spec Elements

- Preconditions: P1, P3, P4, P7
- Error types: InvalidInvocationError, NoTargetRepoError, GitHubAppNotInstalledError, UnauthorizedUserError
- Edge cases: E1 (partial — description sufficiency check), E9 (follow-up routing), E10 (concurrent request branch naming)
