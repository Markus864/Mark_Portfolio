# ADR-0002: Layered Risk Model with a Hard Kill Switch

- **Status:** Accepted
- **Date:** Reference architecture
- **Deciders:** Mark Splawn
- **Related:** ADR-0003 (idempotent execution), `architecture.md` §5

---

## Context

The platform runs multiple strategies, automatically, against live markets. The
existential risk is not "a strategy under-performs" — it's "a strategy bug, a
data glitch, or a runaway loop drains the account before a human intervenes."

Early versions encoded risk as a single hard-coded stop-loss percentage inside
each strategy. That was inadequate:

- **No account-level backstop.** Per-trade stops don't bound *cumulative*
  losses. A losing streak (or a logic bug opening trade after trade) could march
  the balance to zero with every individual trade "behaving."
- **Risk logic duplicated and bypassable.** Each strategy reimplemented sizing
  and stops, so a single strategy could size itself recklessly, and fixes had to
  be made in N places.
- **Sizing ignored stop distance.** Fixed-notional sizing means a wide stop
  risks far more dollars than a tight one — risk per trade was effectively
  random.
- **No "don't trade" state.** Nothing could put the system into a safe,
  non-trading mode on its own.

I needed risk to be **shared infrastructure that every strategy inherits**, with
**multiple independent gates** and a **hard floor that no strategy logic can
override**.

## Decision

Implement risk as a **layered set of independent gates** owned by a central Risk
Manager that sits between every signal and the broker. Each layer can veto;
nothing reaches execution without passing all of them.

1. **Kill switch (hard backstop).** A capital **floor**. When equity breaches it,
   the engine stops opening positions and sets a persisted flag. It re-arms only
   when equity recovers above a higher **reset** threshold. The deliberate gap
   between floor and reset is **hysteresis**, preventing the switch from
   oscillating at the boundary. The switch is checked at the top of every loop —
   the engine won't even scan while tripped.

2. **Expected-value gate.** Entries are allowed only when modelled EV clears a
   minimum threshold, so the system simply waits when its own model doesn't favour
   a trade.

3. **Risk-based position sizing.** Dollar risk per trade is a **fixed fraction of
   current equity**; the number of units is derived from the **stop distance**:

   ```python
   def position_size(equity, risk_pct, entry, stop) -> int:
       risk_distance = abs(entry - stop)
       if risk_distance == 0:
           return 0
       return max(0, int((equity * risk_pct) / risk_distance))
   ```

   Risk per trade is therefore constant regardless of how tight or wide the stop
   is. Above a capital threshold, an optional **fractional-Kelly** tier (half-
   Kelly, clamped to a sane band) scales size for compounding while staying
   conservative.

4. **Pluggable stop & target models.** Strategies declare *which* model they want;
   the risk layer computes the prices: fixed-percentage, **R-multiple** (target =
   N × risk distance, enabling clean reward:risk control and multi-target
   sweeps), and **volatility-/structure-based** (ATR multiple, or recent swing
   high/low).

5. **Soft confirmation filters (fail-open).** Optional external confirmations
   (e.g. order-flow or sentiment) can discourage a trade but **fail open** — a
   flaky third-party signal must never block an otherwise-valid trade — and only
   veto when clearly negative.

6. **Minimum-size guard.** Reject orders too small to be economically sensible.

Peak equity is tracked continuously to feed drawdown logic and reporting.

## Consequences

### Positive

- **Bounded worst case (under assumptions).** The kill switch is designed to put
  a floor under losses that **no strategy logic can override** — the property I
  care about most for running automated strategies with real capital. The bound
  holds **assuming the stop and kill switch fire and the broker honors them**;
  gaps, slippage, or a venue failure can still breach it, so this is strong
  defense in depth rather than an absolute guarantee.
- **Constant, intentional risk per trade.** Stop-distance-based sizing makes
  every trade risk the same fraction of equity, so position size is a deliberate
  decision, not an accident of the stop width.
- **Single source of truth for risk.** All sizing/gating lives in one tested
  place; strategies can't reimplement or bypass it, and improvements apply
  everywhere at once.
- **Composable safety.** Independent layers mean a gap in one (say, a soft
  filter) is still backstopped by the others and ultimately by the kill switch.
- **Model-says-no → no-trade.** The EV gate keeps the system idle when its own
  model doesn't favour a trade, which is itself a form of capital protection.

### Negative / costs

- **More configuration and state.** Floor, reset, risk fraction, Kelly threshold,
  stop models, EV minimum, min size — more knobs to set correctly, and more
  states (armed / tripped / recovering) to reason about and test.
- **Hysteresis tuning.** If the floor/reset gap is too narrow it can still feel
  twitchy; too wide and recovery is slow. Requires thought per deployment.
- **Conservative by design.** Fractional-Kelly and the EV gate intentionally
  leave some upside on the table in exchange for survivability — the right trade
  for capital preservation, but a trade nonetheless.

### Neutral

- "Fail open" for soft filters vs. "fail safe" for capital-touching gates is an
  intentional asymmetry: don't let a flaky signal block trading, but never let
  missing risk inputs *enable* it.

## Alternatives considered

- **Single per-strategy stop-loss only.** Rejected: no account-level backstop, no
  shared enforcement, sizing blind to stop distance.
- **Full-Kelly sizing.** Rejected: mathematically "optimal" for growth but far
  too volatile in practice and brutal under estimation error; half-Kelly with
  clamps is the pragmatic choice.
- **External risk/OMS platform.** Rejected for this scope: heavyweight and
  opaque relative to a small, auditable, in-process risk layer that shares code
  with the backtester.
- **Manual circuit-breaking only (human pulls the plug).** Rejected: humans
  aren't watching at 3 a.m.; the floor must be automatic.
