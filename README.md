# Mark Splawn — OT/IT Solutions Architect

> Cloud & AI platforms, built on a hands-on industrial automation (OT/IT) foundation — designed to survive contact with production.

I'm an **OT/IT Solutions Architect**: I pair hands-on industrial automation and controls
(PLC/SCADA, robotics, OT security) with self-built **cloud and AI platforms** — multi-cloud
across a serverless edge, GCP, and AWS, event-driven serverless, and tool-using AI agents.
I design for the four things that actually matter once a system is live: **scalability,
reliability, security, and cost.**

This repository is a curated set of architecture case studies — systems I designed and
built. Each one is framed the
way I think about real engagements — **Problem -> Architecture -> Key Decisions and
Trade-offs -> Technology -> Outcomes** — with architecture diagrams and Architecture
Decision Records (ADRs) capturing the *why*, not just the *what*.

---

## Professional Summary

I take ambiguous business problems and turn them into systems that are buildable,
operable, and affordable. My sweet spot is the intersection of three worlds that rarely
share a language:

- **Applied AI / agentic systems** — model-backed services, tool-using agents, the
  [Model Context Protocol (MCP)](https://modelcontextprotocol.io) as an integration
  fabric, retrieval-grounded responses, and the guardrails (grounding, approval gates,
  fallbacks) that make autonomy safe to ship.
- **Cloud-native and serverless** — event-driven architectures on managed primitives
  (queues, object storage, edge compute, serverless databases, workflow engines) so the
  bill tracks usage and the operational surface stays small.
- **IT/OT convergence** — bridging deterministic, safety-critical industrial equipment
  (PLCs, SCADA, sensor telemetry) to elastic cloud analytics, without compromising the
  isolation the plant floor requires.

I default to **infrastructure as code**, **CI/CD with security scanning in the
pipeline**, and **observability designed in from day one** — because an architecture you
cannot reproduce, secure, or see is a liability, not an asset.

---

## Core Competencies

| Domain | What I bring |
|---|---|
| **AI / Agent Platforms** | Tool-using agents, MCP servers and adapters, multi-provider model routing with fallback, retrieval-grounded generation, approval-gated autonomy, prompt and context engineering |
| **Cloud — GCP** | BigQuery, Cloud Run, Cloud Functions, Pub/Sub, GCS, IAM, VPC design, billing and quota guardrails |
| **Cloud — AWS** | Lambda, S3, SNS/SQS, CloudWatch, IAM least-privilege, cross-account patterns |
| **Edge & Serverless** | Edge functions, serverless SQL, object storage, durable queues, workflow orchestration, DAG-based pipelines |
| **Event-Driven Architecture** | Queue-decoupled services, idempotent consumers, retry/back-off and dead-letter strategies, async fan-out |
| **IT/OT & Industrial** | PLC/SCADA integration, OPC UA and MQTT telemetry, edge-to-cloud gateways, network segmentation across the IT/OT boundary |
| **Infrastructure as Code** | Terraform modules, environment promotion, reproducible provisioning, drift control |
| **CI/CD** | Pipeline-as-code, automated lint/test/build, secret scanning (gitleaks) and SAST gates, artifact promotion |
| **Security** | Least-privilege IAM, secret externalization (env vars / managed secret stores), defense-in-depth, threat modeling |
| **Observability** | Structured logging, metrics, distributed tracing, health checks, actionable alerting and SLOs |
| **Data & Analytics** | Streaming and batch pipelines, warehouse modeling, time-series and telemetry analytics |

---

## Project Index

Each project is a self-contained case study under [`projects/`](projects/). Start with
the flagships if you only have a few minutes.

| Project | One-line value | Status |
|---|---|---|
| [AI Agent Platform](projects/ai-agent-platform/) | A reusable framework for tool-using agents — a planner over a typed tool registry with **approval-gated autonomy**, a provider-agnostic model mesh, and a step-by-step setup guide to deploy your own. | Flagship |
| [Industrial IoT / IT–OT Convergence Platform](projects/industrial-iot-platform/) | **IT/OT convergence** — plant-floor telemetry bridged one-way to cloud analytics with retrieval-grounded troubleshooting, fully provisioned with **Terraform**. | Flagship |
| [Serverless AI Media-Generation Platform](projects/serverless-ai-media-platform/) | An **event-driven** serverless pipeline — request → moderation → queue → durable workflow → multi-provider AI → object storage — pay-per-use by design. | Case study |
| [Algorithmic Trading Platform](projects/algorithmic-trading-platform/) | A multi-strategy trading engine with **pluggable market-data/broker adapters**, layered risk management and a kill switch, and idempotent execution. | Case study |
| [MCP Integration Pattern](projects/mcp-integration-pattern/) | A clean, reusable pattern for exposing an existing system to AI agents as a typed **MCP server**. | Pattern |

> A consolidated narrative, diagrams, and the cross-cutting Architecture Decision Records
> live in [`docs/`](docs/); each project also carries its own project-specific ADRs in its
> folder — see the [documentation index](docs/README.md).

---

## How to Read This Portfolio

Every case study follows the same structure so you can compare design reasoning across
domains:

1. **Problem** — the business and technical context, and the constraints that shaped it.
2. **Architecture** — a diagram (Mermaid) plus the component walkthrough.
3. **Key Decisions and Trade-offs** — the forks in the road and why I chose each path.
4. **Technology** — the concrete stack, and why each piece earned its place.
5. **Outcomes** — what the design is built to achieve against scalability, reliability,
   security, and cost.

ADRs ([`docs/adr/`](docs/adr/)) record individual decisions in *context / decision /
consequences* form, so the reasoning outlives any single diagram.

---

## Engineering Principles

These show up in every diagram in this repo:

- **Decouple with events.** Queues and async stages turn spikes into backlog instead of
  outages, and let components fail and recover independently.
- **Make state cheap and boring.** Managed, serverless data stores over self-run
  infrastructure unless a hard requirement says otherwise.
- **Least privilege, secrets externalized.** No credential ever lives in source. Access
  is scoped to the smallest role that works. Configuration is by **environment variable
  name**, never a hardcoded value.
- **Design for the bill.** Pay-per-use primitives and right-sized compute so cost scales
  with value delivered, not with idle capacity.
- **Observable or it didn't happen.** Structured logs, metrics, traces, and health
  checks are part of the design, not an afterthought.
- **Reproducible by default.** If it isn't in Terraform or a pipeline, it doesn't exist.

---

## Contact

I'm open to Solutions Architect and Cloud/AI Architect conversations.

- **Name:** Mark Splawn
- **Email:** splawnmark@gmail.com
- **LinkedIn:** https://www.linkedin.com/in/marksplawn/
- **Résumé:** [PDF](resume/Mark_Splawn_Resume.pdf) · [DOCX](resume/Mark_Splawn_Resume.docx) · [Markdown](resume/Mark_Splawn_Resume.md)

---

## License

Released under the [MIT License](LICENSE). The architectures, diagrams, and decision
records here are free to reference and adapt.
