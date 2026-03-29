# Open SWE Developer Reference

Technical reference for delegate's agent harness. See [ADR-0001](decisions/0001-agent-harness.md) for the selection rationale.

**Sources**: [README](https://github.com/langchain-ai/open-swe), [Blog Post](https://blog.langchain.com/open-swe-an-open-source-framework-for-internal-coding-agents/), [Installation](https://github.com/langchain-ai/open-swe/blob/main/INSTALLATION.md), [Customization](https://github.com/langchain-ai/open-swe/blob/main/CUSTOMIZATION.md)

---

## Architecture Overview

Open SWE is a **single Deep Agent** (not three separate graphs) with subagent spawning, middleware hooks, and a curated toolset. It is composed on the [Deep Agents](https://github.com/langchain-ai/deepagents) framework and runs on [LangGraph](https://github.com/langchain-ai/langgraph).

### Deployment Topology

```
┌─────────────────────────────────────────────────────┐
│  Invocation Surfaces                                │
│  ┌─────────┐  ┌─────────┐  ┌──────────────────┐    │
│  │  Slack   │  │ Linear  │  │ GitHub (App)     │    │
│  └────┬────┘  └────┬────┘  └────────┬─────────┘    │
└───────┼────────────┼────────────────┼───────────────┘
        │            │                │
        ▼            ▼                ▼
┌─────────────────────────────────────────────────────┐
│  LangGraph Server (port 2024)                       │
│                                                     │
│  Webhook Endpoints:                                 │
│    POST /webhooks/slack                             │
│    POST /webhooks/linear                            │
│    POST /webhooks/github                            │
│    GET  /health                                     │
│                                                     │
│  Agent: create_deep_agent()                         │
│    ├── Tools (~15 curated)                          │
│    ├── Middleware pipeline                           │
│    ├── Subagent spawning (task tool)                │
│    └── write_todos (planning)                       │
│                                                     │
│  langgraph.json:                                    │
│    graphs.agent → agent.server:get_agent            │
│    http.app    → agent.webapp:app                   │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│  Sandbox (one per task, isolated)                   │
│  Providers: LangSmith │ Modal │ Daytona │ Runloop   │
│             Local (dev only) │ Custom                │
│                                                     │
│  - Full Linux environment with shell access         │
│  - Repo cloned into sandbox                         │
│  - Persistent per thread, auto-recreate if lost     │
│  - Parallel execution across tasks                  │
└─────────────────────────────────────────────────────┘
```

### GitHub App Requirement

Open SWE **requires a GitHub App** for its GitHub integration. There is no workflow-file or GitHub Action alternative. The GitHub App provides:

- **Webhook delivery**: GitHub sends events to your server's `/webhooks/github` endpoint
- **OAuth**: User authentication for the web dashboard
- **API access**: PR creation, issue comments, branch management via installation tokens
- **Required permissions**: Contents (R/W), Pull Requests (R/W), Issues (R/W), Metadata (R)
- **Subscribed events**: Issue comment, PR review, PR review comment

This is a fundamental architectural constraint — see follow-up ADR-0002.

---

## Seven Components

### 1. Agent Harness (Deep Agents)

Open SWE composes on [Deep Agents](https://github.com/langchain-ai/deepagents) rather than forking an existing agent. This provides an upgrade path: improvements to Deep Agents (context compression, prompt caching, planning efficiency) flow downstream automatically.

```python
create_deep_agent(
    model="anthropic:claude-opus-4-6",
    system_prompt=construct_system_prompt(repo_dir, ...),
    tools=[http_request, fetch_url, commit_and_open_pr, linear_comment, slack_thread_reply],
    backend=sandbox_backend,
    middleware=[ToolErrorMiddleware(), check_message_queue_before_model, ...]
)
```

Deep Agents provides:
- **Planning**: `write_todos` tool for structured task breakdown and progress tracking
- **File operations**: `read_file`, `write_file`, `edit_file`, `ls`, `glob`, `grep`
- **Shell access**: `execute` for running commands in the sandbox
- **Subagent spawning**: `task` tool delegates subtasks to isolated child agents
- **Auto-summarization**: Manages context window by summarizing long conversations
- **AGENTS.md**: Convention for repo-level agent configuration (injected into system prompt)

### 2. Sandbox (Pluggable)

Each task runs in a dedicated, isolated cloud sandbox — a remote Linux environment with full shell access inside strict boundaries. The sandbox provider is **pluggable** via the `SANDBOX_TYPE` environment variable:

| Provider | Env Var | Notes |
|----------|---------|-------|
| `langsmith` | `LANGSMITH_API_KEY_PROD` | Default. LangSmith cloud sandboxes. |
| `daytona` | `DAYTONA_API_KEY` | Daytona API. |
| `runloop` | `RUNLOOP_API_KEY` | Runloop API. |
| `modal` | Modal credentials | Modal containers. |
| `local` | — | Development only, no isolation. |
| Custom | — | Implement `SandboxBackendProtocol`. |

**Key behaviors**:
- One sandbox per task — no cross-contamination
- Persistent per conversation thread, reused across follow-up messages
- Auto-recreates if sandbox becomes unreachable
- Multiple tasks run in parallel, each in its own sandbox
- Repository is cloned into the sandbox at startup

**Custom provider**: Implement a factory function in `agent/integrations/my_provider.py` returning an object that implements `SandboxBackendProtocol`. Register it in `agent/utils/sandbox.py` by adding to `SANDBOX_FACTORIES`. Extend `BaseSandbox` for simpler implementations — only override `execute()`.

### 3. Tools (~15 Curated)

Philosophy: "tool curation matters more than tool quantity."

**Open SWE custom tools**:

| Tool | Purpose |
|------|---------|
| `execute` | Shell commands in sandbox |
| `fetch_url` | Fetch web pages as markdown |
| `http_request` | API calls (GET, POST, etc.) |
| `commit_and_open_pr` | Git commit + GitHub draft PR creation |
| `linear_comment` | Post updates to Linear tickets |
| `slack_thread_reply` | Reply in Slack threads |

**Deep Agents built-in tools**: `read_file`, `write_file`, `edit_file`, `ls`, `glob`, `grep`, `write_todos`, `task`

**Adding tools**: Create a function in `agent/tools/my_tool.py` (function name = tool name, docstring = description), then add to the tools list in `get_agent()`.

**Conditional tools**: Vary the toolset based on trigger source. Extract `source` from `config["configurable"].get("source")` and conditionally include/exclude communication tools.

**Removing tools**: Exclude from the tools list passed to `create_deep_agent()`.

### 4. Context Engineering

Two layers of context are assembled before the agent starts:

**Layer 1 — Repository context (`AGENTS.md`)**:
- File at the repository root, read from sandbox and injected into system prompt
- Encodes: coding conventions, testing requirements, architectural decisions, team-specific patterns
- Equivalent to Stripe's rule files

**Layer 2 — Task context (pre-assembled)**:
- Full Linear issue (title, description, comments) OR full Slack thread history OR GitHub issue/PR context
- Passed to the agent before it starts work
- Reduces tool-call discovery overhead — agent begins with complete requirements

System prompt is modular, assembled in `agent/prompt.py`.

### 5. Orchestration (Subagents + Middleware)

**Subagents** — spawned via the `task` tool:
- Each subagent gets: own isolated sandbox, own middleware stack, own todo list, own conversation history
- Main agent fans out independent subtasks to subagents
- Parallel execution; no context cross-pollution between subagents

**Middleware pipeline** — deterministic hooks that run around the agent loop:

| Middleware | Purpose |
|------------|---------|
| `ToolErrorMiddleware()` | Catches and handles tool errors gracefully |
| `check_message_queue_before_model` | Injects follow-up messages (Linear comments, Slack messages arriving mid-run) before next model call. Enables "message it while it's running." |
| `open_pr_if_needed` | Safety net: auto-commits and opens PR if agent didn't. Runs after agent completes. |
| `ensure_no_empty_msg` | Message validation |

Add custom middleware in `agent/server.py` by appending to the middleware list in `create_deep_agent()`.

### 6. Invocation Surfaces

Three built-in trigger surfaces, all producing deterministic thread IDs for follow-up routing:

| Surface | Trigger | How It Works |
|---------|---------|-------------|
| **Slack** | Mention bot in any thread | Bot replies in-thread with status and PR links. Use `repo:owner/name` syntax to specify target repo. |
| **Linear** | Comment `@openswe` on any issue | Agent reacts with 👀, reads full issue context, posts results as comments. Team-to-repo mapping in `agent/utils/linear_team_repo_map.py`. |
| **GitHub** | Tag `@openswe` in issue/PR comments | Addresses review feedback, pushes fixes to same branch. Requires GitHub App webhook. |

**Repository routing**:
- `repo:owner/name` — explicit org and repo
- `repo:name` — repo only; org defaults to `DEFAULT_REPO_OWNER`
- `https://github.com/owner/name` — GitHub URL
- Parsing handled by `extract_repo_from_text()` in `agent/utils/repo.py`
- Fallback: `DEFAULT_REPO_OWNER` / `DEFAULT_REPO_NAME` env vars

**User restrictions**: Edit `agent/utils/github_user_email_map.py` to map GitHub usernames to emails. Org allowlist via `ALLOWED_GITHUB_ORGS` in `agent/webapp.py`.

**Adding new triggers**: Create a webhook endpoint in `agent/webapp.py`, parse the event, create a LangGraph run via `langgraph_client.runs.create(thread_id, "agent", input=..., config=...)`.

### 7. Validation

**Prompt-driven**: Agent is instructed to run linters, formatters, and tests before committing.

**Safety net**: `open_pr_if_needed` middleware automatically commits and opens a PR if the agent finishes without doing so.

**CI integration**: `SKIP_CI_UNTIL_LAST_COMMIT` env var — when `true`, CI only runs on the final commit (cost control during multi-commit agent runs).

**Extensible**: Add deterministic CI checks, visual verification, or review gates as additional middleware.

---

## Infrastructure Requirements

| Component | Required | Purpose |
|-----------|----------|---------|
| **GitHub App** | Yes | Webhooks, OAuth, PR/issue API access |
| **LangGraph Server** | Yes | Agent runtime (durable execution, streaming, checkpointing) |
| **Sandbox provider** | Yes | Isolated execution environments |
| **LLM API key** | Yes | At least one (Anthropic default) |
| **LangSmith** | Yes | Tracing, optional OAuth, optional sandbox provider |
| **ngrok** | Dev only | Exposes local webhooks to internet |

**Hosting options**:
- **Development**: Local LangGraph server (`uv run langgraph dev`) + ngrok tunnel
- **Production**: [LangGraph Cloud](https://smith.langchain.com) (managed by LangChain)
- **Self-hosted**: Any server running `langgraph` with proper webhook URLs

---

## Environment Variables

### Required

| Variable | Purpose |
|----------|---------|
| `GITHUB_APP_ID` | Numeric GitHub App ID |
| `GITHUB_APP_PRIVATE_KEY` | PEM private key for JWT signing |
| `GITHUB_APP_INSTALLATION_ID` | Installation ID (from URL after installing app) |
| `GITHUB_WEBHOOK_SECRET` | Shared secret for webhook validation (`openssl rand -hex 32`) |
| `LANGSMITH_API_KEY_PROD` | LangSmith API key |
| `LANGSMITH_TENANT_ID_PROD` | LangSmith tenant UUID |
| `LANGSMITH_TRACING_PROJECT_ID_PROD` | LangSmith tracing project ID |
| `ANTHROPIC_API_KEY` | Anthropic API key (default LLM provider) |
| `TOKEN_ENCRYPTION_KEY` | Encryption key for secrets (`openssl rand -base64 32`) |

### Optional

| Variable | Purpose |
|----------|---------|
| `OPENAI_API_KEY` | OpenAI models |
| `GOOGLE_API_KEY` | Google models |
| `LLM_MODEL_ID` | Override default model (format: `provider:model_name`) |
| `SANDBOX_TYPE` | Sandbox provider (`langsmith`, `daytona`, `runloop`, `modal`, `local`) |
| `DEFAULT_SANDBOX_TEMPLATE_NAME` | Custom sandbox template name |
| `DEFAULT_SANDBOX_TEMPLATE_IMAGE` | Custom sandbox Docker image |
| `DEFAULT_REPO_OWNER` | Default GitHub org for repo routing |
| `DEFAULT_REPO_NAME` | Default repo name for repo routing |
| `GITHUB_OAUTH_PROVIDER_ID` | OAuth provider ID string |
| `ALLOWED_GITHUB_ORGS` | Comma-separated org allowlist |
| `LINEAR_API_KEY` | Linear API key (enables Linear trigger) |
| `LINEAR_WEBHOOK_SECRET` | Linear webhook secret |
| `SLACK_BOT_TOKEN` | Slack bot OAuth token (enables Slack trigger) |
| `SLACK_BOT_USER_ID` | Slack bot user ID |
| `SLACK_BOT_USERNAME` | Slack bot display name |
| `SLACK_SIGNING_SECRET` | Slack signing secret |
| `EXA_API_KEY` | Exa web search |
| `SKIP_CI_UNTIL_LAST_COMMIT` | Only run CI on final commit |
| `LANGCHAIN_TRACING_V2` | Enable LangChain tracing (`true`) |
| `LANGCHAIN_PROJECT` | LangChain tracing project name |

---

## Customization Points

| Component | Location | How to Customize |
|-----------|----------|-----------------|
| Sandbox provider | `agent/integrations/` | Implement `SandboxBackendProtocol`, register in `agent/utils/sandbox.py` |
| Model | `agent/utils/model.py` | `LLM_MODEL_ID` env var or pass model instance to `create_deep_agent()` |
| Tools | `agent/tools/` | Add functions, register in tools list in `get_agent()` |
| Triggers | `agent/webapp.py` | Add webhook endpoints |
| System prompt | `agent/prompt.py` | Edit prompt sections; provide `AGENTS.md` in target repo |
| Repo mapping | `agent/utils/linear_team_repo_map.py` | Update `LINEAR_TEAM_TO_REPO` dictionary |
| Middleware | `agent/server.py` | Add to middleware list in `create_deep_agent()` |
| User access | `agent/utils/github_user_email_map.py` | Map GitHub usernames to emails |

For full customization details, see the upstream [Customization Guide](https://github.com/langchain-ai/open-swe/blob/main/CUSTOMIZATION.md).

---

## How delegate Maps to Open SWE

| delegate Concept | Open SWE Component |
|-----------------|-------------------|
| Feature spec (`docs/specs/NNN-*.md`) | Task input (issue body / message content) |
| "Hand off specs" | Invocation surface (Slack / Linear / GitHub) |
| Agent implements the spec | Deep Agent with `write_todos` planning + `task` subagents |
| Isolated execution | Sandbox (pluggable provider) |
| "Get back tested PRs" | `commit_and_open_pr` tool + `open_pr_if_needed` middleware |
| Human review gate | PR review on GitHub (mandatory per Stripe's finding: AI code has 1.75x more logic errors) |

**Open questions** (to be addressed in future ADRs):
- How do specs from `docs/specs/` enter the agent pipeline? Manual issue creation? Automated feed?
- Which invocation surfaces do we enable? (Slack, Linear, GitHub, or all three?)
- Which sandbox provider do we use?
- How do we feed repo-specific context (AGENTS.md, conventions) to the agent?
- Do we need custom middleware for our validation requirements?

---

## Production Patterns (Context)

Open SWE codifies patterns from three independent production systems:

| Aspect | Stripe (Minions) | Ramp (Inspect) | Coinbase (Cloudbot) | Open SWE |
|--------|------------------|----------------|---------------------|----------|
| **Harness** | Forked Goose | Composed on OpenCode | Built from scratch | Composed on Deep Agents |
| **Sandbox** | AWS EC2 devboxes | Modal containers | In-house | Pluggable (5 providers) |
| **Tools** | ~500, curated | OpenCode SDK + extensions | MCPs + custom Skills | ~15, curated |
| **Invocation** | Slack + buttons | Slack + web + Chrome | Slack-native | Slack, Linear, GitHub |
| **Validation** | 3-layer + 1 retry | Visual DOM verification | Agent councils + auto-merge | Prompt-driven + PR safety net |

All four converged on: **isolated sandboxes, curated toolsets, chat-first invocation, rich context at startup, and subagent orchestration**.

---

## Related Documents

- [ADR-0001: Agent Harness Selection](decisions/0001-agent-harness.md)
- ADR-0002: Deployment Model (to be written — GitHub App confirmed required)
- ADR-0003: Sandbox Provider Selection (to be written)
- ADR-0004: Invocation Surface Selection (to be written)
- [Spec Conventions](specs/README.md)

## Upstream References

- [Open SWE Repository](https://github.com/langchain-ai/open-swe)
- [Open SWE Installation Guide](https://github.com/langchain-ai/open-swe/blob/main/INSTALLATION.md)
- [Open SWE Customization Guide](https://github.com/langchain-ai/open-swe/blob/main/CUSTOMIZATION.md)
- [Open SWE Blog Post](https://blog.langchain.com/open-swe-an-open-source-framework-for-internal-coding-agents/)
- [Deep Agents Repository](https://github.com/langchain-ai/deepagents)
- [Deep Agents Docs](https://docs.langchain.com/oss/python/deepagents/overview)
- [LangGraph Platform Docs](https://docs.langchain.com/oss/python/langgraph/overview)
