# Setup Guide

This guide takes you from a clean machine to a running agent runtime: prerequisites,
configuration, registering your own tools, and deploying as a service. It is
framework-level — adapt the paths and service names to your own implementation.

Every example below uses **environment-variable placeholders**. Do not paste real
keys anywhere; supply them through your environment or secret store.

---

## 1. Prerequisites

**Runtime host** — any always-on machine you control: a small server, a VM, a
container host, or an edge device. CPU-only is fine for the orchestration layer.

**A language model endpoint** — one of:
- a **local model server** exposing a chat-completion API with tool use, or
- a **hosted model API**.

You can start with one and add an escalation model later (see
[ADR-0003](docs/adr/0003-provider-agnostic-llm-mesh.md)).

**Speech I/O (only if you want the voice channel)** — a microphone and speaker on
the host, plus a speech-to-text engine and a text-to-speech engine. The text,
HTTP, and MCP channels need none of this.

**A place to keep secrets** — at minimum a shell that can set environment
variables; optionally a structured secret-store file outside the repo, or a managed
secrets backend (see [ADR-0002](docs/adr/0002-tiered-secret-storage.md)).

**Network** — if the assistant will reach other machines, put them on a private
network (a mesh VPN or a private subnet) and expose only what you must.

---

## 2. Install

```bash
# 1. Get the framework
git clone <YOUR_FORK_URL> agent-runtime
cd agent-runtime

# 2. Create an isolated environment and install dependencies
#    (use whatever toolchain your implementation targets)
make install         # or: python -m venv .venv && pip install -r requirements.txt

# 3. Copy the example configuration and env template
cp reference/config.sample.yaml config.yaml
cp .env.example .env
```

Open `config.yaml` and `.env` and fill in the placeholders described below.
**Never commit `.env` or `config.yaml` if you put real values in them** — both
should be in `.gitignore`. The committed artifacts are only the `*.sample` /
`*.example` files.

---

## 3. Configure (environment variables)

The runtime resolves every secret by **name**, environment-first. Set the bootstrap
variables in your shell, your process manager, or your container runtime — never in
source.

### 3.1 Core

| Variable | Required | Purpose |
|---|---|---|
| `AGENT_SECRET_STORE_PATH` | optional | Filesystem path to the out-of-repo secret store. Omit to run env-var-only. |
| `AGENT_LOG_LEVEL` | optional | `info` (default), `debug`, `warn`. |
| `AGENT_MAX_LOOP_ITERS` | optional | Hard ceiling on agent-loop iterations. |
| `AGENT_BUDGET_USD_PER_REQUEST` | optional | Per-request cost ceiling enforced by the budget gate. |
| `AGENT_BUDGET_USD_PER_DAY` | optional | Per-window cost ceiling. |

### 3.2 Model mesh

| Variable | Required | Purpose |
|---|---|---|
| `LLM_ROUTER_ENDPOINT` | yes | Base URL of the fast/cheap model used for everyday turns. |
| `LLM_ROUTER_MODEL` | yes | Model identifier for the router tier. |
| `LLM_ROUTER_API_KEY` | if hosted | API key **name** resolved at runtime; leave unset for a keyless local server. |
| `LLM_ESCALATION_ENDPOINT` | optional | Endpoint for the strong model used on escalation. |
| `LLM_ESCALATION_MODEL` | optional | Strong-model identifier. |
| `LLM_ESCALATION_API_KEY` | optional | Strong-model key name. |

### 3.3 Ingress channels

| Variable | Required | Purpose |
|---|---|---|
| `INGRESS_ENABLED` | yes | Comma list: any of `voice,text,http,mcp`. |
| `HTTP_BIND_ADDR` | if http | Address to bind the HTTP endpoint (bind to a private interface). |
| `HTTP_AUTH_TOKEN` | if http | Bearer-token **name** required on inbound HTTP requests. |
| `VOICE_WAKE_PHRASE` | if voice | Wake phrase for the speech channel. |
| `STT_MODEL_PATH` / `TTS_MODEL_PATH` | if voice | Paths to the speech-to-text / text-to-speech model assets. |

