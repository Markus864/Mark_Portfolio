# Serverless AI Media-Generation Platform

> A case study in designing, building, and deploying an event-driven, edge-native SaaS
> backend for long-running, failure-prone AI workloads — with durable execution, a
> multi-provider abstraction layer, billing, auth, rate limiting, moderation, and a DAG
> orchestrator for multi-step jobs.

**Author:** Mark Splawn — Solutions Architect / Platform Engineer
**Type:** Architecture case study (design + working reference implementation)
**Status:** Designed, built, and deployed; the end-to-end pipeline has run live in testing.
This is not a finished paying-public product — one AI component was not completed for a full
public launch — so the write-up focuses on the architecture and the properties it is
designed to provide.
**Domain framing:** Generic B2B/B2C SaaS — applies to any product that turns a user request
into one or more expensive, asynchronous AI calls and returns a stored artifact.

---

## 1. Problem

I needed to build the backend for a SaaS product where a single user action triggers
**slow, expensive, and unreliable** work: a request goes out to one or more third-party AI
generation providers, each call can take anywhere from **30 seconds to several minutes**,
any call can fail or rate-limit me, and the user is paying real money (per-job credits) for
the result.

That combination breaks the naive request/response model immediately:

- **You cannot hold an HTTP connection open for 3 minutes.** Edge and serverless runtimes
  cap request duration; users close tabs; load balancers time out.
- **Provider calls fail in ugly ways** — timeouts, partial responses, 429s, transient 5xx.
  A job that dies halfway through must not silently bill the customer or leave an orphaned
  record stuck in `processing` forever.
- **You are billing per job**, so correctness is a first-class goal: a failed job should
  refund automatically, and a retried job should not double-charge.
- **It is user-generated content**, so every request must pass a moderation gate before any
  paid provider call is made.
- **Provider risk is existential** — pricing, quality, and availability of AI vendors shift
  monthly. Hard-coupling the platform to one vendor's SDK is a business risk, not just a
  code smell.
- **Idle cost matters.** A side-project-scale or early-stage SaaS cannot pay for an
  always-on cluster waiting for traffic that arrives in bursts.

So the real problem was never "call an AI API." It was: **design a system that turns an
unreliable, multi-minute, billed, multi-vendor workload into a reliable, observable,
resumable pipeline that costs nothing when idle.**

---

## 2. Architecture

I designed an **event-driven, edge-native pipeline** with a clean separation between the
fast synchronous path (accept the request, charge intent, return a job ID) and the slow
asynchronous path (do the expensive work durably, in the background).

```
Request  ->  Moderation  ->  Queue  ->  Durable Workflow  ->  Provider Adapters  ->  Object Storage  ->  Result
   (sync, <100ms)                          (async, retryable, checkpointed)
```

### The two paths

**Synchronous path (the API edge):**
1. Authenticate the caller (session cookie for the web app, hashed API key for
   programmatic/agent access).
2. Enforce rate limits and plan/credit entitlements.
3. Run a fast moderation gate on the prompt.
4. **Reserve** credits (not charge — reserve), write a `job` row in state `queued`, enqueue
   a message, and return a `jobId` immediately.

**Asynchronous path (the durable worker):**
5. A queue consumer picks up the message and starts a **durable workflow instance** keyed by
   the job ID. Each meaningful stage is a checkpointed step.
6. The workflow calls the **provider adapter layer** — a vendor-agnostic interface for prompt
   shaping, image generation, motion/video generation, and captioning. Adapters poll
   long-running provider jobs and normalize wildly different response envelopes.
7. Each produced artifact is **copied out of the provider's ephemeral URL into our own object
   storage** so we own the asset and can serve it with range requests, caching, and access
   control.
8. The job row is advanced through fine-grained states
   (`processing:prompt -> processing:image -> processing:caption -> completed`),
   credits are settled, and the user is notified. On any failure the job is failed via a
   write-once transition, credits are refunded, and the message is retried or dead-lettered.

