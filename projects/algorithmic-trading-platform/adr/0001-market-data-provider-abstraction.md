# ADR-0001: Market-Data Provider Abstraction with Circuit Breaker

- **Status:** Accepted
- **Date:** Reference architecture
- **Deciders:** Mark Splawn
- **Related:** ADR-0003 (idempotent execution), `architecture.md` §4

---

## Context

The engine needs OHLCV history and current prices to make decisions. In the
first iteration this came directly from a single commercial market-data vendor,
called inline from the scanner and signal code.

That coupling caused concrete, recurring pain:

- **Quota exhaustion.** A continuously scanning bot burns a paid monthly request
  quota in a matter of hours. When the quota ran out, calls started failing and
  the engine had no graceful behavior — it either errored out or, worse, kept
  trying and acting on whatever it last had.
- **Single point of failure.** Any vendor outage, auth hiccup, or breaking
  payload change took the entire system down.
- **Vendor lock-in.** Swapping providers meant touching every call site, because
  each vendor has its own payload shape, candle ordering (newest- vs
  oldest-first), interval labels, and endpoint set.
- **Cost opacity.** There was no central place to throttle, cache, or budget API
  usage, so spend was unpredictable.

I needed price/candle access to be **reliable, swappable, and cost-bounded**
without leaking any of that complexity into strategy or engine code.

## Decision

Introduce a single **`MarketDataProvider` interface** with a normalized data
model, and route all access through a **facade that adds caching, a circuit
breaker, and a fallback chain**.

1. **Stable interface.** Every provider implements the same minimal contract:

   ```python
   class MarketDataProvider(ABC):
       @abstractmethod
       def get_ohlcv(self, symbol: str, interval: str, count: int) -> list[Candle] | None: ...
       @abstractmethod
       def get_price(self, symbol: str) -> float | None: ...
   ```

2. **Normalized model.** Each adapter maps its vendor payload to a canonical
   candle (`{time, open, high, low, close, volume}`, oldest-first) and a float
   price. Vendor quirks (ordering, labels, missing live-price endpoints) are
   absorbed *inside* the adapter — the engine sees one shape, always.

3. **TTL cache.** OHLCV for a `(symbol, interval)` is cached for roughly one
   candle interval. Repeated reads within a scan cycle (and price reads derived
   from the latest candle) collapse to a single upstream call.

4. **Circuit breaker.** On a quota/rate-limit/repeated-error signal, the breaker
   trips for a cool-off window. While tripped, calls **short-circuit and return
   `None`**, and the engine declines to trade rather than acting on stale data.
   The tripped/healthy state is **persisted** and rehydrated on process restart
   so status surfaces can't get stuck reporting the wrong state. A single alert
   fires on trip and on resume.

5. **Fallback chain.** When the primary provider is unavailable (auth/quota), the
   facade can transparently serve the *same normalized shape* from a secondary
   provider, degrading gracefully instead of halting.

## Consequences

### Positive

- **Vendor independence.** Migrating the primary data source to a different
  provider became a single adapter change with **zero edits** to strategy or
  engine code.
- **Graceful degradation.** A vendor outage or quota wall degrades to "skip this
  cycle" or "use the fallback," rather than "crash" or "trade on stale data."
- **Bounded, visible cost.** Caching plus a capped scan universe plus throttling
  bound API spend; the facade is the one place to tune it.
- **Restart-safe reliability state.** Persisting breaker state removed a class
  of "stuck status" bugs after restarts.
- **Testability.** The interface is trivial to mock, so strategy and engine tests
  run with deterministic, offline data.

### Negative / costs

- **Per-vendor normalization work.** Each new adapter must faithfully map to the
  canonical model, including subtle ordering and precision details. A mapping bug
  is a correctness bug.
- **Contract discipline.** All adapters must stay in lock-step on the interface;
  changing it is a cross-adapter change.
- **Cache staleness window.** Caching trades a small amount of freshness for cost
  and rate-limit safety. Mitigated by tying TTL to the candle interval and
  bypassing cache for the freshest decision inputs where it matters.

### Neutral

- The breaker's "fail safe → return `None`" choice pushes a clear contract onto
  callers: missing data means *do not act*. This is intentional and consistent
  with the platform's risk posture.

## Alternatives considered

- **Keep a single vendor, add retries.** Rejected: retries don't solve quota
  exhaustion, lock-in, or single-point-of-failure; they can worsen rate-limit
  problems.
- **Adopt a heavyweight third-party data framework.** Rejected for this scope:
  more dependencies and operational surface than a small, dependency-light
  facade, with less control over caching/breaker behavior.
- **Cache only, no breaker.** Rejected: caching reduces call volume but does
  nothing for the "vendor is down / over quota" case, which is exactly when
  trading on stale data is most dangerous.
