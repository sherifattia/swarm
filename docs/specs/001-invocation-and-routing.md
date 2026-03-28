# Specification: Invocation & Routing

> **Status**: Placeholder — needs full SDD treatment before TDD.

The entry point of the delegate agent. Receives raw webhook payloads from Slack and GitHub, validates signatures, parses them into structured invocation messages, checks user authorization, resolves the target repository, and either creates a new agent run or routes a follow-up message to an active run.

This is an Open SWE customization. Open SWE already provides webhook endpoints (`POST /webhooks/slack`, `POST /webhooks/github`) and `extract_repo_from_text()` for repo resolution. This spec covers the delegate-specific behavior layered on top: authorization policy, payload validation, description extraction, and follow-up routing.

## Scope

- Webhook signature validation (`GITHUB_WEBHOOK_SECRET`, `SLACK_SIGNING_SECRET`)
- Payload parsing: Slack @-mention events and GitHub `@delegate` issue/PR comments
- User authorization against `ALLOWED_GITHUB_ORGS` allowlist
- Repository resolution from message content:
  - `repo:owner/name` — explicit org and repo
  - `repo:name` — repo only, org defaults to `DEFAULT_REPO_OWNER`
  - `https://github.com/owner/name` — GitHub URL
  - Fallback to `DEFAULT_REPO_OWNER` / `DEFAULT_REPO_NAME` env vars
- GitHub App installation verification (Contents R/W, Pull Requests R/W, Issues R/W, Metadata R)
- Feature description extraction from message body
- Follow-up message routing: match incoming `thread_id` to active runs and inject via `check_message_queue_before_model` middleware
- Initial acknowledgment in source thread (< 30 seconds from webhook receipt)

## Key Interfaces

```
Input:  Raw webhook payload (Slack event JSON or GitHub webhook JSON)

Output: InvocationMessage {
          source:       "slack" | "github"
          thread_id:    string
          user_id:      string
          repo:         { owner: string, name: string }
          description:  string
          raw_message:  string
        }
        | FollowUpMessage { source, thread_id, user_id, content }
        | Silent ignore (invalid invocation or unauthorized user)
        | Error reply in thread (no repo, app not installed)
```

## Error Behaviors

| Condition | Behavior |
|---|---|
| Invalid webhook signature | Reject with 401, no response in thread |
| Message is not an @-mention targeting delegate | Ignore silently |
| User not in `ALLOWED_GITHUB_ORGS` | Ignore silently |
| No repo reference and no defaults configured | Reply in thread: "Please specify a target repo using `repo:owner/name`" |
| GitHub App not installed on target repo | Reply in thread: "The delegate GitHub App is not installed on {owner}/{name}. Please install it first." |
| Message matches an active run's `thread_id` | Route as `FollowUpMessage`, do not create new run |

## Edge Cases to Address

- Malformed `thread_id` (empty string, wrong format)
- Repo owner exists but repo name doesn't
- Message contains images/attachments with no text description
- Multiple concurrent requests for the same repo (each gets its own run — branch names must include unique identifier)
- GitHub URL points to an archived or read-only repo
- Slack message edited after invocation (re-parse or ignore?)

## Security Requirements

- Webhook signatures validated before any processing
- GitHub App tokens scoped to installation (never personal access tokens)
- No secrets in thread replies
- User authorization enforced before any work begins

## Open SWE Extension Points

- `agent/webapp.py` — webhook endpoint handlers
- `agent/utils/repo.py` — `extract_repo_from_text()` for repo parsing
- `agent/utils/github_user_email_map.py` — user mapping
- `agent/webapp.py` — `ALLOWED_GITHUB_ORGS` allowlist
