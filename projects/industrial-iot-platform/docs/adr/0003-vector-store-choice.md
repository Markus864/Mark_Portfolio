# ADR-003 — Co-locate the Vector Store in the Primary Relational Database

- **Status:** Accepted
- **Context area:** Retrieval-augmented AI, datastore topology, operations

## Context

The troubleshooting assistant uses retrieval-augmented generation: it embeds the asset's
documents (drawings, manuals, past work orders), stores those embeddings, and at query time
finds the most relevant chunks by vector similarity before composing an answer with citations.

That requires a vector search capability. The options considered:

1. **A dedicated standalone vector database** alongside the relational database.
2. **A managed third-party vector/search service** over the network.
3. **A vector extension inside the primary relational database**, so documents, their metadata,
   and their embeddings live together.

This is a single-site, plant-scale corpus — thousands of documents, not hundreds of millions of
vectors. The platform is also designed to run **on-premises in the production environment**,
where every additional moving part is one more thing to install, secure, back up, patch, and
reason about. And the people expected to operate it on-site are a maintenance/engineering team,
not a dedicated database reliability group.

## Decision

Use a **vector extension within the primary relational database**. Embeddings are stored in the
same database as the documents they belong to and the rest of the application schema. Similarity
search and ordinary relational queries run in one place, over one connection, inside one
transaction boundary.

## Consequences

**Positive**
- **One datastore to operate.** A single system to back up, restore, secure, patch, and
  monitor — a decisive advantage for an on-prem deployment maintained by a small team.
- **Transactional consistency.** A document and its embedding are written and updated together;
  there is no second store to drift out of sync or to reconcile after a partial failure.
- **No extra network hop or external dependency** for retrieval — relevant for the on-prem
  production environment, where minimising outbound dependencies and external attack surface is a
  goal in its own right.
- **Simpler security and audit story.** Access control, encryption, and audit live with the same
  database that already holds the sensitive maintenance data.
- **Lower cost and lower operational complexity** than standing up and running a separate vector
  engine or paying for a managed search service.

**Negative / accepted trade-offs**
- A dedicated vector engine offers a **higher performance and scale ceiling** (specialised
  indexes, sharding, very-high-dimensional workloads at massive volume). At plant-corpus scale,
  that ceiling is far above what this system needs, so the trade is comfortable.
- Embedding generation and re-indexing still consume database resources; mitigated by running
  indexing **asynchronously on workers**, off the request path.

**Revisit if**
- The corpus or query volume grows by orders of magnitude (e.g. many sites federated into one
  store), or
- Retrieval latency under load becomes a measured bottleneck.

In either case the retrieval layer is deliberately kept behind a thin internal interface, so a
future move to a dedicated vector engine would be an implementation swap rather than a redesign.
