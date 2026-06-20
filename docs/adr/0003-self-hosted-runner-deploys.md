# ADR-0003 — Self-hosted runner for deploys

- **Status:** Accepted
- **Date:** 2026-06-18
- **Deciders:** Mark Splawn
- **Related:** [CI/CD strategy](../architecture/03-cicd-strategy.md),
  [Secrets & least privilege](../architecture/02-secrets-and-least-privilege.md)

## Context

CI (lint, test, secret-scan, build) is cleanly handled by hosted runners — it needs no
privileged access and benefits from disposable, isolated environments. CD is different. The
deploy step must:

- Reach deploy targets, some of which are private or internal and not exposed to the public
  internet.
- Hold **scoped deploy credentials** (registry push, deploy tokens) that I do not want sitting
  in a shared hosted pool's environment.
- Produce a single, auditable origin for every production change.

The forces: **keep deploy credentials tightly held**, **reach private targets**, **stay
auditable**, against the **operational cost of running my own runner** and the convenience of
fully hosted CI.

## Decision

Run **CI on hosted runners** but **CD on a self-hosted runner**:

- The self-hosted runner is the single audited place from which production deploys originate.
- It **resolves its deploy credentials from the managed secret store at job time** (see
  [ADR-0002](0002-managed-secrets-store.md)) rather than storing them statically.
- Production deploys additionally require an **explicit manual approval** before the runner
  acts.
- The deploy *policy* (test → scan → approve → deploy → smoke-check → rollback) is uniform;
  only the final per-target command differs (edge CLI / Cloud Run / Terraform for AWS
  primitives).

## Consequences

**Positive**
- Deploy credentials never live on shared infrastructure; they are fetched, used, and dropped.
- Private/internal targets are reachable without exposing them publicly.
- Every production change has one origin and one log — clean auditability.
- Combined with smoke-check + rollback, a bad deploy self-recovers.

**Negative / costs accepted**
- I operate and patch the runner (kept minimal; the control it buys is worth it).
- The runner is a component that must stay available for deploys to proceed (mitigated by
  keeping CI — the high-frequency path — on hosted runners, so only deploys depend on it).
- A self-hosted runner must be kept hardened, since it holds deploy capability; access to it
  is restricted accordingly.
