# ADR-0002 — Externalize credentials and config out of the repo and out of argv

- **Status:** Accepted
- **Date:** generalized from a production decision
- **Decision owner:** Mark Splawn

## Context

The MCP server authenticates to the platform with a Bearer API key and needs a base URL (which differs between local, staging, and production deployments). I had to decide where those values live.

Constraints and forces:

- The repository is intended to be **publishable**. A single committed secret would burn the key and the repo's history.
- Process command-line arguments are visible to any local user who can list processes, so passing the key as a CLI flag is unsafe.
- Keys must be **rotatable** without a code change or, ideally, even a restart.
- The hosted (HTTP) deployment has a different, per-request notion of "whose key is this" than the local (stdio) one.

Options considered:

1. **Hard-code** the key/URL in source. *(Rejected — leaks on first commit.)*
2. **CLI flags** (`--api-key ...`). *(Rejected — visible in process listings, ends up in shell history.)*
3. **Environment variables only.** Workable, but env blocks still get copied into client configs and can leak via crash dumps / child processes.
4. **An external secret store** (an OS-permission-protected file or KV), with an **env-var override for its location**, resolved per call.

## Decision

**Resolve configuration from an external secret store, addressed by an env-var, and read it per call.** Concretely:

- A `*_CONFIG_PATH`-style environment variable points at a secret file **outside** the repository (with a sensible default location under the user's profile).
- For the hosted transport, the per-request `Authorization: Bearer <key>` header is the credential source instead of a local file.
- The repo ships only a **placeholder example** of the config shape; real values never enter version control.
- Credentials are read at the moment of use, not cached for the process lifetime.

## Consequences

**Positive**

- **Zero secrets in version control.** The repository is verifiably safe to publish; a leaked repo leaks no key.
- **No leakage via process listings.** Nothing sensitive is on the command line.
- **Rotation without restart.** Because the key is read per call, rotating it in the store takes effect on the next tool invocation.
- **Transport-appropriate sourcing.** Local use reads a protected file; hosted use trusts the per-request header — each matches its threat model.

**Negative / costs**

- **A tiny per-call read.** Re-reading the store on each call adds negligible I/O, accepted in exchange for rotation-without-restart and simplicity.
- **Operational dependency.** Whoever deploys the server must provision the store and set its permissions correctly; a misconfigured path fails the call (by design) rather than silently falling back to an insecure default.
- **Security floor = filesystem permissions.** In the local form, the file is protected by OS permissions, not encryption at rest. Documented as a known limitation in `SECURITY.md`.

**Mitigations**

- Ship a clear placeholder config and a `SETUP.md` that uses **only env-var names**, never real values.
- Fail loudly with an actionable message when the store is missing or the key is empty, so misconfiguration is obvious immediately.
