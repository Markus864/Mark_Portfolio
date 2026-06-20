# Architecture Decision Records (ADRs)

An ADR captures a single significant architectural decision: the **context** that forced a
choice, the **decision** taken, and the **consequences** (good and bad) I accepted. They are
deliberately short and dated. The point is that a future reader — or a future me — can
reconstruct *why* the system is the way it is, not just *what* it does.

These ADRs are **cross-cutting**: they apply across the whole portfolio rather than to one
project. Project-specific decisions live with their own case studies.

## Format

Each record follows the same template:

- **Status** — Accepted / Superseded / Deprecated
- **Context** — the forces and constraints in play
- **Decision** — what I chose to do
- **Consequences** — what this makes easy, and what it costs

## Log

| ADR | Title | Status |
|---|---|---|
| [0001](0001-multi-cloud-split.md) | Multi-cloud workload split (edge / GCP / AWS) | Accepted |
| [0002](0002-managed-secrets-store.md) | Managed secrets store, names-only in code | Accepted |
| [0003](0003-self-hosted-runner-deploys.md) | Self-hosted runner for deploys | Accepted |
| [0004](0004-terraform-iac.md) | Terraform as the Infrastructure-as-Code standard | Accepted |
| [0005](0005-monitoring-alerting.md) | Monitoring, alerting & cost guardrails | Accepted |

See the [architecture overview](../architecture/README.md) for the narrative that ties these
decisions together.
