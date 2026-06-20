# ADR-0002: Queue + durable workflow + DLQ for the job pipeline

- **Status:** Accepted
- **Date:** 2026
- **Deciders:** Mark Splawn (architecture owner)
- **Tags:** reliability, durability, billing-correctness, async

## Context

A generation job is **multi-minute, multi-step, failure-prone, and billed per job**:

- It calls one or more third-party AI providers, each of which can take 30s–several minutes
  and can fail with timeouts, rate limits (429), partial responses, or transient 5xx.
- It runs under a serverless runtime that can terminate any invocation (see ADR-0001), so a
  worker can die *mid-job*.
- The customer's credits are reserved at submission, so the system aims for
  **effectively-once billing**: settle on success, refund on failure, avoid double-charging,
  avoid doing free work.

A naive "fire a background request from the API handler and hope it finishes" approach fails
on every axis: it dies with the request, has no retry, can leave a job stuck in `processing`
forever, and can re-run expensive (already-paid) steps if anything reattempts it.

## Decision

Move all slow work onto an **asynchronous, durable pipeline** built from three coordinated
primitives plus a reconciliation backstop:

1. **A queue** decouples submission from execution. The API edge reserves credits, writes the
   job row, enqueues `{ jobId }`, and returns immediately. A queue **consumer** drives
   execution independently of any client connection.

2. **A durable workflow** runs the job as a sequence of **checkpointed steps**
   (`shape prompt → generate artifact → persist artifact → caption → settle`). Each step is
   recorded on completion, so a worker killed mid-job **resumes from the last completed step**
   instead of restarting — which is what prevents re-running and re-billing expensive provider
   calls.

3. **A dead-letter queue (DLQ)** with bounded retries. Transient failures redeliver the
   message (and the workflow resumes from its checkpoint). Messages that exhaust their retry
   budget land in the DLQ for inspection rather than vanishing or looping forever.

4. **A cron "reaper"** as an independent safety net: it sweeps for jobs stuck in a
   non-terminal state past a timeout, marks them `failed` (write-once), and **auto-refunds the
   reserved credits**. This catches the residual case where a worker died so abruptly it never
   recorded a failure.

The billing model that rides on top is **reserve-then-settle** with **write-once terminal
states** (`completed` / `failed`), which is designed to keep the correctness goals holding even
under concurrent retries and reaper runs.

## Consequences

**Positive**

- **Resumability.** A killed worker resumes from its last checkpoint; expensive provider calls
  already completed are not repeated.
- **Effectively-once billing by design.** Reserve-then-settle + write-once terminal states
  are intended to charge a job for outcomes, not attempts; double-settle and double-refund are
  made very unlikely because the first terminal write wins.
- **No lost jobs, by design.** Every enqueued job is intended to reach a terminal state via
  success, retry, DLQ triage, or the reaper.
- **Graceful degradation under provider flakiness.** 429s/timeouts are absorbed by bounded
  retries instead of surfacing as user-facing failures.
- **Observability.** Fine-grained `processing:*` states + a per-job `providerTrace` make
  progress and failures visible; the DLQ is a concrete queue of things to look at.

**Negative / accepted trade-offs**

- **More moving parts.** Producer, consumer, workflow, DLQ, and reaper are more components than
  a single background call. Accepted: each one closes a specific, real failure mode.
- **Eventual-consistency UX.** The client gets a `jobId` and polls; the result is not inline.
  This is inherent to multi-minute work and is surfaced honestly via job states.
- **Idempotency is mandatory, not optional.** Because messages can redeliver, every step and
  every terminal transition must be safe to run more than once. Enforced via write-once
  terminal states and idempotent step bodies.
- **Reaper tuning.** The stale-timeout must sit comfortably above the slowest legitimate job
  (longest video generation + retries) to avoid failing in-flight work. Treated as an
  operational parameter.

## Alternatives considered

- **Background request from the handler:** rejected — dies with the request, no retry, no
  resumption, leaks `processing` rows, can double-bill on any reattempt.
- **Queue with retries but no durable workflow:** rejected — retries would re-run the *entire*
  job (including already-completed, already-paid provider calls) instead of resuming from a
  checkpoint.
- **Workflow without a DLQ:** rejected — poison jobs would retry forever or be lost; the DLQ
  gives a bounded, inspectable failure sink.
- **No reaper, rely only on retries/DLQ:** rejected — neither covers a worker that dies
  without recording a failure, which would strand a job non-terminal and its credits
  reserved indefinitely.
