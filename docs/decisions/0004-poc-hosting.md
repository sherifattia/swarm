---
status: "accepted"
date: 2026-03-28
decision-makers: ["sherifattia"]
---

# POC hosting on LangSmith Cloud

## Context and Problem Statement

Open SWE runs as a LangGraph server that receives webhooks (Slack, GitHub, Linear), executes agent runs with durable state, and manages sandbox lifecycles. We need to host this server somewhere for the POC. Our organization is a GCP shop, but the POC should take the path of least resistance to a working demo. Production hosting will be revisited separately.

## Decision Drivers

* POC priority: fastest path to a working demo with zero infrastructure management
* Must support webhook ingestion (public HTTPS endpoint for Slack, GitHub, Linear)
* Must support LangGraph runtime (durable execution, streaming, checkpointing)
* Must support concurrent agent runs
* Free or near-free is acceptable for POC
* Production will be GCP — POC hosting need not be the final choice

## Considered Options

* [LangSmith Cloud (Developer tier, free)](#langsmith-cloud-developer-tier-free)
* [Self-hosted on GKE](#self-hosted-on-gke)
* [Self-hosted on Cloud Run](#self-hosted-on-cloud-run)
* [LangSmith Enterprise (self-hosted)](#langsmith-enterprise-self-hosted)

## Decision Outcome

Chosen option: "LangSmith Cloud Developer tier", because it is free, fully managed, and the fastest path to a working demo. It provides the LangGraph runtime, webhook endpoints, tracing, and a web dashboard out of the box. The Developer tier limits (1 seat, 5K traces/month, 50 Fleet runs/month) are acceptable for a POC.

We accept that the agent runs on LangChain's infrastructure, not our GCP. This is a deliberate POC tradeoff — production hosting will move to GCP (likely GKE or Cloud Run with self-hosted LangGraph).

### Consequences

* Good, because zero infrastructure to manage — no clusters, load balancers, or TLS certificates
* Good, because the Developer tier is free (1 seat, 5K traces, 50 Fleet runs)
* Good, because the LangGraph runtime, tracing, and web dashboard are included
* Good, because public HTTPS endpoints are provided for webhook ingestion
* Good, because fastest path to demo — deploy and start receiving webhooks immediately
* Bad, because agent runs outside our GCP environment — not representative of production topology
* Bad, because 50 Fleet runs/month limits the number of feature requests we can process
* Bad, because it creates a dependency on LangChain's platform availability and performance
* Bad, because no control over region, scaling, or resource allocation
* Neutral, because migration to self-hosted LangGraph on GCP is a documented path (LangSmith Enterprise or open-source LangGraph server)

## Pros and Cons of the Options

### LangSmith Cloud (Developer tier, free)

[LangSmith](https://smith.langchain.com) · Managed by LangChain · Free tier

LangSmith Cloud is LangChain's managed platform for deploying LangGraph applications. The Developer tier provides 1 seat, 5K traces/month, and 50 Fleet runs/month at no cost. It includes the LangGraph runtime with durable execution, streaming, and checkpointing, plus a web dashboard for task management and plan review.

Deployment is via `langgraph deploy` or GitHub integration. Webhook endpoints are provided automatically with public HTTPS URLs.

* Good, because free — no cost for POC
* Good, because fully managed — zero infrastructure overhead
* Good, because includes tracing, dashboard, and runtime in one platform
* Good, because deployment is a single command
* Bad, because 50 runs/month is a hard limit on POC throughput
* Bad, because single seat limits collaboration
* Bad, because runs on LangChain's infrastructure, not our GCP
* Bad, because vendor lock-in deepens with hosting + tracing + sandboxes all on LangSmith

### Self-hosted on GKE

Google Kubernetes Engine · GCP · Pay-per-use

Run the LangGraph server as a Kubernetes deployment on GKE. This aligns with our org's GCP infrastructure standards and provides full control over scaling, region, and resource allocation. Requires setting up the cluster, configuring ingress with TLS for webhooks, and managing the LangGraph server deployment.

* Good, because it runs in our GCP environment — representative of production
* Good, because full control over scaling, region, and resources
* Good, because GKE is well-understood by the team
* Good, because no external platform dependency for hosting
* Bad, because significant setup overhead (cluster, ingress, TLS, DNS, monitoring)
* Bad, because ongoing infrastructure management cost (time and money)
* Bad, because GKE minimum cost is non-trivial even for a POC
* Bad, because over-engineered for a demo — we are optimizing for speed, not production readiness

### Self-hosted on Cloud Run

Google Cloud Run · GCP · Pay-per-request

Run the LangGraph server as a Cloud Run service. Serverless, scales to zero, and simpler than GKE. Requires configuring a custom domain or using the Cloud Run URL for webhooks.

* Good, because serverless — scales to zero when not in use
* Good, because simpler than GKE (no cluster management)
* Good, because pay-per-request pricing is cost-effective for low-volume POC
* Good, because runs in our GCP environment
* Bad, because Cloud Run's request timeout (up to 60 minutes) may be tight for long agent runs
* Bad, because LangGraph's durable execution model may not map cleanly to Cloud Run's stateless container model
* Bad, because still requires setup (container registry, service deployment, domain/TLS for webhooks)
* Bad, because cold starts could delay webhook processing

### LangSmith Enterprise (self-hosted)

LangSmith Enterprise · Self-hosted · Custom pricing

LangSmith Enterprise allows running the LangSmith platform on your own infrastructure (including GCP). It provides the same capabilities as LangSmith Cloud but with full control over data residency, scaling, and security. Custom pricing based on usage.

* Good, because it runs on our infrastructure with full control
* Good, because same capabilities as LangSmith Cloud (runtime, tracing, dashboard)
* Good, because data stays in our GCP environment
* Bad, because custom pricing — likely expensive for a POC
* Bad, because significant setup overhead (deploy the LangSmith platform itself)
* Bad, because requires sales engagement and onboarding with LangChain
* Bad, because over-engineered for a POC — this is a production solution

## More Information

### Migration Path to Production

The expected production path is:

1. **POC** (now): LangSmith Cloud Developer tier — validate the agent pipeline works end-to-end
2. **Pilot**: Upgrade to LangSmith Cloud paid tier or self-host on Cloud Run — increase run limits, add team seats
3. **Production**: Self-hosted on GKE or LangSmith Enterprise — full GCP integration, org security requirements, SLA guarantees

The LangGraph server is an open-source Python application. Moving from LangSmith Cloud to self-hosted requires deploying the server ourselves and updating webhook URLs. The agent code does not change.
