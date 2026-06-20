# Security Notes

The threat model and secret-handling guarantees for this MCP integration pattern.

## Secrets

**No secrets live in this repository.** The backend API key and base URL are resolved from a
secret store **outside** the repo, located via an environment variable (`APP_CONFIG_PATH`-style)
with a documented default under the user's profile. The repo ships only a placeholder example of
the config shape. The key is never:

- committed to version control,
- baked into a build artifact or image,
- passed as a command-line argument (which would be visible in process listings),
- logged, or echoed back inside tool results.

Protect the store with filesystem permissions, and rotate the key at the platform if exposure is
suspected. Because the key is read per call, rotation takes effect on the next tool invocation.

## Authentication

Outbound only in the local form: the adapter sends `Authorization: Bearer <api_key>` plus a
custom `User-Agent` (a CDN/WAF compatibility requirement on some backends). In the hosted HTTP
form, the credential is taken from the **per-request** `Authorization` header; a missing token
returns `401` with a `WWW-Authenticate` challenge so gateways and clients negotiate correctly.

## Authorization

Authorization stays **server-side on the backend platform** (per plan / per account). The MCP
layer adds one safety convention on top: every state-changing or cost-incurring tool is marked
**DESTRUCTIVE** in both its machine-readable description and the server's top-level instructions,
so well-behaved agents confirm intent before mutating state or spending money.

## Transport surface

- **stdio (default):** no listening socket, no inbound network surface at all. The process runs
  with the invoking user's privileges and only makes outbound HTTPS calls to the configured base URL.
- **Streamable HTTP (optional):** introduces a listening socket. It must therefore be run behind
  TLS (reverse proxy / tunnel), require a per-request token, handle CORS explicitly, and be kept
  stateless. Treat exposing it as a deliberate decision, not a default.

## External calls & SSRF

The server makes exactly one class of upstream call: HTTPS to the configured backend. If a tool
forwards a **user-supplied URL** to the backend (e.g. an "ingest this asset from a URL" tool),
the resulting fetch happens on the platform side — so SSRF exposure, if any, lives in the
platform, not in this client. Still, validate/allowlist such URLs where practical.

## Data sensitivity

- Tool arguments (prompts, parameters) and asset metadata transit to the backend; treat them as
  non-confidential at rest in transit logs you do not control.
- Any tool that lists **local filesystem** contents reveals file names, sizes, and paths to the
  connected MCP client. Point such tools only at directories you are comfortable exposing.

## Reliability as a security property

Every outbound call has an explicit timeout, and upstream failures are returned as **structured
data** rather than raised as exceptions. This prevents a hostile or flaky backend from hanging a
tool call indefinitely and gives the agent a deterministic failure it can handle.

## Known limitations

1. In the local form, the secret store is protected by **OS file permissions only** — no
   encryption at rest. Use an encrypted volume or a managed KV/secret manager if your threat model
   requires it.
2. Treat this document as the design's security baseline, not a hardening checklist: any
   implementation should add automated tests (including error-path tests), pin dependency versions
   for supply-chain reproducibility, and add structured audit logging before any production,
   multi-user deployment.
3. The destructive-tool classification is a **convention** that relies on the client honoring it;
   it reduces blast radius but is not an authorization control. Real authorization remains the
   backend's responsibility.

## Reporting

This is a public reference/portfolio project. Please open an issue rather than including any
sensitive detail in a public report.
