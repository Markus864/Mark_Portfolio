# ADR-0003 — Provider-agnostic LLM mesh

- **Status:** Accepted
- **Decision drivers:** vendor independence, cost, reliability, quality-on-demand

## Context

The reasoning engine is a large language model, but the model market moves fast:
prices change, new models ship, providers have outages, and capability-per-dollar
varies wildly by task. A runtime welded to one provider's SDK and tool-call format
inherits all of that volatility and has no answer to two structural realities:

- **Most requests are easy and a few are hard.** Paying top-model prices for every
  greeting and status check is wasteful; routing every hard problem to a cheap
  model produces bad results. A single fixed model is wrong for one of these cases
  no matter which you pick.
- **A single provider is a single point of failure.** If the one model endpoint is
  down or rate-limited, an agent hardwired to it is simply offline.

## Decision

**Depend on a uniform model-adapter contract, not on any concrete provider, and
operate the models as a mesh with a cheap-router / strong-escalation split.**

1. **One interface, many implementations.** The core defines a single adapter
   contract — *"given messages and a tool catalog, return the assistant's next
   message, including any tool calls."* Every model, local or hosted, is wired in
   behind that contract. Provider-specific concerns (auth scheme, request shape,
   the fact that some models return tool calls as structured fields and others
   embed them as text) are absorbed *inside* the adapter and normalized before the
   planner sees them. The core contains zero provider branches.

2. **A routing policy across tiers.**
   - A **fast, cheap model** (local or a low-cost hosted tier) handles everyday
     turns and the first pass of most requests.
   - **Stronger models** are reserved for genuinely hard work — deep reasoning,
     code, ambiguity — and are reached by *escalation*.
   - Escalation is not left to chance: tools can be flagged such that the cheap
     router model is **forbidden from making the final decision** on them, and the
     dispatch gate enforces that boundary, redirecting to a stronger model.

3. **Escalation targets are pluggable too.** A stronger model can be wired in as a
   direct API adapter or as a command-line tool invoked non-interactively —
   whichever the provider offers — behind the same escalation contract.

## Consequences

**Positive**
- **No lock-in.** Swapping or adding a provider is one new adapter file plus a
  registry entry. The orchestration logic never changes.
- **Cost follows difficulty.** The common case runs on the cheap tier; spend
  concentrates on the minority of hard requests, governed further by the budget
  gate.
- **Graceful degradation.** If one provider is unavailable, the mesh can fall back
  to another tier; an outage degrades quality rather than taking the system down.
- **Quality on demand.** Hard problems transparently get a stronger model without
  the operator having to route them by hand.

**Negative / trade-offs**
- The adapter layer is extra indirection and must absorb each provider's quirks —
  accepted, because that complexity is contained in small, individually testable
  files and is exactly what buys the independence.
- A routing policy ("when does a turn escalate?") needs ownership and tuning; the
  capability-flag approach makes it explicit and per-tool rather than a hidden
  heuristic.
- Behavior can differ subtly between providers; a small conformance test suite
  against the adapter contract keeps implementations honest.

**Net:** treating the model as a replaceable component behind a uniform contract —
and running cheap-by-default with strong-on-escalation — gives the best of cost,
reliability, and quality while keeping the operator free to follow the market.