### Reliability backbone

- **Queue + Dead-Letter Queue + bounded retries.** Transient failures retry automatically;
  exhausted messages land in a DLQ for inspection instead of vanishing.
- **Durable workflow with per-step checkpoints.** A worker that is killed mid-job resumes
  from the last completed step rather than re-running expensive provider calls or
  double-billing.
- **Scheduled reconciliation (the "reaper").** A cron job sweeps for jobs stuck in a
  non-terminal state past a timeout, fails them, and **auto-refunds credits** — the
  safety net intended to ensure a customer is not left charged for a job that never finished.
- **Idempotent state transitions.** Terminal states (`completed`, `failed`) are designed to be
  write-once, so retries and the reaper should not corrupt a finished job.

### Multi-step / DAG orchestration

Single-artifact jobs are a linear chain. The same workflow primitive scales to **multi-step
DAG jobs** — "produce three variations, pick the best, then run it through a post-processing
template" — by composing checkpointed steps with fan-out/fan-in. Because each node is an
independently retryable step keyed under one workflow instance, the design avoids re-charging
or re-running steps 1–3 when step 4 of a 6-step DAG fails. The orchestrator is described in
[`docs/architecture.md`](docs/architecture.md).

### Surfaces

The same backend is exposed two ways:
- A **web application** (authenticated UI, billing portal, asset gallery).
- A **programmatic / agent surface** implemented as an **MCP server** so AI agents and
  external automations can submit jobs, poll status, and read account state through the
  exact same job pipeline — no shadow code path. (See ADR-0003.)

A full system diagram, the async-pipeline sequence, and the provider-adapter layer are in
**[`docs/architecture.md`](docs/architecture.md)**.

---

## 3. Key Decisions & Trade-offs

These are the decisions I would defend in a design review. The three with the most
architectural weight are written up as full ADRs in [`docs/adr/`](docs/adr/).

| Decision | Why | Trade-off I accepted |
|---|---|---|
| **Edge/serverless runtime over an always-on cluster** ([ADR-0001](docs/adr/0001-edge-serverless-runtime.md)) | Near-zero idle cost, global low-latency edge, integrated data plane (object store + queue + workflow + SQL) behind one deploy | Runtime time/CPU limits force a strict "no long-lived process" discipline — which I turned into an asset, not a constraint |
| **Queue + durable workflow + DLQ for the job pipeline** ([ADR-0002](docs/adr/0002-queue-workflow-dlq-durability.md)) | The way I chose to make a multi-minute, billed, failure-prone job *resumable* and *effectively-once-billed* under a runtime that can kill you mid-request | More moving parts (producer, consumer, workflow, DLQ, reaper) than a single background call |
| **Vendor-agnostic provider adapter layer** ([ADR-0003](docs/adr/0003-provider-abstraction-layer.md)) | AI vendor pricing/quality/availability shifts monthly; swapping or A/B-testing a provider must be a config change, not a rewrite | A normalization layer to maintain, and a "lowest common denominator" capability contract |
| **Reserve-then-settle credit model** | Bill *intent* up front, settle on success, auto-refund on failure — prevents both double-charging and free work | A credit ledger and reconciliation logic instead of a simple "charge on click" |
| **Moderation gate before any paid call** | UGC safety is table stakes; also avoids spending money generating content that will be rejected | Added latency on the sync path and false-positive handling |
| **Copy provider output into our own object storage** | We own the artifact, can serve it with range requests / immutable caching / access control, and survive the provider expiring its temp URL | Extra egress + storage cost and a copy step per artifact |
| **Hashed API keys + same pipeline for agents** | One job path for web and programmatic callers; keys are never stored in plaintext | Key rotation/management UX to build |

---

## 4. Tech

Chosen for the *shape* of the problem, named generically so the design transfers to any
comparable stack (the same pattern maps cleanly onto AWS Lambda + SQS + Step Functions + S3,
or GCP Cloud Run + Pub/Sub + Workflows + GCS).