### 3.4 Integrations (examples — add what you use)

| Variable | Purpose |
|---|---|
| `SVC_<NAME>_BASE_URL` | Base URL for an HTTP service integration. |
| `SVC_<NAME>_TOKEN` | Token name for that service, resolved from the store. |
| `OAUTH_<PROVIDER>_CLIENT_ID` / `_CLIENT_SECRET` | Client credentials for a user-delegated OAuth integration. |
| `REMOTE_HOST_KEY_PATH` | Path to the private key used by the remote-host adapter. |
| `MEMORY_STORE_PATH` | Path/URI for the namespaced memory store. |

> **Rule of thumb:** if a value is sensitive, the variable holds a *name* the
> resolver looks up in the secret store — not the secret itself. See
> [ADR-0002](docs/adr/0002-tiered-secret-storage.md).

---

## 4. Set up the secret store (optional but recommended)

For anything beyond a trivial deployment, create a structured secret store **outside
the repository** and point `AGENT_SECRET_STORE_PATH` at it. Group credentials by
integration:

```jsonc
// stored at $AGENT_SECRET_STORE_PATH, OUTSIDE the repo, NEVER committed
{
  "llm": {
    "router_api_key":     "<PLACEHOLDER>",
    "escalation_api_key": "<PLACEHOLDER>"
  },
  "http_ingress": {
    "auth_token": "<PLACEHOLDER>"
  },
  "services": {
    "example_service": { "token": "<PLACEHOLDER>" }
  },
  "oauth": {
    "example_provider": {
      "client_id":     "<PLACEHOLDER>",
      "client_secret": "<PLACEHOLDER>",
      "refresh_token": "<FILLED_BY_PROVISIONING_STEP>"
    }
  }
}
```

Back this file up securely and restrict its permissions. To **rotate** any
credential: change it here (or in your managed backend), then restart the service —
nothing in source references the old value.

For teams, point the same resolver at a managed secrets backend instead of a file;
no application code changes.

---

## 5. Authentication setup

### Service-account integrations (machine-owned resources)
Create a dedicated machine identity with **least-privilege** permissions in the
target system, store its credential in the secret store, and reference it by name.

### User-delegated integrations (a person's own accounts)
Run the one-time, interactive consent flow to obtain a refresh token:

```bash
# one-time, interactive — opens a browser for consent, then writes the
# refresh token into the secret store (never printed to logs)
make provision-oauth PROVIDER=<provider> CLIENT_SECRET_FILE=<path>
```

Request the **minimum scopes** the feature needs. Remember that the runtime
narrows scope further at call time, so even a coarse grant is constrained in code
(see [ADR-0004](docs/adr/0004-service-account-vs-user-auth.md)). The user can revoke
the grant at any time from their account's permissions page.

---

## 6. Register your tools

A **tool** is one action the assistant can take. Each is a typed descriptor in the
registry; nothing executes that is not described here. Define a tool by specifying:

- a **name** and **description** (what the model sees),
- a **parameter schema** (JSON-schema shape),
- a **handler** (the adapter method that runs),
- and its **policy fields**: `category`, `destructive`, `needs_consent`, `tier`,
  `cost_estimate`, `timeout`, and capability flags.

Conceptual example (pseudocode — adapt to your implementation language):

```python
registry.add(Tool(
    name="archive_report",
    description="Move last week's generated report to long-term storage.",
    parameters=obj_schema(
        {"report_id": {"type": "string", "description": "ID of the report"}},
        required=["report_id"],
    ),
    handler=storage_adapter.archive,
    category="storage",
    destructive=True,        # changes durable state
    needs_consent=True,      # require explicit confirmation before firing
    tier="default",          # not reachable from low-trust ingress
    cost_estimate=0.0,       # local op
    timeout=30,
))
```

Guidelines:

- **Default to least authority.** Start a tool at `low_trust` and raise its tier
  only if a higher-trust caller needs it. The gate enforces tiers in code, so
  low-trust ingress cannot reach higher-tier tools.
- **Flag anything that mutates** as `destructive=True`; add `needs_consent=True`
  for the higher-risk ones. Destructive tools that accept a `dry_run` parameter run
  in simulation by default.
