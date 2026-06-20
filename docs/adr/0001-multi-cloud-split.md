# ADR-0001 — Multi-cloud workload split (edge / GCP / AWS)

- **Status:** Accepted
- **Date:** 2026-06-18
- **Deciders:** Mark Splawn
- **Related:** [Multi-cloud strategy](../architecture/01-multi-cloud-strategy.md)

## Context

The portfolio spans three genuinely different workload shapes: a globally distributed,
latency-sensitive request surface; stateful, data-heavy AI/analytics processing; and a small
set of operational primitives where one provider is clearly best. No single cloud is the best
home for all three at once:

- An all-edge stack lacks a first-class data warehouse and heavy long-running compute.
- An all-GCP stack pays idle/cold-start penalties for a global front door and gives up a few
  best-of-breed primitives elsewhere.
- An all-AWS stack would work but offers no compelling reason to abandon the edge platform's
  latency/cost profile or GCP's data tooling.

The competing forces are **latency**, **data gravity**, **cost curve at low/spiky volume**,
**operational simplicity**, and **avoiding lock-in** — and they pull toward different
providers.

## Decision

Adopt a **deliberate three-lane split**, routing each workload to a cloud by its shape:

1. **Serverless edge cloud** owns the public request surface (APIs, webhooks, agent
   endpoints, light orchestration) — chosen for global low latency and pay-per-request cost.
2. **GCP** owns the data and AI core (containerized services, data warehouse, bulk object
   storage, inference) — chosen for data gravity and integrated, Terraform-friendly data
   tooling.
3. **AWS** is used **thinly** for a few best-of-breed managed primitives (notifications,
   alarms), consumed over their APIs.

Lanes are coupled only through portable contracts: HTTPS, S3/GCS-compatible object storage,
and a message queue. A new cloud service is adopted only if it is *materially* better for a
workload than what an existing lane already provides.

## Consequences

**Positive**
- Each workload runs where it is fastest and cheapest; no single dimension is compromised.
- Lock-in is bounded — the front door and AWS lane are thin and portable; only the
  data core is intentionally sticky (justified by data gravity).
- Placement is teachable: "what shape is this work?" deterministically selects a lane.

**Negative / costs accepted**
- More than one cloud's IAM, console, and billing model to operate.
- Cross-cloud calls add a network hop and a second failure domain (mitigated by retries,
  timeouts, and graceful degradation — see [ADR-0005](0005-monitoring-alerting.md)).
- Potential egress cost when a non-data lane reads large data (mitigated by keeping compute
  next to the data).

**Guardrails to prevent sprawl**
- One lane owns each workload class; no duplicate stacks "just in case."
- Contracts stay boring and portable; no lane reaches into another's database.
- Identity and cost are managed per-cloud but reviewed centrally.
