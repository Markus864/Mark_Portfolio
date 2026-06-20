# ADR-0001 — Expose the platform to agents via MCP, not per-agent clients

- **Status:** Accepted
- **Date:** generalized from a production decision
- **Decision owner:** Mark Splawn

## Context

An existing REST platform needed to be callable by several different AI agents (a CLI coding agent, a desktop assistant, an autonomous operations agent), with more likely to come. Each agent runtime has its own way of defining "tools" or "functions" it can call.

Three options were on the table:

1. **Per-agent REST client.** Write a bespoke HTTP integration inside each agent runtime.
2. **A shared SDK/library.** Publish one client library and import it into every agent.
3. **A Model Context Protocol (MCP) server.** Expose the platform once as MCP tools and let any MCP-capable agent connect.

The forces in play:

- The API bills real money on certain calls, so agents need a *machine-readable* notion of which operations are dangerous.
- Agents do better with **typed, introspectable** tools than with raw JSON-body construction.
- I did not want to maintain the same auth/retry/error logic in N places.
- The platform sits behind a CDN/WAF that needs a specific request fingerprint — a detail I wanted to solve exactly once.

## Decision

**Adopt MCP as the single integration contract.** Build one thin MCP server that adapts the platform's existing REST surface into typed tools, and let every agent reach the platform through that server.

A shared SDK (option 2) was rejected because it still requires per-runtime wiring and gives the agent no standard way to *discover* tools or reason about side effects — it just relocates the per-agent glue. Per-agent clients (option 1) were rejected outright as the status quo whose costs prompted this work.

## Consequences

**Positive**

- **One integration, every agent.** Any current or future MCP-speaking client gets the full tool catalog with no new code.
- **Typed, self-describing tools.** Parameters are validated at the boundary; descriptions carry intent and consequences, so agents fill arguments correctly and know what is destructive.
- **Single place for cross-cutting concerns.** Auth, timeouts, error shaping, and the CDN/WAF User-Agent workaround live in one server, fixed once.
- **Clean separation.** Business logic, authorization, and billing stay in the platform; the MCP server is a thin, honest adapter with no duplicated logic.

**Negative / costs**

- **Extra hop and dependency.** An additional process and protocol now sit in the path, and the design leans on the maturity of the MCP ecosystem and its client support.
- **Protocol learning curve.** Contributors must understand MCP transports and tool semantics, not just HTTP.

**Mitigations**

- Keep the server deliberately thin so the "extra hop" adds negligible logic and latency.
- Document the tool catalog and the read-vs-destructive convention so the contract is obvious to both humans and agents.