- **Mark high-stakes tools** so the cheap router model cannot make the final
  decision on them — the gate will escalate to a stronger model.
- **Mark state-dependent tools** so the gate requires freshly-collected world-state
  before they fire.
- **Give every tool a realistic `cost_estimate`** so the budget gate can do its
  job.

Register tools at startup. They appear automatically in the model's tool catalog
and in the reporting layer's grouping.

---

## 7. Run locally

```bash
# start the runtime with the channels you enabled in INGRESS_ENABLED
make run
```

Smoke-test each channel you turned on:

- **HTTP:** send an authenticated request to `HTTP_BIND_ADDR` and confirm a JSON
  reply.
- **Text:** send a message and confirm a response.
- **Voice:** say the wake phrase and confirm a spoken reply.
- **MCP:** connect an MCP-aware client and list the exposed tools.

Then verify the guardrails explicitly — ask for a **destructive** action and
confirm the runtime asks for consent (or refuses from a low-trust channel) rather
than just doing it. The gate working is the most important thing to confirm before
you deploy.

---

## 8. Deploy as a service

Run the runtime as a long-lived, auto-restarting service under whatever supervisor
your host uses.

**Generic checklist (any platform):**

- [ ] Run under a process supervisor with automatic restart on crash.
- [ ] Inject all configuration via environment variables / mounted secrets — never
      bake secrets into the image or the unit file.
- [ ] Bind the HTTP ingress to a **private** interface; expose it publicly only
      behind an authenticating proxy if you must.
- [ ] Put any remote hosts the agent reaches on a private network.
- [ ] Set `AGENT_BUDGET_USD_PER_DAY` so a runaway loop cannot run up unbounded
      cost.
- [ ] Ship logs to your log aggregator; confirm the reporting layer scrubs
      sensitive content from output.
- [ ] Enable the secret-scanning CI step so no credential can ever be committed.

**Container sketch (illustrative):**

```dockerfile
# Illustrative only. No secrets in the image.
FROM <base-runtime-image>
WORKDIR /app
COPY . .
RUN make install
# All real values come from the environment / mounted secrets at run time:
#   -e LLM_ROUTER_ENDPOINT=... -e AGENT_SECRET_STORE_PATH=/run/secrets/store ...
CMD ["make", "run"]
```

**Service-unit sketch (illustrative):**

```ini
# Illustrative only. Reference an env file kept OUTSIDE version control.
[Service]
EnvironmentFile=/etc/agent-runtime/agent.env   # contains var NAMES + non-secret config
ExecStart=/usr/bin/make run
Restart=always
```

---

## 9. Operate

- **Add a capability:** register a new tool (§6). No core changes.
- **Add a channel:** enable it in `INGRESS_ENABLED` and supply its variables. No
  core changes.
- **Add or swap a model provider:** write one adapter satisfying the model contract
  and point the relevant `LLM_*` variables at it
  ([ADR-0003](docs/adr/0003-provider-agnostic-llm-mesh.md)).
- **Rotate a secret:** update the store/backend and restart the service (§4).
- **Tune cost:** adjust the budget variables and per-tool cost estimates; review
  the cost ledger.

---

## 10. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Startup fails resolving a secret | Variable unset or store path wrong | Confirm the env var and `AGENT_SECRET_STORE_PATH`; the resolver fails closed by design. |
| A tool call is refused as "tier too low" | A low-trust channel tried a higher-tier tool | Expected guardrail — raise the channel's tier only if truly appropriate, or lower the tool's. |
| Destructive action only simulates | Dry-run-by-default on a destructive tool | Pass an explicit opt-out for that call once you have verified the simulation. |
| Hard requests give weak answers | No escalation model configured | Set the `LLM_ESCALATION_*` variables and flag the relevant tools for escalation. |
| Costs higher than expected | Too much traffic on the strong tier | Verify the fast-path and router tier are engaged; tighten budgets and escalation flags. |
| Stale-state refusal | A state-dependent tool ran without fresh world-state | Ensure the relevant state collector is configured and within its freshness window. |
