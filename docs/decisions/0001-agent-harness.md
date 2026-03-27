---
status: "accepted"
date: 2026-03-27
decision-makers: ["sherifattia"]
---

# Use Open SWE as the agent harness for autonomous coding

## Context and Problem Statement

Swarm is a GitHub App that takes feature specs and produces reviewed pull requests using autonomous coding agents. We need to choose an agent harness — the framework that receives a task, spins up an isolated environment, executes the coding work, and delivers a PR. This is the foundational architectural decision for the entire project.

Several companies have independently built production systems like this: [Stripe (Minions)](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents), [Ramp (Inspect)](https://blog.langchain.com/open-swe-an-open-source-framework-for-internal-coding-agents/), and [Coinbase (Cloudbot)](https://blog.langchain.com/open-swe-an-open-source-framework-for-internal-coding-agents/). Their architectures converged on the same patterns: isolated sandboxes, curated tools, Slack/GitHub-first invocation, rich context at startup, and subagent orchestration.

## Decision Drivers

* Must support autonomous, headless execution triggered by GitHub webhooks
* Must produce pull requests end-to-end (clone repo, implement feature, run tests, open PR)
* Must run in isolated sandboxes (one task per environment, no cross-contamination)
* Should be open source with a permissive license
* Should support multiple LLM providers (avoid vendor lock-in)
* Should have an existing GitHub integration or a clear path to building one
* Should be actively maintained with a healthy community
* Should be designed for the "internal coding agent" use case, not just interactive pair programming

## Considered Options

* [Open SWE](https://github.com/langchain-ai/open-swe) (by LangChain)
* [OpenCode](https://github.com/anomalyco/opencode) (by Anomaly)
* [Goose](https://github.com/block/goose) (by Block/Square)
* [Claude Code CLI / Agent SDK](https://github.com/anthropics/claude-agent-sdk-python) (by Anthropic)
* [Pi](https://github.com/badlogic/pi-mono) (by Mario Zechner)

## Decision Outcome

Chosen option: "Open SWE", because it is purpose-built for exactly the use case we are building — an autonomous coding agent system modeled after what Stripe, Ramp, and Coinbase independently converged on. It provides the full pipeline out of the box: GitHub webhook triggers, sandboxed execution, a planner/programmer agent architecture with human-in-the-loop plan approval, and automatic PR creation. It is open source (MIT), Python-based, and designed to be a customizable foundation rather than a finished product.

If Open SWE proves insufficient, **OpenCode** is our fallback — it is the foundation Ramp used for their production system (Inspect), has the deepest GitHub integration, and its client/server architecture is well-suited for building orchestration on top of. The tradeoff is that we would need to build the full spec-to-PR pipeline ourselves.

### Consequences

* Good, because it captures the architectural patterns that three major companies independently validated in production
* Good, because it includes GitHub integration (issue label triggers, PR @-mention triggers, webhooks) out of the box
* Good, because the three-graph architecture (Manager → Planner → Programmer) with human-in-the-loop plan approval aligns with our spec-driven workflow
* Good, because it runs on LangGraph Platform which provides durable execution, streaming, checkpointing, and session management
* Good, because Python aligns with the broader AI/ML ecosystem and LangChain tooling
* Bad, because it is very new (released March 2026) and may have rough edges
* Bad, because it currently only supports Daytona as a sandbox provider, creating a vendor dependency
* Bad, because the LangGraph Platform dependency adds infrastructure complexity
* Bad, because the LangChain ecosystem moves fast and APIs may change

## Pros and Cons of the Options

### Open SWE

[GitHub](https://github.com/langchain-ai/open-swe) · [Blog Post](https://blog.langchain.com/open-swe-an-open-source-framework-for-internal-coding-agents/) · Python · MIT · ~6.2K stars

Open SWE is LangChain's open-source framework explicitly modeled after the internal coding agent systems at Stripe, Ramp, and Coinbase. It decomposes work into three LangGraph graphs: a **Manager** (orchestration/routing), a **Planner** (reads codebase, proposes a task plan, waits for approval), and a **Programmer** (executes the plan, runs tests, creates a PR). Each task runs in an isolated Daytona sandbox.

Invocation is via GitHub issue labels (`open-swe-auto`), PR @-mentions, a web dashboard (Next.js), a CLI, or the LangGraph SDK programmatically. Tools include a text editor, shell, Git operations (branch/commit/push/PR), and arbitrary MCP servers.

* Good, because it is purpose-built for the "spec in, PR out" use case
* Good, because the Planner/Programmer split maps directly to our spec-driven workflow
* Good, because human-in-the-loop plan approval provides a safety gate before code changes
* Good, because it supports multiple LLM providers (Anthropic default, plus OpenAI and Google)
* Good, because MCP server extensibility allows adding tools without modifying core code
* Good, because the LangGraph Platform provides durable execution, streaming, and checkpointing
* Good, because the web dashboard provides task management and plan review UI
* Bad, because it is very new (March 17, 2026) with limited production track record
* Bad, because Daytona is the only supported sandbox provider (no Docker/Modal/E2B alternative)
* Bad, because it depends on the LangGraph Platform infrastructure
* Bad, because the monorepo couples the web app, agent, and CLI deployments

### OpenCode

[GitHub](https://github.com/anomalyco/opencode) · TypeScript · MIT · ~131K stars

OpenCode is a provider-agnostic AI coding agent with a client/server architecture. It was used by Ramp as the foundation for their internal coding agent system (Inspect). It runs as a headless HTTP server (`opencode serve`) with a full REST API, and has official TypeScript and Python SDKs. Its GitHub Action supports PR review, issue triage, and comment-triggered automation (`/oc` commands).

The agent system supports multi-agent configurations (primary agents and subagents with different models and tool access), a plugin system with 20+ lifecycle hooks, and MCP server integration. OpenCode supports 75+ LLM providers.

* Good, because Ramp built their production system (Inspect) on top of it
* Good, because it has the deepest GitHub integration of any option (PR review, issue triage, comment triggers)
* Good, because the client/server architecture is ideal for building orchestration on top
* Good, because it supports 75+ LLM providers — maximum flexibility
* Good, because 131K stars indicates massive community adoption
* Good, because the plugin system allows deep customization (custom tools, permission gates, lifecycle hooks)
* Bad, because it is a lower-level building block — you compose on top of it rather than getting the full pipeline
* Bad, because it is young (~11 months) with 6,340 open issues suggesting rapid growth outpacing maintenance
* Bad, because there is no built-in sandbox — you must provide your own container isolation
* Bad, because the full "spec → plan → implement → PR" pipeline would need to be built by us

### Goose

[GitHub](https://github.com/block/goose) · Rust · Apache 2.0 · ~33.7K stars

Goose is Block's open-source AI agent — the project that Stripe forked and customized into Minions, which now ships 1,300+ PRs per week. It features a YAML-based recipe system for packaging reproducible workflows, headless execution (`goose run`), sub-recipe orchestration (sequential and parallel, up to 10 concurrent workers), and MCP-native extensibility (extensions are MCP servers).

Goose supports 20+ LLM providers, has Docker deployment support, and documented GitHub Actions CI/CD patterns. Its recipe system supports parameterization, retry logic, and structured JSON output schemas.

* Good, because Stripe's Minions (1,300+ merged PRs/week) is built on a fork of Goose — proven at massive scale
* Good, because the recipe system provides composable, version-controlled workflow definitions
* Good, because sub-recipe orchestration supports parallel execution (up to 10 workers)
* Good, because MCP-native architecture means any MCP server becomes an extension
* Good, because it supports 20+ LLM providers with per-recipe model overrides
* Good, because Apache 2.0 license and active Block backing
* Bad, because Stripe's orchestration layer (Blueprints, Toolshed, devbox management) is proprietary — you get the agent but not the production infrastructure
* Bad, because there is no built-in GitHub webhook/PR pipeline — you build that yourself
* Bad, because headless execution is CLI-based (`goose run`), not a persistent HTTP service
* Bad, because the gap between "Goose the agent" and "Minions the system" is significant engineering work

### Claude Code CLI / Agent SDK

[Agent SDK (Python)](https://github.com/anthropics/claude-agent-sdk-python) · [Agent SDK (TypeScript)](https://github.com/anthropics/claude-agent-sdk-typescript) · [GitHub Action](https://github.com/anthropics/claude-code-action) · Proprietary CLI, MIT SDKs

The Claude Agent SDK wraps Claude Code into a programmable library. It provides the same built-in tools (Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch), context management, and sandboxing that power Claude Code. The `query()` function runs headlessly, and the official GitHub Action (`anthropics/claude-code-action@v1`) enables PR/issue comment triggers.

The SDK supports subagent orchestration via the Task tool, custom MCP tools, hooks (PreToolUse/PostToolUse), permission modes, session persistence/forking, structured output, and file checkpointing. Anthropic provides extensive documentation on secure containerized deployment with defense-in-depth patterns.

* Good, because the inner-loop agent (read code, make changes, run tests, create PR) is battle-tested and production-grade
* Good, because built-in tools, context management, and sandboxing are best-in-class
* Good, because the official GitHub Action already handles PR/issue comment triggers
* Good, because extensive security and sandboxing documentation (Docker hardening, gVisor, proxy architecture)
* Good, because subagent orchestration, hooks, and custom tools provide deep customization
* Bad, because it is locked to Claude models only — no provider flexibility
* Bad, because the SDK spawns a CLI subprocess (~1GB RAM per instance), not a lightweight API call
* Bad, because you must build the entire outer loop yourself (webhook receiver, job queue, worker pool, status reporting)
* Bad, because the CLI binary must be installed in every runtime environment
* Bad, because the proprietary CLI means you depend on Anthropic's release cadence for the core runtime

### Pi

[GitHub](https://github.com/badlogic/pi-mono) · [Website](https://pi.dev) · TypeScript · MIT · ~26K stars

Pi is a minimalist, open-source coding agent by Mario Zechner. Its design philosophy is "minimal core, extend everything" — only 4 built-in tools (read, write, edit, bash) with an ~800 token system prompt. It is structured as a three-layer monorepo: `pi-ai` (unified LLM API across 20+ providers), `pi-agent-core` (stateful agent management), and `pi-coding-agent` (full coding agent SDK).

Pi can be embedded via TypeScript SDK (`createAgentSession()`) or RPC mode (JSONL over stdin/stdout for language-agnostic integration). Its extension system supports lifecycle hooks, permission gates, custom tools, slash commands, and system prompt injection.

* Good, because it is the most embeddable option — TypeScript SDK or language-agnostic RPC mode
* Good, because the modular monorepo means you can adopt just the LLM layer (`pi-ai`) or the full agent
* Good, because 20+ LLM providers with unified API, including local models (Ollama)
* Good, because the extension system is first-class (lifecycle hooks, permission gates, custom tools)
* Good, because minimal system prompt (~800 tokens) leaves maximum context for actual work
* Bad, because there is no native GitHub integration — no webhook handling, no PR creation workflow
* Bad, because there is no built-in sandbox — you must provide your own isolation
* Bad, because "dictatorial" governance means limited ability to influence the roadmap
* Bad, because the full "spec → plan → implement → PR" pipeline would need to be built entirely from scratch

## More Information

### Background: The Stripe Minions Architecture

The architecture we are building toward was pioneered by [Stripe's Minions system](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents). Key patterns from Stripe that informed this decision:

* **Blueprints**: Workflows alternating deterministic steps (git operations, linting, CI) with agentic steps (code writing, test fixing)
* **Devboxes**: Isolated execution environments pre-warmed and ready in ~10 seconds
* **Toolshed**: Centralized MCP server with ~500 curated tools per agent
* **Two-round CI limit**: Bounded iteration prevents runaway costs
* **Mandatory human review**: AI-authored code gets 1.75x more logic errors — human review is a critical safety net

### LangChain's Comparative Analysis

The [Open SWE blog post](https://blog.langchain.com/open-swe-an-open-source-framework-for-internal-coding-agents/) compares three companies that independently built this pattern:

| | Stripe (Minions) | Ramp (Inspect) | Coinbase (Cloudbot) |
|---|---|---|---|
| **Built on** | Forked Goose | Composed on OpenCode | Built from scratch |
| **Sandbox** | AWS EC2 devboxes | Modal containers | In-house |
| **Tools** | ~500, curated per-agent | OpenCode SDK + extensions | MCPs + custom Skills |
| **Invocation** | Slack + embedded buttons | Slack + web + Chrome extension | Slack-native |
| **Validation** | 3-layer (local + CI + 1 retry) | Visual DOM verification | Agent councils + auto-merge |

All three converged on: isolated sandboxes, curated toolsets, Slack-first invocation, rich context at startup, and subagent orchestration.

### Option Comparison Matrix

| | Open SWE | OpenCode | Goose | Claude Code | Pi |
|---|---|---|---|---|---|
| **Language** | Python | TypeScript | Rust | TS/Python SDKs | TypeScript |
| **License** | MIT | MIT | Apache 2.0 | Proprietary CLI | MIT |
| **LLM providers** | 3+ | 75+ | 20+ | Claude only | 20+ |
| **Sandbox** | Daytona | BYO | BYO + container-use | Built-in + Docker | BYO |
| **GitHub triggers** | Labels, @-mentions, webhooks | Action, `/oc`, issue triage | CI/CD pattern | Action (`@claude`) | None |
| **Spec → PR pipeline** | Built-in | Build yourself | Build yourself | Build yourself | Build yourself |
| **Subagents** | Planner/Programmer | Multi-agent config | Sub-recipes (parallel) | Task tool | Via extensions |
| **Production precedent** | Codifies Stripe/Ramp/Coinbase | Ramp (Inspect) | Stripe (Minions) | GitHub Action ecosystem | OpenClaw |
