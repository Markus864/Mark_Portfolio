# ADR-0003: Vendor-agnostic provider abstraction layer

- **Status:** Accepted
- **Date:** 2026
- **Deciders:** Mark Splawn (architecture owner)
- **Tags:** extensibility, vendor-risk, testability, cost-control

## Context

The platform depends on third-party AI providers for prompt shaping, image generation,
motion/video generation, and captioning. This dependency carries unusual **business and
engineering risk**:

- **Vendor volatility.** AI provider pricing, output quality, latency, and even availability
  shift on a monthly cadence. The best provider for a capability today may be the wrong choice
  in a quarter.
- **API heterogeneity.** Providers differ in auth scheme, request shape, and especially
  response shape — some return results inline, others use an asynchronous
  `submit → poll status_url → fetch response_url` handshake, and many nest the payload under
  varying keys (`images`, `data.images`, `video`, `data.video`, …).
- **Cost and testing.** Every live call costs money. Local development, CI, and demos must run
  the *whole* pipeline without spending, and the platform needs a kill switch to disable live
  spend instantly.

If business logic imported a specific vendor's SDK directly, switching or A/B-testing a
provider would be a cross-cutting rewrite, and the pipeline would be untestable without
real spend.

## Decision

Introduce a **single vendor-agnostic adapter interface** that the pipeline core depends on,
with **swappable implementations** selected by configuration:

```ts
interface MediaProvider {
  shapePrompt(input: PromptInput): Promise<StructuredPrompt>;
  generateImage(prompt: string, refs: string[], ratio: AspectRatio): Promise<Artifact>;
  generateVideo(prompt: string, startFrame: string, ratio: AspectRatio): Promise<Artifact>;
  generateCaption(prompt: string): Promise<string>;
}
```

Rules the abstraction enforces:

1. **Core never imports a vendor.** The durable workflow calls only the interface. The
   concrete provider is loaded from environment config
   (`MEDIA_PROVIDER`, with `PROVIDER_API_KEY` injected as an env var) — never referenced by
   business logic.
2. **One normalized result shape.** Every adapter returns the same
   `Artifact { hostedUrl, contentType, fileName, providerTrace }`, hiding each vendor's
   bespoke envelope. The core never branches on vendor-specific JSON.
3. **Adapters own long-running-job mechanics.** The submit/poll/backoff loop (with
   capability-appropriate timing — images poll faster and shorter than video) lives entirely
   inside the adapter. The workflow just `await`s one method.
4. **A deterministic fallback adapter** is always available. When live providers are disabled
   (`ENABLE_LIVE_PROVIDERS=false`), the fallback returns structured, deterministic output so
   the full pipeline runs in local dev, CI, and as an emergency cost-control kill switch — with
   zero changes to core logic.
5. **`providerTrace` is mandatory** on every artifact for observability: which provider, which
   endpoint, which storage tier handled this job.

## Consequences

**Positive**

- **Provider swaps are intended to be configuration, not code.** Replacing or adding a vendor
  should touch only an adapter; the queue, workflow, billing, storage, and API stay untouched.
- **A/B testing and fallback routing become tractable.** Because selection is config-driven and
  the result shape is uniform, routing a percentage of traffic to a challenger provider — or
  failing over from a degraded one — is a localized change.
- **The whole pipeline is testable and demoable for free.** The deterministic fallback runs CI
  and local dev end-to-end without spending or network access.
- **Instant cost kill switch.** Flipping `ENABLE_LIVE_PROVIDERS` off stops all paid calls
  immediately without a deploy of business logic.
- **Resilience to vendor churn.** When a provider changes its API or degrades, the blast radius
  is intended to be contained to one adapter file.

**Negative / accepted trade-offs**

- **A normalization layer to maintain.** Mapping each vendor's envelope to the common
  `Artifact` shape is ongoing work, and a provider's breaking change still requires an adapter
  update (but only there).
- **Lowest-common-denominator contract.** The interface exposes capabilities common across
  providers; a vendor's unique premium feature is either omitted or modeled as an optional
  extension, to keep the core vendor-neutral.
- **Indirection cost.** One more layer between the workflow and the network call — a deliberate
  and cheap price for the isolation it buys.

## Alternatives considered

- **Call vendor SDKs directly in the pipeline:** rejected — turns every provider swap into a
  cross-cutting rewrite and makes the pipeline impossible to test without live spend.
- **A heavyweight plugin framework / registry:** rejected as over-engineering at this scale; a
  plain interface plus env-based selection delivers the needed flexibility with far less
  ceremony.
- **A third-party "AI gateway" aggregator:** considered as a possible *implementation behind*
  the interface, but rejected as the *only* path because it would add another vendor dependency
  and another pricing/availability risk in front of the providers — the very risk this ADR
  exists to contain. The abstraction stays ours; an aggregator can later sit behind it as just
  another adapter.
