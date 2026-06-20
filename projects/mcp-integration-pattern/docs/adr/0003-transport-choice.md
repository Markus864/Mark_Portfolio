# ADR-0003 — stdio as the default transport, Streamable HTTP for hosted use

- **Status:** Accepted
- **Date:** generalized from a production decision
- **Decision owner:** Mark Splawn

## Context

MCP supports more than one transport. The two that mattered here:

- **stdio** — the MCP client launches the server as a child process and talks to it over standard input/output. No network, lifecycle managed by the client.
- **Streamable HTTP** — the server runs as a long-lived HTTP service and clients connect to an endpoint, typically authenticating per request.

The deployment context spanned two quite different topologies:

- **Local, single-operator:** one person, one machine, an agent that can spawn a subprocess. The priority is the smallest possible attack surface and trivial setup.
- **Shared / hosted:** several clients (possibly remote, possibly an agent registry) needing to reach one running instance behind a tunnel or load balancer.

Forces:

- A listening socket is attack surface; if I do not need one, I should not open one.
- Hosted access requires solving auth, CORS, and request statelessness correctly — non-trivial, and pure cost in the local case.
- I did not want **two implementations** of the tools — the transport should be swappable beneath identical tool logic.

## Decision

**Default to stdio; offer Streamable HTTP as an alternate transport over the same tool code.**

- The local/default entry point runs the server on **stdio**, so the client spawns and owns its lifecycle and there is no network surface.
- A second entry point mounts the **identical tool registry** under a **stateless Streamable-HTTP** transport: a fresh server/transport per request, credential taken from the per-request `Authorization` header, with a `/health` endpoint and correct CORS preflight handling.
- Tool definitions are shared; only the transport wrapper differs.

## Consequences

**Positive**

- **Minimal surface by default.** stdio opens no port, so the common local case has essentially no network attack surface and the client manages start/stop.
- **Scales when needed without a rewrite.** The HTTP form fans out to many clients and supports central hosting behind a tunnel/load balancer — and reuses the same tools, so there is one source of truth for behaviour.
- **Right tool for each topology.** Single-operator gets simplicity; shared deployments get reach.

**Negative / costs**

- **stdio is one-client-per-process.** Each client spawns its own instance; that is correct for local use but does not share state across clients (acceptable — job state lives in the platform, not the server).
- **HTTP reintroduces real concerns.** Running it means handling authentication (per-request token, with a proper `401`/`WWW-Authenticate` challenge), CORS, statelessness, and the operational burden of an always-on service.
- **Two entry points to maintain.** Mitigated by keeping all logic in the shared tool registry and letting the entry points be thin.

**Mitigations**

- Keep the HTTP transport **stateless** (new server+transport per request) to avoid session-management bugs and make horizontal scaling trivial.
- Expose a `/health` endpoint for load-balancer probes and return a proper auth challenge when the token is missing, so clients and gateways behave predictably.
