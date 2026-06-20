# Reference project structure

A recommended folder layout for an implementation of the runtime. This is a
**generic blueprint** — there is no application code here, only the shape that keeps
the five layers cleanly separated and the project shippable. Adapt names to your
language and toolchain.

The organizing rule: **core depends on adapter *contracts*, never on concrete
integrations.** Everything provider- or service-specific lives behind an interface,
so it can be swapped without touching the brain.

```
agent-runtime/
├── README.md
├── SETUP.md
├── LICENSE
├── .gitignore                  # ignores .env, config.yaml, secret store, build junk
├── .env.example                # every required VAR NAME, placeholder values only
├── reference/
│   ├── project-structure.md    # this document
│   └── config.sample.yaml      # annotated config, ENV-VAR placeholders only
│
├── docs/
│   ├── architecture.md
│   └── adr/                    # architecture decision records
│
├── src/
│   ├── core/
│   │   ├── registry/           # typed Tool descriptor + the tool catalog + schema helpers
│   │   ├── policy/             # the dispatch gate (validate/tier/consent/budget/freshness)
│   │   │   ├── dispatch_gate   #   the single chokepoint every tool call passes through
│   │   │   └── budget          #   per-call / per-window cost ceilings + cost ledger
│   │   ├── planner/            # the plan->tool->verify agent loop + fast-path + system prompt
│   │   ├── state/              # state store + collectors (TTL-cached world-state slices)
│   │   └── reporting/          # channel-aware output shaping + sensitive-content scrub
│   │
│   ├── adapters/               # ONE file per integration, each satisfying a contract
│   │   ├── contracts           #   the Protocol/interface definitions the core depends on
│   │   ├── model_router        #   fast/cheap model adapter
│   │   ├── model_escalation    #   strong model adapter (API or CLI form)
│   │   ├── http_service        #   generic HTTP/REST service adapter
│   │   ├── remote_host         #   authenticated remote command execution
│   │   ├── filesystem          #   local file read/write on the execution host
│   │   └── memory              #   namespaced persistent memory
│   │
│   ├── ingress/                # thin shells, ONE per channel — normalize in, pick formatter
│   │   ├── voice/              #   wake -> capture -> STT ... TTS out
│   │   ├── text/
│   │   ├── http/
│   │   └── mcp/
│   │
│   ├── integrations/           # concrete tool handlers grouped by domain
│   │   └── <your_domains>/     #   each registers Tool descriptors into the registry
│   │
│   └── secrets/
│       └── resolver            # name-based, environment-first secret resolution
│
├── tools/
│   └── provision_oauth         # one-time interactive user-delegation flow (writes to store)
│
├── tests/
│   ├── smoke                   # hermetic: no model, no network, no I/O
│   ├── gate                    # policy-gate behavior: tiers, consent, budget, freshness
│   └── adapter_conformance     # each adapter satisfies its contract
│
└── deploy/
    ├── Dockerfile              # no secrets baked in; all config via env at run time
    ├── service.unit.example    # references an env file kept OUTSIDE version control
    └── ci/                     # lint + tests + SECRET SCANNING (fails on any committed key)
```

## Why this shape

- **`core/` is provider-free.** It imports only the contracts in
  `adapters/contracts`. You can read the entire brain without learning any vendor's
  SDK, and you can replace any integration without editing it.
- **`adapters/` is one file per integration.** Each absorbs one vendor's auth,
  request shape, and quirks, and exposes a uniform contract. This is the seam that
  makes the model layer provider-agnostic and keeps every integration's blast
  radius to a single file.
- **`ingress/` shells are thin.** They contain no tools and no reasoning — only
  input normalization and output-formatter selection — so adding a channel is
  cheap and behavior stays identical across all of them.
- **`integrations/` is where capabilities live.** Domain handlers register typed
  tool descriptors into the registry; the registry, not the model, is the source of
  truth for what the assistant can do.
- **`secrets/resolver` centralizes the name-based, env-first lookup**, so no other
  file ever contains a secret value.
- **`tests/` proves the guardrails.** The `gate` suite is as important as the smoke
  suite: it is where you assert that low-trust callers can't reach high-trust tools,
  destructive actions require consent, and budgets are enforced.
- **`deploy/ci/` includes secret scanning.** The build fails if a credential is
  ever committed — the automated backstop behind
  [ADR-0002](../docs/adr/0002-tiered-secret-storage.md).

## What deliberately is **not** here

- No secret values, no `.env`, no populated config — only `*.example` /
  `*.sample` artifacts are committed.
- No machine names, no network addresses, no account identifiers anywhere in the
  tree.
- No UI: build a front-end on top of the HTTP or MCP surface if you want one.
