# ADR-0001: Edge/serverless runtime over an always-on cluster

- **Status:** Accepted
- **Date:** 2026
- **Deciders:** Mark Splawn (architecture owner)
- **Tags:** compute, cost, scalability, deployment

## Context

The platform turns user requests into expensive, bursty, asynchronous AI workloads. Traffic
is spiky: long idle periods punctuated by bursts when users batch-generate content. The
product is early-stage, so **idle infrastructure cost is a direct threat to viability**, and
the team operating it is small, so **operational surface area must stay minimal**.

The workload also needs a tightly integrated data plane: object storage for media, a queue
for async dispatch, a durable workflow engine for resumable jobs, and a SQL store for
state/billing. Stitching those together from separate, self-managed services would multiply
the ops burden.

The candidate runtimes were:

1. **An always-on container cluster** (e.g. Kubernetes / a managed container service with a
   minimum running count) behind a load balancer.
2. **A traditional autoscaling VM/container group** that scales on CPU.
3. **An edge/serverless platform** with first-class managed primitives for functions, a
   queue, a durable workflow engine, object storage, and serverless SQL — all deployable
   behind a single configuration and CLI.

## Decision

**Adopt an edge/serverless runtime as the primary deployment target**, with the request
handlers, queue consumers, and the cron worker all running as serverless functions, and the
queue / workflow / object store / SQL all provisioned as managed bindings of that same
platform.

A critical secondary decision falls out of this: **no component may assume a long-lived
process.** Request handlers must return quickly; all multi-minute work must live behind the
queue in the durable workflow (see ADR-0002). I treat the runtime's per-invocation time and
CPU limits not as a limitation to fight but as a **forcing function** that pushes the
architecture toward the stateless, resumable shape it needs anyway.

To avoid hard lock-in, the design is kept **portable by intent**: data access sits
behind a repository interface, provider calls behind an adapter interface, and the
queue/workflow concepts (producer, consumer, checkpointed steps, DLQ) map directly onto the
equivalent managed primitives of other major clouds.

## Consequences

**Positive**

- **Near-zero idle cost.** The platform scales to zero between bursts; we pay per request and
  per message, not for an idle cluster.
- **Elastic burst handling.** Each queued message can be processed concurrently up to
  configured limits without pre-provisioning capacity.
- **Minimal ops surface.** Functions + managed bindings deploy behind one config; there is no
  cluster, node pool, or load balancer to patch and babysit.
- **Global low latency** for the synchronous edge (auth, validation, enqueue, status polling).
- **Architectural discipline for free.** The "no long-lived process" rule it imposes is
  exactly what pushes the job pipeline toward being resumable and the billing effectively-once.

**Negative / accepted trade-offs**

- **Hard runtime limits.** Per-invocation time/CPU/memory caps mean any genuinely long task
  *must* be decomposed into queued, checkpointed steps. (Mitigated — that decomposition is
  ADR-0002, and it's a feature for reliability.)
- **Cold starts** on the synchronous path. Acceptable because the sync path only does cheap
  work (auth, moderation, a DB write, an enqueue) and returns a job ID.
- **Some platform-specific bindings.** Mitigated by the repository + adapter seams and by
  choosing only primitives that have direct equivalents on other clouds, keeping a port a
  bounded effort rather than a rewrite.
- **Local development needs emulation.** Mitigated by the deterministic fallback provider and
  local bindings so the full pipeline runs offline.

## Alternatives considered

- **Always-on cluster:** rejected primarily on idle cost and ops burden for an early-stage,
  bursty workload; its main advantage (no execution-time limit) is neutralized once work is
  queued and checkpointed anyway.
- **Autoscaling VMs on CPU:** rejected because AI jobs are I/O-bound (waiting on slow provider
  polls), so CPU-based autoscaling tracks load poorly and still pays for warm idle capacity.
