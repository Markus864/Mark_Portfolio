# ADR-0003: Idempotent Execution and Broker-as-Source-of-Truth Reconciliation

- **Status:** Accepted
- **Date:** Reference architecture
- **Deciders:** Mark Splawn
- **Related:** ADR-0001 (data provider), ADR-0002 (risk model), `architecture.md` §6

---

## Context

In automated trading, the most expensive bugs aren't bad signals — they're
**execution and state bugs**, because they put *real, unintended exposure* on the
book:

- **The duplicate-order problem.** "Place order" times out. The caller can't tell
  whether the order was accepted. A naive retry submits a second order — now the
  system holds **2× the intended position** and 2× the risk.
- **The lost-position problem.** The engine keeps the open position only in
  memory. A crash/restart loses it: the bot either never exits (no stop
  management) or re-enters on the next scan (double exposure).
- **The phantom-position problem.** Local state says "in position," but the
  position was closed out of band (manual close, liquidation, transfer). The
  engine monitors something that no longer exists, forever.
- **The settlement-lag problem.** A fill succeeds, but the venue's
  balance/position read briefly lags. Reading too early returns zero and the
  engine wrongly concludes the trade didn't happen.

Local memory simply **cannot be trusted** as the record of what the account
holds. The venue is reality; the engine's job is to stay consistent with it.

## Decision

Make execution **idempotent** and treat the **broker as the source of truth**,
reconciling against it continuously. The goal is **at-most-once exposure**
per logical intent.

1. **Confirm-then-commit.** Working state flips to "in position" **only after the
   venue confirms the fill** (and the resulting position is readable). If submit
   times out or confirmation fails, state is **not** mutated — the next
   reconciliation pass discovers the truth. This is what makes a retry safe: a
   retry that finds the position already open simply adopts it instead of
   doubling it.

2. **Pre-trade safety checks.** Before sending, validate execution conditions
   (e.g. price-impact / spread within limits). Orders that would fill at a bad
   price are aborted *before* hitting the venue.

3. **Attach protection at fill.** Where the venue supports it, stop-loss and
   take-profit are attached **on fill** (server-side), so protection exists even
   if the engine dies immediately after entry.

4. **Settlement-lag tolerance.** After a fill, position/balance reads **retry
   with backoff**. State commits only once the position is observable; a brief
   lag never causes a false "no fill."

5. **Reconciliation every cycle and on startup.** The engine compares its state
   against the broker's reported positions and converges:
   - **Adopt** a position that exists on the venue but isn't tracked locally
     (e.g. opened manually), wrapping it in managed state with computed
     stop/target so the engine starts protecting it.
   - **Clear** a locally tracked position whose venue balance has effectively
     vanished, using a **dust threshold** so fractional leftovers don't trap the
     engine into trying to close an unsellable scrap.

6. **Fail-safe exits.** If a close order fails, working state is **preserved**
   (not optimistically wiped) so the exit is retried on the next pass. The engine
   would rather re-attempt an exit than "forget" an open position.

## Consequences

### Positive

- **Toward at-most-once exposure.** The design targets the duplicate-position
  class of bugs: a retried intent that finds the position already open adopts it
  rather than opening a second one.
- **Crash/restart resilience.** Because state is a checkpoint reconciled against
  the venue, the engine resumes mid-trade correctly after any restart, redeploy,
  or host move — open positions are picked back up, not lost or duplicated.
- **Robust to out-of-band actions.** Manual trades and external closes/
  liquidations are handled gracefully (adopt / clear) instead of corrupting the
  engine's view.
- **No false "no fill."** Settlement-lag retries make post-fill state commits
  reliable.
- **Protection survives engine death.** Server-side stop/target on fill means a
  position is defended even if the process crashes the instant after entry.

### Negative / costs

- **Extra round-trips on the happy path.** Confirm + read-back + per-cycle
  reconciliation cost additional venue calls and a little latency versus
  fire-and-forget. For real money this is an easy trade.
- **More involved logic.** Adoption, phantom-clearing, dust thresholds, and
  backoff are real complexity that must be carefully tested — including
  partial-fill and reject paths.
- **Threshold tuning.** The dust threshold is a judgment call: too low and the
  engine clings to unsellable scraps; too high and it could discard a small but
  real position. Chosen conservatively and documented.

### Neutral

- "Broker is source of truth" means local state is explicitly a *cache*. That
  framing is the whole point — it forces every code path to ask the venue rather
  than trust memory, which is the correct discipline for handling money.

## Alternatives considered

- **Fire-and-forget with retries.** Rejected outright: directly causes duplicate
  positions on the exact failure (timeout) where retries are most tempting.
- **Client-generated idempotency keys only.** A good complementary technique
  (and worth adding where a venue supports it), but **not sufficient on its
  own**: it doesn't recover lost in-memory state after a crash, and it doesn't
  reconcile out-of-band changes. Reconciliation against the venue covers the
  cases keys can't.
- **Trust local state, reconcile manually.** Rejected: a human is not in the loop
  at 3 a.m.; consistency with the venue must be automatic and continuous.
- **Full event-sourcing of execution now.** Attractive and noted as future work
  (derive all state from an immutable event log for complete replayability), but
  heavier than required to drive toward at-most-once exposure today; the
  confirm-then-commit + reconcile design covers the core cases with far
  less machinery.
