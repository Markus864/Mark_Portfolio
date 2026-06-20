# ADR-0004 — Terraform as the Infrastructure-as-Code standard

- **Status:** Accepted
- **Date:** 2026-06-18
- **Deciders:** Mark Splawn
- **Related:** [Multi-cloud strategy](../architecture/01-multi-cloud-strategy.md),
  [CI/CD strategy](../architecture/03-cicd-strategy.md)

## Context

Infrastructure spans more than one cloud (a data/AI core on GCP, a few primitives on AWS, plus
edge configuration). Click-ops across multiple consoles is unreproducible, undocumented, and
drifts silently — the exact opposite of what a multi-cloud estate needs. I needed
infrastructure to be **versioned, reviewable, reproducible, and cloud-agnostic in tooling** so
the same workflow applies everywhere.

The forces: **reproducibility**, **a single tool across clouds**, **reviewability in PRs**, and
**state/drift management**, weighed against per-cloud native IaC (which would mean a different
tool per cloud) and the learning/state-management overhead any IaC tool carries.

## Decision

Standardize on **Terraform** as the Infrastructure-as-Code tool across all clouds:

- Infrastructure is declared in version-controlled Terraform and changed through pull
  requests, so every infra change is reviewed exactly like application code.
- The same `plan` / `apply` workflow applies to every provider, giving one mental model across
  GCP and AWS.
- Terraform **state is stored in managed remote backend storage** (not on a laptop), with
  state locking to prevent concurrent corruption.
- **No secret values live in Terraform variables committed to the repo.** Sensitive inputs are
  supplied from the managed secret store or via `*.tfvars.example` files that contain
  **placeholders only** (see [ADR-0002](0002-managed-secrets-store.md)).

## Consequences

**Positive**
- Infrastructure is reproducible and self-documenting — the repo *is* the source of truth.
- One tool and one workflow across multiple clouds; no per-cloud IaC dialect to context-switch.
- `plan` output makes the blast radius of an infra change visible *before* it is applied, and
  PR review catches mistakes early.
- AWS primitives in the estate are applied through the same pipeline as everything else.

**Negative / costs accepted**
- Terraform state is itself sensitive and must be secured (remote backend + locking + access
  control); a `.tfvars` with a real value would be a leak, so this is enforced by the secret
  scan.
- Provider abstractions are not perfectly uniform — some cloud-specific resources still require
  cloud-specific knowledge.
- A learning/maintenance cost for state and module structure (accepted; it is the price of
  reproducibility).
