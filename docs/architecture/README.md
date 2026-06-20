# Cross-Cutting Architecture

This section documents the **architectural principles that span every project** in this
portfolio rather than any single application. Individual project case studies cover their
own problem statements and designs; the documents here capture the *reusable principles
and practices* I design for and apply — how I split workloads across clouds, how I handle
secrets and access, how I ship code safely, and how I keep systems observable, reliable,
and within budget. Not every practice is live in every project; they are the stance I bring
and apply where each system warrants it.

I designed these as a coherent platform stance. They are written to be adopted wholesale
by an individual operator or a small-to-midsize engineering org: nothing here assumes a
large platform team, yet every choice scales up cleanly when one arrives.

## Documents

| # | Topic | Problem it solves |
|---|-------|-------------------|
| [01 — Multi-Cloud Strategy](01-multi-cloud-strategy.md) | When and why to use GCP vs. a serverless edge cloud vs. AWS | Avoids both single-vendor lock-in and undisciplined "multi-cloud sprawl" |
| [02 — Secrets Management & Least Privilege](02-secrets-and-least-privilege.md) | A single model for secrets storage, rotation, and scoped identity | Closes off the "key in the repo" failure class and over-broad credentials |
| [03 — CI/CD Strategy](03-cicd-strategy.md) | Test + security-scan + deploy pipeline with a hard secrets gate | Makes "it builds, it's scanned, it ships" the default path, not a manual one |
| [04 — Observability, Reliability & Cost Guardrails](04-observability-reliability-cost.md) | One model for logs/metrics/alerts, SLOs, and spend control | Catches outages and runaway spend before users (or the bill) do |

## Architecture Decision Records

The reasoning behind the load-bearing choices lives in the [ADR log](../adr/README.md).
Each ADR is a short, dated record of **context → decision → consequences** so a reader can
reconstruct *why* a decision was made, not just *what* was decided.

## Design principles (the through-line)

Every document below is an application of the same five principles:

1. **Right tool for the workload, not one tool for everything.** Workload shape — latency,
   data gravity, statefulness, cost curve — drives the platform choice. Convenience does not.
2. **Secrets live in a managed store; code carries only the *name* of a secret.** No
   credential is ever committed, printed, or baked into an image.
3. **The safe path is the default path.** If a pipeline can ship code, it can also run the
   tests and the secret scan first — automatically, with no opt-out.
4. **If it isn't observable, it isn't in production.** Every service emits structured logs,
   exposes health, and has at least one alert that pages a human before a user notices.
5. **Cost is a first-class non-functional requirement.** Budgets and quotas are
   architecture — designed into code and configuration so spend isn't a monthly surprise.