- **Compute:** Edge/serverless functions (request handlers + queue consumers + a cron-
  triggered worker).
- **Durable execution:** A managed **workflow** engine with per-step checkpointing.
- **Messaging:** A managed **queue** with a **dead-letter queue** and configurable retry/
  batch policy.
- **State:** A serverless **SQL database** accessed through a thin **repository interface**
  (so the persistence engine is swappable and the rest of the code never sees raw SQL).
- **Object storage:** A serverless blob store fronted by an asset route that supports HTTP
  **range requests** and immutable caching for media playback/download.
- **Auth:** Session-based auth for the web app; **SHA-256-hashed API keys** for programmatic
  access.
- **Billing:** A third-party payments provider for subscriptions, plus an internal
  **credit ledger** with idempotent, signature-verified webhook handling.
- **Agent interface:** An **MCP (Model Context Protocol) server** exposing the job pipeline
  as tools.
- **AI providers:** Pluggable behind the adapter interface (prompt shaping, image, video,
  captioning) — selected via environment configuration, never imported directly by business
  logic.

> All credentials are injected as environment variables (e.g. `PROVIDER_API_KEY`,
> `PAYMENTS_SECRET_KEY`, `PAYMENTS_WEBHOOK_SECRET`, `OBJECT_STORE_BINDING`). No secrets,
> account IDs, or endpoints appear in source. See [`.env.example`](.env.example).

---

## 5. Outcomes

What the architecture is designed to provide, framed as capabilities (this is a design case
study, so the "results" are the reliability and cost properties the design targets):

- **A customer should not be left charged for a job that didn't finish.** Reserve-then-settle
  plus the reaper-with-auto-refund is a reconciliation flow that minimizes wrongful billing
  rather than leaving it to chance.
- **A killed worker is designed to lose no work and avoid double-billing.** Checkpointed
  workflow steps are intended to make the multi-minute pipeline resumable and effectively-once
  in its billing.
- **Provider swaps are a config change.** Adding or replacing an AI vendor touches only an
  adapter; business logic, billing, and the queue/workflow are untouched.
- **Near-zero idle cost.** The platform scales to zero between bursts and scales out per
  message — no idle cluster to pay for.
- **One pipeline, two surfaces.** Web users and AI agents submit jobs through identical
  code, so there is no second, less-tested path to secure or maintain.
- **Operationally observable.** Every job carries a provider trace and moves through
  fine-grained states; failures dead-letter instead of disappearing.

---

## 6. Repository Map

```
serverless-ai-media-platform/
├── README.md                 # This case study (Problem -> Architecture -> Decisions -> Tech -> Outcomes)
├── .env.example              # Required environment variables (names only, no values)
└── docs/
    ├── architecture.md       # System diagram, async-pipeline sequence, provider-adapter layer (Mermaid)
    └── adr/
        ├── 0001-edge-serverless-runtime.md
        ├── 0002-queue-workflow-dlq-durability.md
        └── 0003-provider-abstraction-layer.md
```

---

## 7. What I'd Build Next

The roadmap is itself part of the design story — knowing where a system is *not yet* fully
production-hardened (it is production-shaped, not production-at-scale) is a senior skill:

1. **Per-tenant rate-limit isolation** — move from global token buckets to per-API-key /
   per-plan quotas enforced at the edge before the moderation gate.
2. **DAG cost pre-flight** — estimate provider spend for a multi-step job and require explicit
   confirmation above a threshold.
3. **Provider health-aware routing** — circuit-break a degraded provider and shift traffic to
   a fallback adapter automatically.
4. **Sensitive-asset lifecycle** — EXIF stripping, scoped/expiring asset URLs, and
   user-initiated export/delete to satisfy privacy requirements for user-uploaded reference
   media.
5. **End-to-end idempotency keys** — accept a client idempotency key on job submission so a
   retried *submission* (not just a retried execution) is deduplicated.
