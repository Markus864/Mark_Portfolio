# ADR-0005 — Monitoring, alerting & cost guardrails

- **Status:** Accepted
- **Date:** 2026-06-18
- **Deciders:** Mark Splawn
- **Related:** [Observability, reliability & cost](../architecture/04-observability-reliability-cost.md),
  [Multi-cloud strategy](../architecture/01-multi-cloud-strategy.md)

## Context

A multi-cloud estate of small services has three quiet failure modes: blind spots (a service
degrades unnoticed), fragile recovery (incidents handled by memory under pressure), and
runaway cost (pay-per-use and managed-data spend creeping up silently). The third is not
hypothetical — a **billing lapse can silently freeze a downstream data pipeline**, which is
why *billing health is best treated as an operational signal, not just a finance concern*.

The forces: I need **enough visibility to catch problems before users do** and **enough
spend control to avoid surprises**, without creating **alert fatigue** (which destroys the
value of a pager) or a heavyweight observability stack that a small team can't sustain.

## Decision

Adopt one monitoring/reliability/cost model applied to every service on every cloud:

1. **Observability by default.** Every service emits **structured JSON logs with a
   correlation id**, exposes **RED metrics** (rate/errors/duration), and serves **health/ready
   endpoints**. Logs and metrics ship to **central aggregation** so a cross-cloud request can
   be traced in one place.
2. **Symptom-based alerting against SLOs.** Each user-facing service has a simple
   availability/latency SLO with an error budget. Alerts fire on **what the user feels** (error
   rate, latency burn, unreachable health probe), routed by severity: **critical → page a
   human; warning → notify a channel**. Every alert is actionable and links to a runbook.
3. **Reliability by design.** Top failure modes have written runbooks; cross-cloud calls use
   timeouts, bounded retries with backoff, and graceful degradation; failed deploys roll back
   on a bad smoke check.
4. **Cost guardrails as configuration.** Per-project **budget alerts** (tripped well before
   month-end), **quotas/rate limits** to bound abuse and bugs, **scale-to-zero** compute, and
   **billing/spend anomalies routed into the same alert channels** as outages.

## Consequences

**Positive**
- Problems surface before users report them; a cross-cloud request is debuggable from one
  aggregation point.
- Incidents become procedures (runbooks + auto-rollback), not panics.
- Spend tracks usage and can't silently run away; a billing problem pages someone instead of
  quietly breaking a pipeline.
- The pager stays trustworthy because it only fires for human-actionable, user-affecting
  events.

**Negative / costs accepted**
- Central aggregation adds a small ingest cost and one more component to maintain.
- Symptom-based alerting means some slow-burn issues surface on dashboards rather than as an
  instant page (an accepted trade to avoid fatigue).
- Budgets/quotas occasionally need a deliberate, reviewed bump for a legitimate spike.
- Runbooks cover the top failure modes, not every conceivable one; rare novel failures still
  need live debugging.
