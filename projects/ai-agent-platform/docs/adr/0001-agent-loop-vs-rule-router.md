# ADR-0001 — Agent loop over a rule-based intent router

- **Status:** Accepted
- **Decision drivers:** flexibility, maintenance cost, latency, predictable spend

## Context

The runtime has to turn an open-ended request ("check the incidents and draft a
note", "what's the status of the data job, and if it's done, archive last week's
output") into a sequence of concrete tool calls. There are two established ways to
do this.

**Option A — a rule-based intent router.** Classify each request into one of a
fixed set of intents, then run a hand-written handler. Deterministic and cheap,
but every new capability or phrasing is a code change, multi-step requests need
bespoke orchestration per combination, and the intent tree grows without bound as
the surface area expands.

**Option B — an agent loop.** Give a language model the request plus a catalog of
available tools and let it choose, in a bounded *plan → act → verify* loop, which
tools to call and in what order. Flexible and composable, but a naive
implementation is slow, expensive, and — most dangerously — trusts the model's
judgment about whether an action is safe.

The hard requirement is broad capability without a combinatorial explosion of
hand-written routes, *and* without surrendering safety to the model.

## Decision

**Adopt the agent loop (Option B) as the primary path, with two disciplines that
neutralize its weaknesses:**

1. **A deterministic fast-path in front of the loop.** Trivial, unambiguous
   requests (greetings, simple status reads, canned lookups) are matched and
   answered *before* any model call. This keeps the common case as cheap and fast
   as the rule-router would have been.

2. **A policy gate behind the loop.** The model never executes anything directly.
   Every proposed tool call passes through a central dispatch gate that enforces
   schema validation, trust tier, consent for destructive actions, a cost budget,
   and state-freshness — in code. The model's role is reduced to *proposing*;
   *authority* is enforced declaratively per tool.

The loop itself is bounded: a hard ceiling on iterations and an overall cost
budget guarantee termination.

## Consequences

**Positive**
- New capabilities are added by registering a tool, not by editing a router.
  Multi-step requests compose for free.
- The fast-path keeps median latency and cost near the rule-router baseline; full
  reasoning is spent only where it is needed.
- Safety is independent of model quality: even a poor completion cannot bypass the
  gate, because authority is not something the model can talk its way into.
- The same loop serves every ingress channel, eliminating per-channel
  orchestration logic.

**Negative / trade-offs**
- Harder requests cost more and are slower than a pure rule lookup — accepted,
  because they are the minority and the fast-path covers the rest.
- Behavior on novel requests is less perfectly predictable than a fixed tree —
  mitigated by the bounded loop, the dry-run-by-default rule, and consent gating,
  which cap the downside of any single bad decision.
- Requires a disciplined tool registry and a well-maintained gate; the complexity
  moves from many handlers into one well-tested policy layer (a trade worth
  making, because that layer is auditable in one place).

**Net:** the loop's flexibility, plus a fast-path for cost and a gate for safety,
delivers broad capability without the maintenance burden of a rule tree or the
risk of an ungoverned agent.
