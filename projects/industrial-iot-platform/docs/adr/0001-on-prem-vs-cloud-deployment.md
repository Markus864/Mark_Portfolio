# ADR-001 — Develop in the Cloud, Run Production On-Premises

- **Status:** Accepted
- **Context area:** Deployment topology, cost, data residency

## Context

The platform serves a single manufacturing site. The data it touches — drawings, repair
history, fault events, anything derived from the control system — is proprietary and, by plant
policy, should not leave the building in steady-state operation.

At the same time, building the system demanded fast, disposable, reproducible infrastructure:
spin an environment up, exercise it against simulated PLC and document data, tear it down, and
do it again without hand-configuring servers. Waiting on physical hardware procurement for
every iteration would have throttled delivery.

We also had a clear long-run cost signal: a small, single-site workload running 24/7 in the
cloud is a permanent monthly bill, whereas the business already owns capable on-prem hardware
that is otherwise idle.

## Decision

Run **two environments with one container stack**:

- **Development / test → cloud.** A fully Terraform-defined cloud environment (private network,
  hardened VM, managed Postgres, object storage, secret store, WAF) used for all build-out and
  integration testing against simulated OT data.
- **Production → on-premises.** The *same* container stack, intended to be deployed on an on-prem
  server inside the plant and connected to live OT systems through the edge gateway.

Production cutover is deliberately boring: pull the images, supply the production configuration,
run the database migrations. Because the stack is identical, "works in dev" carries real weight.

After go-live, the cloud environment is downsized to a minimal instance — kept for ongoing
development and staging, not for running production load.

## Consequences

**Positive**
- In the production topology, proprietary plant data is designed to stay on-site; the only
  outbound calls are to the allow-listed CMMS and LLM endpoints.
- Development is fast and reproducible — environments are cattle, defined in code, not pets.
- Steady-state cost is intended to be dominated by hardware the business already owns, with the
  recurring cloud bill shrinking to a minimal instance after go-live.
- Identical stacks across environments collapse a whole class of "it only breaks in prod"
  problems.

**Negative / accepted trade-offs**
- Two environments to maintain, and an explicit responsibility to keep them in lockstep.
- On-prem operational burden falls on whoever runs the box: backups, OS patching, certificate
  renewal, and physical/network security have to be run on-site, not by a managed service.
- The on-prem footprint must be capacity-planned up front, since it can't autoscale like cloud.

**Mitigations**
- Single source of truth for the application stack (one container set, one config schema)
  minimises drift.
- The cloud environment doubles as a staging mirror for validating changes before they reach
  the on-prem production server.
- On-prem operational steps (backup/restore, cert renewal, cutover) are written up as runbooks
  rather than living in someone's head.
