# Documentation

The architecture narrative for this portfolio lives here. It is organized into the
cross-cutting platform decisions I apply across every project, plus a dated log of the
individual decisions behind them.

| Section | What's inside |
|---|---|
| [Cross-Cutting Architecture](architecture/README.md) | Multi-cloud strategy, secrets and least privilege, CI/CD, and observability/reliability/cost — the reusable stance applied across all projects. |
| [Architecture Decision Records](adr/README.md) | Dated `context -> decision -> consequences` records for the cross-cutting, load-bearing choices; each project also carries its own project-specific ADRs in its folder. |

For an individual system end-to-end (Problem -> Architecture -> Trade-offs -> Tech ->
Outcomes), see the case studies under [`../projects/`](../projects/), indexed from the
[repository README](../README.md).
