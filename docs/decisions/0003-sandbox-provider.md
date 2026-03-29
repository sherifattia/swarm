---
status: "accepted"
date: 2026-03-28
decision-makers: ["sherifattia"]
---

# Use LangSmith Sandboxes as the sandbox provider for POC

## Context and Problem Statement

Open SWE requires an isolated sandbox per task — a remote Linux environment with full shell access where the agent clones a repo, runs tests, and produces commits. The sandbox provider is pluggable via `SANDBOX_TYPE` and the `SandboxBackendProtocol` interface (see [Open SWE reference](../open-swe-reference.md#2-sandbox-pluggable)). We need to choose a provider for the POC phase. Production will likely require a GCP-native solution, but the POC should take the path of least resistance.

## Decision Drivers

* POC priority: fastest path to a working demo
* Must support full Linux environment (shell, git, test runners)
* Must allow parallel task execution (one sandbox per task)
* Must integrate with the LangGraph Platform we are already deploying on ([ADR-0004](0004-poc-hosting.md))
* Should minimize additional credentials and infrastructure setup
* Production will be GCP-native — POC provider need not be the final choice

## Considered Options

* [LangSmith Sandboxes (default)](#langsmith-sandboxes-default)
* [Daytona](#daytona)
* [Runloop](#runloop)
* [Modal](#modal)
* [Local (dev only)](#local-dev-only)

## Decision Outcome

Chosen option: "LangSmith Sandboxes", because it is the default provider in Open SWE and requires zero additional setup beyond the LangSmith API key we already need for tracing and hosting ([ADR-0004](0004-poc-hosting.md)). For a POC, eliminating infrastructure decisions is more valuable than optimizing for performance or cost.

We will revisit this decision for production, where a GCP-native solution (custom provider on GKE or Cloud Run) would align with our org's infrastructure standards.

### Consequences

* Good, because zero additional infrastructure — LangSmith sandboxes come with the LangSmith ecosystem we are already using
* Good, because it is the default and best-tested provider in Open SWE
* Good, because integrated tracing — sandbox execution traces flow into the same LangSmith project as agent traces
* Good, because sandbox lifecycle is managed (creation, persistence per thread, auto-recreation on failure)
* Bad, because it adds another dependency on LangChain's platform (vendor concentration risk)
* Bad, because it is not GCP-native — production will need a different provider, meaning a migration
* Bad, because sandbox resource limits and pricing are controlled by LangChain, not by us
* Neutral, because the `SandboxBackendProtocol` interface means switching providers later is a code change, not an architecture change

## Pros and Cons of the Options

### LangSmith Sandboxes (default)

Default provider in Open SWE. Configured via `LANGSMITH_API_KEY_PROD` environment variable.

LangSmith cloud sandboxes are managed by LangChain. Each sandbox provides a full Linux environment with shell access, repo cloning, and tool execution. Sandboxes are persistent per conversation thread and auto-recreate if they become unreachable.

* Good, because zero additional setup — uses the same API key as LangSmith tracing
* Good, because it is the default and most exercised code path in Open SWE
* Good, because sandbox lifecycle management (create, persist, recreate) is handled automatically
* Good, because tracing integration provides visibility into sandbox operations
* Bad, because LangSmith dependency deepens vendor lock-in
* Bad, because we have no control over sandbox hardware, region, or resource limits
* Bad, because not GCP-native — cannot run in our org's infrastructure

### Daytona

Third-party sandbox provider. Configured via `DAYTONA_API_KEY`.

Daytona provides cloud development environments with API access. It supports custom workspace configurations and is designed for reproducible development environments.

* Good, because it is purpose-built for cloud development environments
* Good, because it supports custom workspace templates
* Bad, because it requires a separate Daytona account and API key
* Bad, because it adds a third-party dependency outside the LangSmith ecosystem
* Bad, because it is less exercised in Open SWE than the default LangSmith path
* Bad, because it is not GCP-native

### Runloop

Third-party sandbox provider. Configured via `RUNLOOP_API_KEY`.

Runloop provides isolated execution environments designed for AI agent workloads with API access.

* Good, because it is designed specifically for AI agent execution
* Good, because it provides isolation guarantees
* Bad, because it requires a separate Runloop account and API key
* Bad, because it adds a third-party dependency outside the LangSmith ecosystem
* Bad, because it is less exercised in Open SWE than the default LangSmith path
* Bad, because it is not GCP-native

### Modal

Third-party sandbox provider. Configured via Modal credentials.

Modal provides serverless container execution with a Python-first API. It is popular in the ML/AI community for running compute workloads.

* Good, because Modal's serverless containers are fast to spin up
* Good, because Python-first API aligns with Open SWE's Python codebase
* Good, because Modal is well-established in the AI/ML ecosystem
* Bad, because it requires Modal credentials and account setup
* Bad, because Modal's pricing model (per-second compute) may be unpredictable for agent workloads
* Bad, because it is not GCP-native
* Bad, because it is less exercised in Open SWE than the default LangSmith path

### Local (dev only)

No isolation. No external credentials needed.

Runs agent commands directly on the host machine. Intended only for local development and testing — there is no sandbox isolation.

* Good, because zero setup — no accounts, keys, or infrastructure
* Good, because fastest iteration for local development
* Bad, because no isolation — agent has full access to the host filesystem and network
* Bad, because not suitable for any shared or production environment
* Bad, because cannot run parallel tasks safely (filesystem conflicts)
* Bad, because not representative of production behavior

## More Information

### Switching Providers

The sandbox provider is configured via the `SANDBOX_TYPE` environment variable and the `SandboxBackendProtocol` interface. Switching providers requires:

1. Set `SANDBOX_TYPE` to the new provider name
2. Configure the provider's credentials (API key, etc.)
3. Optionally customize sandbox templates via `DEFAULT_SANDBOX_TEMPLATE_NAME` and `DEFAULT_SANDBOX_TEMPLATE_IMAGE`

For a custom GCP-native provider (production), implement a factory function in `agent/integrations/` returning an object that implements `SandboxBackendProtocol`, then register it in `agent/utils/sandbox.py`.

See the [Open SWE reference](../open-swe-reference.md#2-sandbox-pluggable) for full details.
